# HumanPlusMoCapPicoSDK 使用手册

本手册与 `HumanPlusMoCapPicoSDK.aar` 一起交付即可使用，不需要项目源码或其他文档。SDK 运行在 Android 设备上，提供 BLE 连接、两件衣物管理、时间同步、逐帧 IMU 数据、T-Pose 校准、JSONL 文件录制，以及可选 UDP 实时传输；SDK 不提供界面。主要入口是 `com.humanplus.wearsdk.SmartWearClient`。推荐在 PICO 4 Ultra Enterprise 上使用，最低 Android API 为 29。

## 衣物上电与连接

衣服和裤子**接上电源后会自动开机、开始采集数据并发出 BLE 广播**，不需要在 SDK 中发送开机命令。宿主应用授予蓝牙权限后调用 `scan()` 可发现它们。选择并绑定地址后，调用 `connect(role)`；SDK 发现对应设备、建立 GATT 连接、订阅数据通知并启动 BLE 时间服务。**连接和订阅成功后，衣物会自动持续发送 IMU 数据帧，SDK 通过 `onFrame` 交付给宿主**，无需再调用“开始采集”；`startRecording()` 只是把这些帧另外保存到文件。SDK 不主动扫描或自动连接，宿主应用需要发起这些操作。

**衣服是一台 BLE 设备，裤子是另一台 BLE 设备；每台设备内部各有六颗 IMU。客户只需连接衣服和裤子这两台设备，无需逐颗连接 IMU。**一件衣物通常不能同时连接两个接收端。`SHIRT`、`PANTS` 角色由绑定关系确定，不能根据蓝牙地址末位判断。每次成功连接的地址会保存在宿主应用的本地偏好设置中，重建 client 后可从 `boundAddress(role)` 读取；首次使用需先扫描并绑定。当前最多连接两台衣物设备，连接请求应依次发起。

## 安装 AAR 与权限

把交付的 `HumanPlusMoCapPicoSDK.aar` 放到宿主项目的 `app/libs/`，在宿主模块的 `build.gradle.kts` 中加入：

```kotlin
dependencies {
    implementation(files("libs/HumanPlusMoCapPicoSDK.aar"))
}
```

AAR 含 SDK 代码和 Android Manifest；宿主需使用 Android/Kotlin 构建环境并提供 Kotlin 标准库。宿主应用的最终 Manifest 应合并 `BLUETOOTH_SCAN`、`BLUETOOTH_CONNECT`、`INTERNET` 等声明。Android 12 及以上必须在运行时请求“附近设备”权限，旧版 Android 需请求位置权限；SDK 用 `requiredPermissions()` 返回当前系统所需权限，用 `hasPermissions()` 检查，但**弹出授权对话框由宿主 Activity 完成**。建议首次扫描前请求，并在用户拒绝时停止调用连接 API。

```kotlin
val missing = client.requiredPermissions().filter {
    checkSelfPermission(it) != android.content.pm.PackageManager.PERMISSION_GRANTED
}
if (missing.isNotEmpty()) requestPermissions(missing.toTypedArray(), 100)
// 收到授权结果且 client.hasPermissions() == true 后，再调用 client.scan()/connect()。
```

SDK 内部回调在 Android 主线程交付。`onFrame` 可能持续高频触发，图像计算、磁盘写入或网络处理应交给宿主自己的工作线程。一个 client 对应一组衣服和裤子；宿主退出页面时调用 `close()`。若需后台持续采集，由宿主在前台服务中持有 client。

## 完整公开 API

以下表格列出客户应使用的**全部稳定接口**。`ClockBridge`、`WearBleManager`、`SessionRecorder`、`UdpFrameSender` 和 `Protocol` 是 AAR 内部实现类，不属于客户接入契约；不要直接实例化或依赖它们的行为。`Context` 建议传 `applicationContext`；`SmartWearClient` 持有 listener，使用完后必须关闭。

