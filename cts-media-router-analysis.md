# CtsMediaHostTestCases MediaRouter2 失败分析

## 概述

本文档分析了 AOSP W 版本上 `CtsMediaHostTestCases` 模块中 3 个 MediaRouter2 相关测试用例的失败原因。

### 失败用例列表

| # | 测试用例 | 模块 |
|---|---------|------|
| 1 | `MediaRouter2HostSideTest#screenOffScan_onProxyRouter_allowedWithMediaContentControl` | arm64-v8a CtsMediaHostTestCases |
| 2 | `MediaRouter2HostSideTest#screenOffScan_onLocalRouter_allowedWithMediaContentControl` | arm64-v8a CtsMediaHostTestCases |
| 3 | `MediaRouter2HostSideTest#requestScan_withOnScreenScan_withScreenOff_doesNotScan` | arm64-v8a CtsMediaHostTestCases |

设备序列号：`6UNBB26123000351`

---

## 一、失败现象

### 共同的错误堆栈

三个用例的失败堆栈 **100% 一致**：

```
com.google.common.util.concurrent.UncheckedExecutionException: 
  android.platform.test.flag.util.FlagReadException: Flag ALL_FLAGS read error
    at com.google.common.cache.LocalCache$Segment.get(LocalCache.java:2085)
    ...
    at android.platform.test.flag.junit.host.HostFlagsValueProvider.setUp(HostFlagsValueProvider.java:88)
    at android.platform.test.flag.junit.CheckFlagsRule$1.evaluate(CheckFlagsRule.java:46)
    ...
Caused by: android.platform.test.flag.util.FlagReadException: Flag ALL_FLAGS read error
    at android.platform.test.flag.junit.host.DeviceFlags.lambda$getAconfigParsedFlags$1(DeviceFlags.java:157)
    at android.aconfig.HostDeviceProtos.parsedFlagsProtoPaths(HostDeviceProtos.java:81)
    ...
Caused by: com.android.tradefed.device.DeviceUnresponsiveException
  [DEVICE_UNRESPONSIVE|520751|LOST_SYSTEM_UNDER_TEST]: 
  Attempted adb shell multiple times on device 6UNBB26123000351 
  without communication success. Aborting.
```

### 关键错误信息

```
DeviceUnresponsiveException[DEVICE_UNRESPONSIVE|520751|LOST_SYSTEM_UNDER_TEST]
```

- `520751` = 超时时间（约 8.7 分钟）
- `LOST_SYSTEM_UNDER_TEST` = system_server 无响应

---

## 二、测试功能说明

### 背景知识

#### MediaRouter2 是什么？

MediaRouter2 是 Android 的**媒体路由管理框架**，用于管理音频/视频输出的路由设备：
- 蓝牙耳机
- WiFi 音箱（Chromecast、AirPlay）
- HDMI 输出
- 车载音响

#### 路由扫描（Route Scanning）

应用需要**扫描**（发现）可用的路由设备。扫描有两种模式：
- **亮屏扫描**：屏幕亮着时扫描（正常场景）
- **息屏扫描**：屏幕关闭时仍然扫描（特殊场景，如手表、车载）

#### Local Router vs Proxy Router

```
Local Router:
  MediaRouter2.getInstance(context)
  - 管理自己的路由
  - 只能控制自己的媒体播放

Proxy Router:
  MediaRouter2.getInstance(context, "other.app.package")
  - 管理其他应用的路由
  - 可以控制其他应用的媒体播放
  - 需要特殊权限：MEDIA_CONTENT_CONTROL 或 MEDIA_ROUTING_CONTROL
```

### 三个测试用例的验证目标

#### 测试 1：`screenOffScan_onLocalRouter_allowedWithMediaContentControl`

**验证目标：** 拥有 `MEDIA_CONTENT_CONTROL` 权限的应用，能否在**息屏状态下**通过 **Local Router** 进行路由扫描。

**测试流程：**
1. 获取 `DEVICE_POWER` + `MEDIA_CONTENT_CONTROL` 权限
2. 让设备息屏
3. 注册路由回调，监听 `FEATURE_SAMPLE` 类型的路由
4. 通过 Local Router 请求息屏扫描（`setScreenOffScan(true)`）
5. **期望：** 能在息屏状态下发现路由设备

