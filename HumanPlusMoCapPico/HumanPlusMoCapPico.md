# HumanPlusMoCapPico 使用说明

本产品是运行在 PICO 4 Ultra Enterprise 上的智能衣物动作数据采集应用。衣服和裤子各是一台蓝牙设备，每台设备内部有六颗 IMU；只需分别连接衣服和裤子，无需逐颗连接 IMU。衣物接上电源后会自动开机、广播并开始发送数据。应用提供 BLE 时间同步、T-Pose 校准、本地录制和可选的 UDP 实时发送。

## 版本与安装

- 本说明对应 `HumanPlusMoCapPico-v0.1.0-debug.apk`。这是使用 Android 调试密钥签名的**预发布安装包**，最低支持 Android API 29；推荐设备为 PICO 4 Ultra Enterprise。
- 从同一个 [HumanPlusMoCapPico v0.1.0 Release](https://github.com/XMU-HumanPlus/HumanPlus-Products/releases/tag/HumanPlusMoCapPico/v0.1.0) 下载 APK 和本说明。不要把 APK 与其他版本的说明混用。
- 在头显打开 USB 调试，连接电脑后运行 `adb devices` 确认设备在线，再运行 `adb install -r HumanPlusMoCapPico-v0.1.0-debug.apk`。也可以在头显中打开 APK 安装。首次启动请授予“附近设备”权限，并保持蓝牙开启。
- 此包使用调试签名；若头显中已安装相同应用 ID、但签名不同的版本，覆盖安装会失败。正式部署需要稳定的正式签名包。

## 连接衣服和裤子

1. 给衣服和裤子接通电源。若它们正在连接其他接收端，先断开旧连接；一件衣物通常不能同时连接两个接收端。
2. 在左侧的“衣服”和“裤子”卡片分别点击“尝试连接”。连接期间可点击“终止连接”。卡片会显示连接状态、时间同步状态与帧率。
3. 首次使用或更换衣物时，在右侧对应设备一栏点击“重新绑定”，从下方扫描区域选择设备并确认。应用按**绑定关系**区分衣服和裤子，不能凭蓝牙地址末位识别类型。绑定地址保存在应用本地，下次启动可继续使用。
4. 两件衣物均连接且显示“时间同步：已启用”后，再校准或录制。意外断连会弹出提示；录制中意外断连会自动终止并保存文件。

## 校准与录制

- 点击“开始校准”后，弹窗倒计时 **3、2、1**。倒计时结束，出现“请保持T-Pose”时才开始采集，采集持续 **2 秒**。请在整个采集窗口保持姿势。
- 未录制时，成功校准会在 `Download/HumanPlus/` 生成 `cali_yyyy-MM-dd-HH-mm-ss-SSS.jsonl`。再次成功校准会删除上一次独立校准文件；校准失败则保留上一次成功的文件。
- 点击“开始录制”后，应用立即记录数据并自动启动一次上述校准流程。录制期间再次点击校准可产生多个校准区间；这些区间记录在同一个录制文件里，不另建校准文件。点击“停止录制”完成保存。
- 文件名输入框为空时使用 `smartwearmocap`。可开启附加开始时间选项，文件名为 `<名称>-yyyy-MM-dd-HH-mm-ss.jsonl`；关闭时为 `<名称>.jsonl`。文件名中的非字母、数字、下划线、连字符会替换为下划线。
- 录制与独立校准均保存在头显公共目录 `Download/HumanPlus/`，可通过头显文件管理器查看，也可用 `adb pull /sdcard/Download/HumanPlus/` 导出。

## 可选 UDP 实时发送

在右侧“02 / 校准”区域填写接收电脑的局域网 IP 和 UDP 端口（默认 `5005`），然后开启“UDP 实时发送”方形开关。衣服和裤子的有效帧**分别**作为 UTF-8 JSON 数据报发送至同一目标 IP/端口，不合并成一帧；是否录制不影响普通帧发送。接收端按 `role` 的 `shirt`/`pants` 区分衣物。

每个普通数据报的 `type` 为 `imu`，`formatVersion` 为 `1`；包含 `device`、`role`、`timestampUtcNs`、`timestampSource`、`frameUtcNs`、`receiveUtcNs`、`receiveElapsedNs`、`clockUncertaintyNs`、帧序号信息和六颗 IMU 的 `samples`，不包含文件中的 `rawHex`。纳秒时间戳以**十进制字符串**表示，便于接收端准确解析 64 位数。`timestampSource=garment` 表示使用衣物时间，`pico_receive` 表示回退到 PICO 接收时间。

开启 UDP 时，校准完成后应用还会向同一目标发送 `calibration_begin`、逐帧 `calibration_imu` 和 `calibration_end`；接收端需向发送源 IP/端口回复 `calibration_ack`。只有收到 `status=applied`，才表示电脑端已应用校准。开启 UDP 的校准要求每件衣物在采集窗口内至少有 60 个不同帧；关闭 UDP 时至少需要一帧。UDP 不保证普通数据报必达，完整记录请使用本地文件。

## 本地文件格式与时间戳

文件为 UTF-8 JSON Lines（`.jsonl`），每行一个 JSON 对象。`type` 有 `session`、`imu`、`event`、`calibration` 四种；文件格式版本 `formatVersion=2`。每件衣物的每帧 `imu` 行包括蓝牙地址 `device`、文件内逐设备递增的 `frameIndex`、`frameUtcNs`、`receiveElapsedNs`、`clockUncertaintyNs`、`protocolBytes`、`sequenceKind`、`sequence`、`rawHex` 和六颗 IMU 的 `samples`。每个样本有 `imu`（1–6）、`packetCnt`、`sampleUtcNs`、`acceleration`（XYZ，m/s²）与 `quaternionXyzw`（XYZW）。`frameUtcNs` 或 `sampleUtcNs` 在衣物未给出有效时间时可能为 `null`。

```json
{"type":"imu","device":"AA:BB:CC:DD:EE:FF","frameIndex":123,"receiveElapsedNs":"123456789000000","clockUncertaintyNs":"1000200","protocolBytes":137,"sequenceKind":"none","sequence":null,"frameUtcNs":"1800000000000000000","rawHex":"fffe...","samples":[{"imu":1,"packetCnt":42,"sampleUtcNs":null,"acceleration":[0.0,0.0,9.8],"quaternionXyzw":[0.0,0.0,0.0,1.0]}]}
```

示例仅展示一颗 IMU 且 `rawHex` 已截短；实际有效帧包含六颗及完整原始十六进制数据。`calibration` 行记录 T-Pose 采集的开始/结束时间及按设备列出的 `firstFrameIndex`、`lastFrameIndex`、帧数。倒计时中的帧属于普通录制，只有倒计时后 2 秒的帧进入校准区间。

应用通过 BLE 时间服务使衣物与头显对时。`frameUtcNs` 是衣物帧时间（Unix UTC 纳秒，十进制字符串）；`receiveElapsedNs` 是头显开机后的单调时钟纳秒，仅能在同一次开机内直接比较。头显系统时钟若有误，绝对 UTC 也会偏移；需要与其他设备对齐时，请先校对头显及其他设备的时钟来源，并检查衣物时间是否有效。

## 当前验证状态

此预发布包已通过 `:app:assembleDebug` 与 `:app:testDebugUnitTest`。旧版应用曾在 PICO 真机完成双衣连接、时间同步、录制、校准及 UDP 联调；本次 SDK 模块重构后的安装包尚未在 PICO 上重新进行真机回归，因为构建时 ADB 未检测到设备。正式使用前建议在目标头显验证连接、文件写入与接收端校准流程。