| `SmartWearClient` API | 参数 / 返回 | 用途与约束 |
| --- | --- | --- |
| `SmartWearClient(context, listener)` | `Context`, `SmartWearListener` | 创建实例；不会自动扫描或连接 |
| `requiredPermissions()` | `Array<String>` | 当前 Android 版本所需运行时蓝牙权限 |
| `hasPermissions()` | `Boolean` | 检查权限是否已授予 |
| `scan()` | `Unit` | 开始扫描 SmartWear，并启动 BLE 时间服务；需先授权 |
| `stopScan()` | `Unit` | 停止当前扫描 |
| `bind(role, address)` | `WearRole`, 蓝牙地址 → `Unit` | 绑定并持久化角色地址；录制或校准中不可更换，同一地址不可绑定两个角色 |
| `boundAddress(role)` | `WearRole` → `String?` | 读取当前/上次成功绑定的地址 |
| `connect(role)` | `WearRole` → `Unit` | 扫描并寻找绑定设备；20 秒未找到会通过 `onConnection` 报告失败 |
| `connectScanned(role)` | `WearRole` → `Unit` | 连接本轮扫描中已发现的绑定地址 |
| `cancelConnect(role)` | `WearRole` → `Unit` | 中止正在寻找或连接的设备 |
| `disconnect(role)` | `WearRole` → `Unit` | 主动断开；结果由 `onConnection` 报告 |
| `isConnected(role)` | `WearRole` → `Boolean` | 查询连接状态 |
| `isSynchronized(role)` | `WearRole` → `Boolean` | 查询衣物校时状态 |
| `refreshTimeSync()` | `Unit` | 立即刷新 PICO 单调时钟到系统 UTC 的映射；SDK 默认每 30 秒自动刷新 |
| `calibrate()` | `Unit` | 启动 3 秒倒计时及 2 秒 T-Pose 采集；两件衣物须连接并校时，结果由 `onCalibration` 报告 |
| `cancelCalibration()` | `Unit` | 取消当前倒计时或采集 |
| `isCalibrating()` | `Boolean` | 查询是否正在倒计时或采集 |
| `startRecording(fileName, appendTime, autoCalibrate)` | `String`, `Boolean`, `Boolean` → `OutputFile` | 开始 JSONL 录制；默认文件名 `smartwearmocap`、不加时间、自动校准；需两件衣物连接并校时 |
| `stopRecording(unexpected)` | `Boolean` → `OutputFile?` | 停止录制；未在录制时返回 `null`；文件完成信号为 `onRecordingStopped`，客户通常无需传 `unexpected` |
| `isRecording()` | `Boolean` | 查询录制状态 |
| `currentFile()` | `OutputFile?` | 当前或最近一次创建的文件；即使录制结束也可能非空 |
| `startUdp(host, port, result)` | 地址、1–65535 端口、`(Boolean, String) -> Unit` | 可选：向指定电脑地址开启实时发送；结果回调在主线程 |
| `stopUdp()` | `Unit` | 关闭实时发送及未完成的校准传输 |
| `close()` | `Unit` | 释放 BLE、录制和 UDP 资源；关闭后的实例不可复用 |

`scan()`、`connect()`、`connectScanned()` 在权限缺失时抛 `IllegalArgumentException`；未绑定、重复校准、录制条件不满足等会抛 `IllegalStateException`。宿主应在调用前检查状态，必要时捕获异常并显示给用户。连接、扫描和数据接收本身是异步的，以 listener 事件为准。

| `SmartWearListener` 回调 | 参数 | 含义 |
| --- | --- | --- |
| `onStatus(message)` | `String` | BLE、UDP、校准等可显示的状态或错误文字 |
| `onDeviceFound(address, name, connectable)` | 地址、名称、`Boolean` | 扫描候选；只应让用户选择 `connectable=true` 的设备 |
| `onFrame(frame)` | `WearFrame` | 一件衣物的一帧有效数据，与录制或 UDP 开关无关 |
| `onConnection(role, address, connected, expected, details)` | 角色可空、地址、连接布尔值、是否预期断开、详情 | 连接成功/失败和断连；`expected=false` 可用于意外断连提醒 |
| `onTimeSync(role, address, synchronized)` | 角色可空、地址、布尔值 | 衣物校时状态变化 |
| `onCalibration(phase, outcome)` | 阶段、结果可空 | 倒计时、采集、完成或失败；结果只在正常采集结束时提供 |
| `onRecordingStopped(file, unexpected)` | `OutputFile?`、`Boolean` | 文件已完成；意外断连导致结束时 `unexpected=true` |

