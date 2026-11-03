# HumanPlusMoCapDesktop 使用手册

本手册适用于 HumanPlusMoCapDesktop.exe 安装包（v1.0.0 Preview）。这是 Windows 64 位人体动作捕捉发送端：接收上衣和裤子的 IMU 数据，完成校准与姿态解算，向 Unity 或第三方程序输出。安装包不包含 Unity 接收端、SteamVR 或 PICO 应用。

## 安装与启动

1. 从本 Release 下载 HumanPlusMoCapDesktop.exe 和本手册。双击安装包，阅读许可协议并选择安装目录；安装需要管理员权限，可选创建桌面快捷方式。
2. 从开始菜单或桌面快捷方式启动。主程序窗口名为 HumanPlusMoCap；首次运行默认中文、经典模式、蓝牙连接，BLE 地址留空，需要填写自己的衣物地址。
3. 左侧设置连接方式、地址和端口，点击“开始连接”。运行时不能切换连接方式或动捕模式；先点击“停止连接”再修改。右侧显示衣服、裤子、校准、Unity、UDP 输出及实时姿态状态，信息区域支持滚轮滚动；圆形“中/en”按钮切换语言。

配置保存在 %LOCALAPPDATA%\HumanPlusMoCap\runtime_config.json。安装包中的默认配置不覆盖已有用户配置。主程序文件仍叫 HumanPlusMoCap.exe；HumanPlusMoCapDesktop 是安装包和产品名称。

## 衣物连接与校准

本版本需要兼容新版 SmartWear 协议的上衣和裤子固件，不兼容旧版衣物数据格式。两件衣物都在传输后才能完成校准。校准期间保持 T-Pose。

- BLE：打开 Windows 蓝牙，填写上衣和裤子的 BLE 地址，确保设备开机且可连接。开始连接后，桌面端在两件衣物就绪时自动校准。
- Serial：插入兼容的 CH340 串口接收器，确认驱动正常且串口未被其他程序占用。选择串口模式后开始连接，无需 BLE 地址。桌面端自动查找接收器并在衣物就绪后校准；接收器使用电脑的 UTC 时间回应衣物校时请求，请先同步 Windows 系统时钟。
- PICO UDP：PICO 连接两件衣物，桌面端只接收 PICO 的局域网 UDP 数据。PICO 的目标 IP 填写桌面端显示的本机局域网 IP，目标端口填写 5005。桌面端选择 PICO UDP、本机监听地址 0.0.0.0、端口 5005，点击“开始连接”。随后在 PICO 上点击“开始校准”并保持 T-Pose；PICO 发送完整校准批次，桌面端应用后回复确认。仅发送实时衣物数据不会完成校准。重新校准也在 PICO 上发起；桌面端或 Unity 的重新校准请求会让桌面端等待下一批 PICO 校准数据。

跨设备使用时，允许相关 UDP/TCP 入站流量通过 Windows 防火墙。PICO 输入只接受本次运行中首个有效发送方 IP；PICO 的 IP 改变后需停止并重新开始连接。PICO 模式目前按最新收到的帧推理，不按 UTC 时间戳对齐两件衣物；断流约 2 秒后不会继续使用旧帧。

## Unity 与 VR Assist

Unity 输出使用 TCP，默认监听 0.0.0.0:8888。Unity 接收端连接桌面端的本机 IP 和端口 8888；同机可以使用 127.0.0.1。可另行安装 [HumanPlusMoCapUnity](https://github.com/XMU-HumanPlus/HumanPlus-Products/releases/tag/HumanPlusMoCapUnity/v0.1.0) 或使用兼容的 Unity 工程。Unity 未连接或中途断开，不影响已开启的外部 UDP 姿态输出。

VR Assist 需要在电脑上启动 SteamVR，使头显和左右两个手柄处于可追踪状态。开始动捕前在桌面端切换到 VR 辅助模式；检测失败时不会切换。先让 Unity 接收端连接，再在 Unity 的 MoCapToVR 场景点击 VR Assist 按钮。桌面端在 VR 辅助模式采集头显和手柄的位置并通过姿态 UDP 数据提供；头部/手臂的 VR 姿态修正由兼容的 Unity 接收端完成。切换模式前应先停止动捕。

## 外部姿态 UDP 输出

姿态输出是发往指定地址的 UDP JSON，初始关闭，默认目标为 127.0.0.1:8889。在界面开启并设置目标 IP/端口；接收程序监听对应端口即可，不需要先与桌面端建立连接。跨设备时，目标 IP 应为接收电脑的局域网 IP。

三个端口的用途不同：5005 为 PICO 输入 UDP，8888 为 Unity TCP，8889 为默认姿态输出 UDP。

每个数据报是一帧 UTF-8 JSON：

| 字段 | 含义 |
| --- | --- |
| schema_version | 当前为 1 |
| coordinate_system | right-handed |
| timestamp, frame_index | Unix 时间（秒）和递增帧号 |
| joint_names | 24 个关节的顺序 |
| local_euler_xyz_deg | 24 × 3 局部 XYZ 欧拉角，单位为度 |
| local_quaternion_wxyz | 24 × 4 局部四元数，分量顺序 wxyz |
| root_global_euler_xyz_deg, root_global_quaternion_wxyz | 根节点旋转，当前直接取根节点局部旋转 |
| root_global_position_xyz | 根节点全局位置 XYZ |
| vr_tracking | 可选的头显、左右手柄位置与有效性；未启用时 enabled 为 false |

关节顺序为 pelvis, l_hip, r_hip, spine1, l_knee, r_knee, spine2, l_ankle, r_ankle, spine3, l_toe, r_toe, neck, l_collar, r_collar, head, l_shoulder, r_shoulder, l_elbow, r_elbow, l_wrist, r_wrist, l_palm, r_palm。姿态采用右手坐标系；Unity 使用自己的坐标转换，第三方程序不应直接套用 Unity 的轴符号。vr_tracking 只提供位置与有效性，不提供控制器旋转。UDP 不保证送达、顺序或重传。

以下接收示例只需 Python 标准库，在同一台电脑上监听默认端口：

~~~python
import json
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", 8889))
print("Listening on UDP 8889")

while True:
    data, sender = sock.recvfrom(65535)
    packet = json.loads(data.decode("utf-8"))
    if packet.get("schema_version") != 1:
        continue
    names = packet["joint_names"]
    angles = packet["local_euler_xyz_deg"]
    head_xyz = angles[names.index("head")]
    root_xyz = packet["root_global_position_xyz"]
    print(sender, "head XYZ deg:", head_xyz, "root position:", root_xyz)
~~~

## 故障排查与预发布说明

- 衣物未连接：检查连接方式、固件版本、BLE 地址或 CH340 驱动，以及设备是否开机；PICO 模式还需检查目标 IP、UDP 端口和防火墙。
- PICO 已发送实时数据但尚未校准：在 PICO 上点击“开始校准”，等待批次发送和桌面端确认。
- Unity 未连接：检查桌面端 TCP 监听地址与端口、Unity 的目标 IP、接收端是否启动及防火墙。
- VR 辅助不可用：检查 SteamVR、头显及双手柄追踪，确认桌面端已切换 VR 辅助模式并在 Unity 场景中启用。
- 未收到姿态 UDP：确认已开启输出、目标 IP/端口正确、接收程序正在监听，并已完成衣物连接和校准。

此版本为 Preview。Windows 构建、15 项自动化测试及打包程序启动冒烟测试通过；新命名安装包在目标电脑的完整安装与真实衣物/PICO/Unity/SteamVR 联机仍需现场验证。