**所需 Flags：**
- `FLAG_ENABLE_SCREEN_OFF_SCANNING`
- `FLAG_ENABLE_FULL_SCAN_WITH_MEDIA_CONTENT_CONTROL`

#### 测试 2：`screenOffScan_onProxyRouter_allowedWithMediaContentControl`

**验证目标：** 拥有 `MEDIA_CONTENT_CONTROL` 权限的应用，能否在**息屏状态下**通过 **Proxy Router** 控制其他应用的路由扫描。

**测试流程：**
1. 获取权限、息屏、注册回调（同上）
2. 创建 Proxy Router（`MediaRouter2.getInstance(context, packageName)`）
3. 通过 Proxy Router 请求息屏扫描
4. **期望：** Proxy Router 的息屏扫描也能发现路由

**所需 Flags：**
- `FLAG_ENABLE_SCREEN_OFF_SCANNING`
- `FLAG_ENABLE_FULL_SCAN_WITH_MEDIA_CONTENT_CONTROL`

#### 测试 3：`requestScan_withOnScreenScan_withScreenOff_doesNotScan`

**验证目标：** **普通扫描请求**（非息屏扫描）在**息屏状态下不应该工作**。

**测试流程：**
1. 获取权限、息屏、注册回调（同上）
2. 请求普通扫描（没有 `setScreenOffScan(true)`）
3. **期望：** 息屏状态下，普通扫描**不会**发现路由（这是正确的省电行为）

**所需 Flags：**
- `FLAG_ENABLE_SCREEN_OFF_SCANNING`

### 三个测试的关系

```
┌──────────────────────────────────────────────────────────────┐
│              MediaRouter2 息屏扫描功能测试                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  测试 1: Local Router + 息屏扫描                              │
│  └─ 验证：有 MEDIA_CONTENT_CONTROL 权限时，Local Router       │
│         可以在息屏时扫描                                       │
│  └─ 期望：✅ 能扫描到路由                                      │
│                                                               │
│  测试 2: Proxy Router + 息屏扫描                              │
│  └─ 验证：有 MEDIA_CONTENT_CONTROL 权限时，Proxy Router       │
│         可以在息屏时控制其他应用扫描                             │
│  └─ 期望：✅ 能扫描到路由                                      │
│                                                               │
│  测试 3: Local Router + 普通扫描（息屏状态）                   │
│  └─ 验证：普通扫描（非息屏扫描）在息屏时不应该工作             │
│  └─ 期望：❌ 不能扫描到路由（这是正确的行为）                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 实际应用场景

这些测试验证的功能用于：

**场景 1：智能手表控制手机音乐**
```
┌─────────────┐         ┌─────────────────┐
│  智能手表    │         │    手机          │
│             │         │                 │
│  手表应用    │ ──────→ │  音乐播放器      │
│  (Proxy     │  BT/WiFi│  (媒体播放)      │
│   Router)   │         │                 │
│             │         │  蓝牙耳机        │
│  息屏时也能  │         │  (路由设备)       │
│  扫描路由    │         │                 │
└─────────────┘         └─────────────────┘
```

**场景 2：车载系统**
```
┌─────────────────────────────────────────────┐
│              车载系统                         │
│                                             │
│  多个媒体应用：                               │
│  - Spotify                                   │
│  - 网易云音乐                                 │
│  - 播客应用                                   │
│                                             │
│  车载 MediaRouter (Proxy Router)             │
│  - 统一管理所有应用的路由                      │
│  - 屏幕关闭时仍能扫描新的蓝牙设备              │
│  - 需要 MEDIA_CONTENT_CONTROL 权限            │
└─────────────────────────────────────────────┘
```

---

## 三、失败原因深度分析

### 失败机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    Host 端 (PC)                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  JUnit 执行测试方法                                              │
│    ↓                                                             │
│  CheckFlagsRule.before()                                         │
│    ↓                                                             │
│  HostFlagsValueProvider.setUp()                                  │
│    ↓                                                             │
│  DeviceFlags.init() → getAconfigFlags()                          │
│    ↓                                                             │
│  HostDeviceProtos.parsedFlagsProtoPaths(device)                  │
│    ↓                                                             │
│  device.executeAdbCommand("cmd flag get ...")                    │
│    ↓                                                             │
│  NativeDevice.performDeviceAction()                              │
│    ↓                                                             │
│  AdbAction.run() → executeShellCommand()                         │
│    ↓                                                             │
│  ❌ ADB 命令失败（cmd: Can't find service: flag）                 │
│    ↓                                                             │
│  throw IOException                                               │
│    ↓                                                             │
│  throw DeviceUnresponsiveException[LOST_SYSTEM_UNDER_TEST]       │
│    ↓                                                             │
│  throw FlagReadException                                         │
│    ↓                                                             │
│  测试标记为 FAIL                                                  │
│                                                                  │
│  【测试方法体从未执行】                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 关键代码位置

| 层级 | 类名 | 关键方法 |
|---|---|---|
| 测试入口 | `MediaRouter2HostSideTest` | `@Rule CheckFlagsRule` |
| Flag 检查 | `CheckFlagsRule` | `before()` |
| Flag 提供 | `HostFlagsValueProvider` | `setUp()` |
| Flag 读取 | `DeviceFlags` | `getAconfigParsedFlags()` |
| Proto 路径 | `HostDeviceProtos` | `parsedFlagsProtoPaths()` |
| ADB 执行 | `NativeDevice` | `executeAdbCommand()` |

### 根本原因：服务名称不匹配

**执行命令：**
```bash
adb shell cmd flag get media_solutions/enable_screen_off_scanning
```

**错误输出：**
```
cmd: Can't find service: flag
```

**设备上的服务列表：**
```bash
$ adb shell service list | grep -Ei "device_config|flag|settings"
84      device_config: []
104     feature_flags: [android.flags.IFeatureFlags]
144     lock_settings: [com.android.internal.widget.ILockSettings]
227     settings: []
```

**结论：**
- 设备上有 `device_config` 服务（旧版命名）
- 设备上有 `feature_flags` 服务（新版接口）
- **没有** `flag` 服务

### 源码证据

从 AOSP 源码 `SettingsProvider.java` 中可以看到：

```java
// packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java