| 数据类型 | 字段 / 枚举值 | 说明 |
| --- | --- | --- |
| `WearRole` | `SHIRT`, `PANTS` | 衣服、裤子 |
| `CalibrationPhase` | `COUNTDOWN_3`, `COUNTDOWN_2`, `COUNTDOWN_1`, `CAPTURING`, `FINISHED`, `FAILED` | 校准流程阶段 |
| `CalibrationOutcome` | `successful`, `counts: Map<WearRole, Int>`, `output: OutputFile?` | 采集是否成功、各角色帧数、独立校准文件或当前录制文件 |
| `OutputFile` | `name: String`, `uri: android.net.Uri` | 用 `ContentResolver.openInputStream(uri)` 读取；不要把 `Uri` 当作普通绝对路径 |
| `WearFrame` | `role`, `deviceAddress`, `timestampUtcNs`, `timestampSource`, `receiveUtcNs`, `receiveElapsedNs`, `clockUncertaintyNs`, `frame` | 实时帧及时间信息，详见下文 |
| `ImuFrame` | `sequence`, `sequenceKind`, `protocolBytes`, `frameUtcNs`, `samples` | 一帧协议数据；`sequence` / `frameUtcNs` 可空 |
| `ImuSample` | `index`, `packetCount`, `sampleUtcNs`, `acceleration`, `quaternion` | 单颗 IMU 数据；`sampleUtcNs` 可空 |

## Kotlin 示例

```kotlin
import com.humanplus.wearsdk.*

val client = SmartWearClient(applicationContext, object : SmartWearListener {
    override fun onDeviceFound(address: String, name: String, connectable: Boolean) {
        // 在宿主界面展示设备；用户确认“衣服/裤子”后调用 bind()。
    }

    override fun onFrame(frame: WearFrame) {
        // 接收一件衣物的一帧。与相机等数据对齐时使用 timestampUtcNs。
        val role = frame.role
        val utcNs = frame.timestampUtcNs
        val sixImus = frame.frame.samples
    }

    override fun onCalibration(phase: CalibrationPhase, outcome: CalibrationOutcome?) {
        // 显示倒计时、T-Pose 采集状态及结果。
    }

    override fun onConnection(role: WearRole?, address: String, connected: Boolean,
                              expected: Boolean, details: String) {
        // 衣服连接成功后，再调用 client.connect(WearRole.PANTS)。
    }

    override fun onRecordingStopped(file: OutputFile?, unexpected: Boolean) {
        // 此时文件已写完，可用 contentResolver.openInputStream(file!!.uri) 读取。
    }
})

// Activity 请求权限并收到授权结果后：
check(client.hasPermissions())
client.scan()
// 以下地址仅为示例；正式使用应让用户从 onDeviceFound 的列表选择。
client.bind(WearRole.SHIRT, "F5:C9:68:F5:91:72")
client.bind(WearRole.PANTS, "E7:B6:92:B6:0C:61")
// 建议先等衣服的 onConnection(connected=true)，再连裤子。
client.connect(WearRole.SHIRT)
```

在衣服的 `onConnection(connected=true)` 中调用 `client.connect(WearRole.PANTS)`。两件衣物都已连接并校时后，宿主界面的录制/停止按钮可这样调用：

```kotlin
if (client.isConnected(WearRole.SHIRT) && client.isConnected(WearRole.PANTS) &&
    client.isSynchronized(WearRole.SHIRT) && client.isSynchronized(WearRole.PANTS)) {
    client.startRecording("experiment_01", appendTime = true)
    // 默认自动校准；采集结束时会收到 onCalibration(FINISHED, ...)。
}
// 用户点击停止按钮时：
client.stopRecording()
// 等待 onRecordingStopped 交付后，再读取文件或关闭 client。
```

若要单独校准，先调用 `calibrate()`，等待 `onCalibration(FINISHED, outcome)`，再调用 `startRecording()`；录制期间可再次 `calibrate()`。连接中的 `cancelConnect(role)` 可中止尝试。需要换绑时，建议先 `disconnect(role)`、等待断开回调，再 `bind(role, 新地址)` 并 `connect(role)`。录制期间衣物意外断连会自动停止并保存文件。

## 时间与数据

`WearFrame` 为**单件衣物的一帧**，不会将衣服、裤子合并。`role`、`deviceAddress` 标识来源。`timestampUtcNs` 优先使用衣物的 UTC 帧时间；无有效帧时间时回退到 PICO 的接收 UTC，`timestampSource` 分别为 `garment`、`pico_receive`。`receiveUtcNs`、`receiveElapsedNs` 和 `clockUncertaintyNs` 同时保留，便于宿主评估时间质量。所有 `Long` 时间字段均为纳秒；跨设备对齐优先比较 `timestampUtcNs`，并检查时间来源。PICO 系统时钟若不准确，绝对 UTC 也会偏移；正式采集前应检查头显时间。

SDK 会每 30 秒刷新 PICO 单调时钟与系统 UTC 的映射；宿主如需立刻重采样，可调用 `refreshTimeSync()`。衣物 BLE 校时服务由 SDK 在扫描时启动。

`frame.samples` 中 `index=1..6`，加速度 `acceleration` 为 XYZ（m/s²），`quaternion` 为 XYZW；不删除裤子的 IMU 4，也不做宿主姿态模型的重排。`packetCount` 是单颗 IMU 的包计数，`sampleUtcNs` 可为 `null`；`frame.sequence` 对部分 137 字节固件帧也可为 `null`。监听回调在主线程交付，耗时处理应转到宿主自己的工作线程。SDK 不保证不同衣物的帧交替到达，也不保证它们有相同帧率。

## 校准、文件与 UDP

`calibrate()` 发出 `COUNTDOWN_3/2/1`、`CAPTURING`、`FINISHED` 或 `FAILED`。倒计时不算校准帧，之后采集 2 秒。UDP 关闭时要求每件衣物至少 1 帧；UDP 开启时要求每件至少 60 个不同帧。未录制时保存独立 `cali_yyyy-MM-dd-HH-mm-ss-SSS.jsonl`，只保留最近一次成功文件；录制中将每次成功校准的帧区间写入当前文件。失败时不会覆盖上一份成功的独立校准文件。

默认录制写到设备公共 `Download/HumanPlus/`，文件名是处理过的 `fileName` 加可选 `-yyyy-MM-dd-HH-mm-ss` 和 `.jsonl`；空名使用 `smartwearmocap`。录制文件与独立校准文件都是 UTF-8 **JSON Lines**，一行一个 JSON 对象。`startRecording()` 返回时文件已创建但仍在写入；`stopRecording()` 返回时也可能仍在写入队列中，**以 `onRecordingStopped` 为文件完成信号**。文件 `Uri` 通过 `ContentResolver` 读取，勿假设可用普通文件路径访问。宿主可在录制期间调用 `calibrate()` 多次。

### JSONL 文件格式

| `type` | 字段与意义 |
| --- | --- |
| `session` | 每文件首行；`formatVersion=2`、`purpose=recording` 或 `calibration`、`startedUtcMs`、`clock` |
| `imu` | 一件衣物的一帧；字段见下例。`frameIndex` 从该文件内每个设备的 1 开始递增 |
| `event` | `device`、`name`、`details`、`elapsedNs`、`utcMs`，记录连接、校时、异常和校准事件 |
| `calibration` | 一次成功校准；`pose=T-Pose`、采集开始/结束的 UTC 毫秒及单调纳秒、各设备的 `ranges` |

```json
{"type":"imu","device":"F5:C9:68:F5:91:72","frameIndex":123,"receiveElapsedNs":"123456789000000","clockUncertaintyNs":"1000200","protocolBytes":137,"sequenceKind":"none","sequence":null,"frameUtcNs":"1800000000000000000","rawHex":"fffe...","samples":[{"imu":1,"packetCnt":42,"sampleUtcNs":null,"acceleration":[0.0,0.0,9.8],"quaternionXyzw":[0.0,0.0,0.0,1.0]}]}
```

示例只展示一颗 IMU，实际有效帧有六颗。`rawHex` 是完整原始 BLE 帧；`protocolBytes` 为 136 或 137。`sequenceKind` 为 `updateSequence`、`frameSequence` 或 `none`，`sequence` 可能为 `null`。文件 `imu` 行没有 `role` 和统一的 `timestampUtcNs`，使用 `device` 与已绑定地址关联，优先用 `frameUtcNs`；为空时可用采集时保存的单调时钟记录与宿主的时钟映射处理。

```json
{"type":"calibration","pose":"T-Pose","startedUtcMs":1800000000000,"endedUtcMs":1800000002000,"startedElapsedNs":"123456789000000","endedElapsedNs":"123458789000000","ranges":[{"device":"F5:C9:68:F5:91:72","count":130,"firstFrameIndex":101,"lastFrameIndex":230,"firstSequence":null,"lastSequence":null}]}
```