ServiceManager.addService("settings", new SettingsService(this));
ServiceManager.addService("device_config", new DeviceConfigService(this));
```

**服务注册名是 `device_config`，不是 `flag`。**

### DeviceConfigService 是什么？

```java
// packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java

/**
 * Receives shell commands from the command line related to device config flags,
 * and dispatches them to the SettingsProvider.
 */
public final class DeviceConfigService extends Binder {

    // aconfig flag proto 文件的存储位置
    private static final List<String> sAconfigTextProtoFilesOnDevice = List.of(
            "/system/etc/aconfig_flags.pb",
            "/system_ext/etc/aconfig_flags.pb",
            "/product/etc/aconfig_flags.pb",
            "/vendor/etc/aconfig_flags.pb");

    @Override
    public void onShellCommand(...) {
        // 处理 cmd device_config 命令
        (new MyShellCommand(mProvider))
                .exec(this, in, out, err, args, callback, resultReceiver);
    }
}
```

**DeviceConfigService 是 SettingsProvider 的一部分**，负责：
- 管理 aconfig flags 的读写
- 处理 `cmd device_config get/put/list/override` 等命令
- 从 `/system/etc/aconfig_flags.pb` 等文件加载 flag 定义

### 版本不匹配

```
┌─────────────────────────────────────────────────────────────┐
│                    AOSP 标准                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  服务注册名：device_config                                    │
│  命令：cmd device_config get <namespace> <key>               │
│  实现：DeviceConfigService (在 SettingsProvider 中)           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    CTS 工具链（新版本）                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  命令：cmd flag get <namespace> <key>                        │
│  服务名：flag                                                │
│  → 这是新版本的命名                                          │
│  → W 版本设备上不存在这个服务                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、验证方法

### 快速检查（2 分钟）

```bash
# 1. 检查设备是否在线
adb devices

# 2. 检查 device_config 服务是否存在
adb shell service list | grep device_config

# 3. 尝试用旧版命令读取 flag
adb shell cmd device_config get media_solutions enable_screen_off_scanning

# 4. 检查 aconfig flag 文件是否存在
adb shell ls -la /system/etc/aconfig_flags.pb
adb shell ls -la /vendor/etc/aconfig_flags.pb
```

### 手动执行测试核心逻辑（5 分钟）

绕过 CTS 框架，直接执行测试逻辑：