实际 `ranges` 包含衣服和裤子。一次录制可有多条 `calibration` 行，表示开始录制时的自动校准和后续手动校准。

UDP 也是 SDK 功能，但宿主可完全不调用。`startUdp(host, port, callback)` 的 `host` 应是接收电脑的局域网 IP，默认示例端口 `5005`；`127.0.0.1` 指 PICO 本机。每收到一件衣物的一帧有效数据，就向相同目标发**一条 UTF-8 JSON UDP 数据报**，衣服和裤子不合并，按 `role=shirt` 或 `pants` 区分。发送与录制无关；`stopUdp()` 关闭。电脑端需监听对应 UDP 端口并允许防火墙入站。普通数据报不要求确认，队列满时可能丢帧，UDP 也不保证送达、顺序或唯一性；文件录制不依赖 UDP。

### 普通 UDP 帧格式

```json
{"type":"imu","formatVersion":1,"device":"F5:C9:68:F5:91:72","role":"shirt","timestampUtcNs":"1800000000000000000","timestampSource":"garment","frameUtcNs":"1800000000000000000","receiveUtcNs":"1800000000001000000","receiveElapsedNs":"123456789000000","clockUncertaintyNs":"1000200","protocolBytes":137,"sequenceKind":"none","sequence":null,"samples":[{"imu":1,"packetCnt":42,"sampleUtcNs":null,"acceleration":[0.0,0.0,9.8],"quaternionXyzw":[0.0,0.0,0.0,1.0]}]}
```

实际 `samples` 有六颗；与 JSONL 文件相比，UDP 帧增加 `role`、统一 `timestampUtcNs`、`timestampSource`、`receiveUtcNs`，不含 `rawHex`。所有纳秒时间戳用**十进制字符串**编码，避免 JavaScript 解析 64 位整数时丢精度。`frameUtcNs` 和 `sampleUtcNs` 可为 `null`。电脑端应按角色分别缓存，再用 `timestampUtcNs` 与其他数据对齐，不要按包到达顺序强行配对衣服与裤子。

### UDP 校准消息与确认

若 UDP 已开启，手动校准、自动校准或录制期间的再次校准完成后，SDK 额外发送如下批次。每个 JSON 对象单独占一个 UDP 数据报，与普通帧共用目标 IP/端口和 PICO 源 socket；普通 `type=imu` 仍可同时到达。每次校准使用新的 UUID `calibrationId`。

1. `calibration_begin`：采集结束后发送，含 `formatVersion=1`、`calibrationId`、`pose="T-Pose"`、实际采集窗口 `startedElapsedNs`/`endedElapsedNs`、`devices={"shirt":地址,"pants":地址}`、`frameCounts={"shirt":数量,"pants":数量}`。
2. `calibration_imu`：每个采集帧一包，沿用普通 UDP 帧字段和六颗 `samples`，另加 `calibrationId`、`calibrationFrameIndex`；索引按各角色采集顺序从 0 连续编号。原 `receiveElapsedNs` 落在 begin 的采集窗口内，不使用后续发包时间。
3. `calibration_end`：所有校准帧后发送，含 `formatVersion=1`、`calibrationId` 和 `frameCounts`。
4. 电脑端应向该批次的**源 IP 和源 UDP 端口**回复 `calibration_ack`：`status="applied"` 表示完成；`status="missing"` 携带 `missing={"shirt":[缺失索引],"pants":[缺失索引]}`，SDK 补发对应帧和 end；`missingBegin=true` 时补发 begin；`status="rejected"` 可带 `reason`。1 秒内无回复时 SDK 重发 end，最多尝试 5 次。没有 `applied` 不代表电脑端完成校准。

电脑端应按 `(calibrationId, role, calibrationFrameIndex)` 去重和收齐 `frameCounts`，只用 `calibration_imu` 组成校准批次；普通 `imu` 不应混入。SDK 不修改裤子 IMU 4 或 IMU 物理顺序。失败或缺帧时电脑端应保留上一份已成功的校准。SDK 的 `onCalibration(FINISHED)` 代表**本机采集和保存成功**；电脑端是否已应用请以 `onStatus("电脑端已应用 T-Pose 校准")` 或电脑端自己的状态为准。关闭 UDP 或开始新的校准会取消旧的未确认批次。

SDK 目前随调用者进程运行，`close()` 会结束连接和发送。需要后台持续运行时，由宿主在前台服务中持有 `SmartWearClient` 并管理其生命周期。