```bash
# 安装测试 APK
adb install CtsMediaRouterHostSideTestBluetoothPermissionsApp.apk

# 获取权限
adb shell pm grant android.media.router.cts.bluetoothpermissionsapp android.permission.DEVICE_POWER
adb shell pm grant android.media.router.cts.bluetoothpermissionsapp android.permission.MEDIA_CONTENT_CONTROL

# 息屏
adb shell input keyevent KEYCODE_POWER
sleep 2

# 通过 am instrument 运行单个测试
adb shell am instrument -w \
  -e class android.media.router.cts.bluetoothpermissionsapp.MediaRouter2DeviceTest#screenOffScan_onLocalRouter_allowedWithMediaContentControl \
  android.media.router.cts.bluetoothpermissionsapp.test/android.test.InstrumentationTestRunner
```

### 检查 Flag 是否存在

```bash
# 用旧版命令检查 flag
adb shell cmd device_config list | grep enable_screen_off_scanning
adb shell cmd device_config list | grep enable_full_scan_with_media_content_control

# 检查 flag 的值
adb shell cmd device_config get media_solutions enable_screen_off_scanning
adb shell cmd device_config get media_better_together enable_full_scan_with_media_content_control
```

---

## 五、解决方案

### 方案 1：确认 CTS 版本兼容性

```bash
# 确认 CTS 版本
./cts-tradefed --version

# 确认设备 Android 版本
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
```

如果 CTS 版本太新，需要使用与设备匹配的 CTS 版本。

### 方案 2：使用旧版命令验证

```bash
# 如果这个命令成功，说明 flag 存在，只是命名问题
adb shell cmd device_config get media_solutions enable_screen_off_scanning
```

### 方案 3：从测试计划中排除

如果 W 版本确实不支持这些测试，可以在测试计划中排除：

```xml
<Module name="CtsMediaHostTestCases">
  <Test name="android.media.router.cts.MediaRouter2HostSideTest">
    <Exclude name="screenOffScan_onProxyRouter_allowedWithMediaContentControl"/>
    <Exclude name="screenOffScan_onLocalRouter_allowedWithMediaContentControl"/>
    <Exclude name="requestScan_withOnScreenScan_withScreenOff_doesNotScan"/>
  </Test>
</Module>
```

---

## 六、结论

### 问题总结

| 项目 | 说明 |
|---|---|
| **失败类型** | CTS 工具链与设备服务名称不匹配 |
| **直接原因** | CTS 使用 `cmd flag` 命令，但设备上只有 `cmd device_config` |
| **失败阶段** | Test Setup（`CheckFlagsRule.setUp()` 读取 aconfig flags） |
| **测试是否执行** | 否，全部在 setup 阶段失败 |
| **根本原因** | CTS 工具链版本与 W 版本设备不兼容 |

### 与 android.xr 的关系

**没有直接关系。** 问题不是 system_server 崩溃，也不是设备无响应，而是：
- CTS 工具链期望的服务名：`flag`
- 设备实际的服务名：`device_config`

这是**命名不匹配**问题，不是系统功能缺失。

### 建议处理方案

1. **确认 CTS 版本与设备 Android 版本是否兼容**
2. **如果兼容**，检查是否需要更新 CTS 工具链或设备系统
3. **如果不兼容**，使用与设备匹配的 CTS 版本
4. **如果 W 版本确实不支持这些功能**，从测试计划中排除这些用例

---

## 七、相关源码文件

| 文件 | 路径 |
|------|------|
| MediaRouter2 | `frameworks/base/media/java/android/media/MediaRouter2.java` |
| MediaRouter2HostSideTest | `cts/hostsidetests/media/src/android/media/router/cts/MediaRouter2HostSideTest.java` |
| MediaRouter2DeviceTest | `cts/hostsidetests/media/app/MediaRouterWithBTPermissionsApp/src/android/media/router/cts/bluetoothpermissionsapp/MediaRouter2DeviceTest.java` |
| DeviceConfigService | `frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java` |
| SettingsProvider | `frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java` |
| aconfig flags 定义 | `frameworks/base/media/java/android/media/flags/media_better_together.aconfig` |

---

## 八、参考链接

- [AOSP MediaRouter2 文档](https://developer.android.com/reference/android/media/MediaRouter2)
- [aconfig 特性标志系统](https://source.android.com/docs/core/architecture/modular-system/aconfig)
- [CTS 测试框架](https://source.android.com/docs/compatibility/cts)

---

*文档创建时间：2026-06-16*  
*分析基于 AOSP W (Android 15) 版本*
