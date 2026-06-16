# AOSP MediaSession & MediaRouter CTS 失败用例分析

基于 AOSP W (Android 15) 版本的 CTS 失败用例深度分析。

## 概述

本文档库包含两部分 CTS 失败用例分析：

### 1. CtsMediaSessionTestCases（已完成）

分析了以下 3 个测试用例的执行路径和失败原因：

- `testSetOnVolumeKeyLongPressListener`
- `testSetOnMediaKeyListener`  
- `testRemoteUserInfo`

### 2. CtsMediaHostTestCases - MediaRouter2（新增）

分析了以下 3 个 MediaRouter2 息屏扫描测试用例：

- `MediaRouter2HostSideTest#screenOffScan_onProxyRouter_allowedWithMediaContentControl`
- `MediaRouter2HostSideTest#screenOffScan_onLocalRouter_allowedWithMediaContentControl`
- `MediaRouter2HostSideTest#requestScan_withOnScreenScan_withScreenOff_doesNotScan`

## 文档结构

### MediaSession 相关
- `analysis.md` - 完整的执行路径分析、失败原因和日志添加建议
- `verification.md` - 快速验证方法（不需要修改 AOSP 源码）
- `log-analysis.md` - 实际日志分析结果和根本原因定位

### MediaRouter2 相关（新增）
- `cts-media-router-analysis.md` - MediaRouter2 息屏扫描测试失败分析
  - 测试功能说明（Local Router vs Proxy Router）
  - 失败原因深度分析（服务名称不匹配问题）
  - 验证方法和解决方案

### ADB 验证工具（新增）
- `adb-flag-verification.md` - ADB 快速验证 Flag 能力指南
  - 检查 Flag 服务是否可用
  - 读取 Flag 值的各种方法
  - 一键验证脚本
  - 常见问题排查

### 通用
- `README.md` - 本文件

## 适用版本

- AOSP W (Android 15.0.0)
- 涉及仓库: `frameworks/base`

## 关键发现

### MediaSession 测试
- 失败原因：权限检查、用户设置状态、设备配置等
- 关键文件：MediaSessionManager, MediaSessionService, PhoneWindowManager

### MediaRouter2 测试（新增）
- **失败原因：CTS 工具链与设备服务名称不匹配**
  - CTS 使用 `cmd flag` 命令
  - 设备实际服务名：`cmd device_config`
  - 导致测试在 setup 阶段就失败，测试逻辑从未执行
- **不是系统功能缺失，是命名不匹配问题**

## 快速诊断

### MediaSession 测试

```bash
# 检查 shell 权限
adb shell dumpsys package com.android.shell | grep -E "SET_VOLUME_KEY|SET_MEDIA_KEY"

# 检查设备平台类型
adb shell dumpsys audio | grep "Platform Type"
adb shell getprop ro.build.characteristics

# 检查用户设置状态
adb shell settings get secure user_setup_complete

# 检查全局优先级 session
adb shell dumpsys media_session | grep -i "global"
```

### MediaRouter2 测试（新增）

```bash
# 检查 device_config 服务是否存在
adb shell service list | grep device_config

# 尝试用旧版命令读取 flag
adb shell cmd device_config get media_solutions enable_screen_off_scanning

# 检查 aconfig flag 文件是否存在
adb shell ls -la /system/etc/aconfig_flags.pb
adb shell ls -la /vendor/etc/aconfig_flags.pb

# 列出所有 device_config flags
adb shell cmd device_config list
```

## 关键文件

### MediaSession
| 文件 | 路径 |
|------|------|
| MediaSessionManager | `media/java/android/media/session/MediaSessionManager.java` |
| MediaSessionService | `services/core/java/com/android/server/media/MediaSessionService.java` |
| PhoneWindowManager | `services/core/java/com/android/server/policy/PhoneWindowManager.java` |
| AudioService | `services/core/java/com/android/server/audio/AudioService.java` |
| MediaSessionLegacyHelper | `media/java/android/media/session/MediaSessionLegacyHelper.java` |

### MediaRouter2（新增）
| 文件 | 路径 |
|------|------|
| MediaRouter2 | `media/java/android/media/MediaRouter2.java` |
| MediaRouter2HostSideTest | `cts/hostsidetests/media/src/android/media/router/cts/MediaRouter2HostSideTest.java` |
| MediaRouter2DeviceTest | `cts/hostsidetests/media/app/MediaRouterWithBTPermissionsApp/src/.../MediaRouter2DeviceTest.java` |
| DeviceConfigService | `packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java` |
| aconfig flags | `media/java/android/media/flags/media_better_together.aconfig` |

## 解决方案

### MediaRouter2 测试（新增）

**问题：** CTS 工具链使用 `cmd flag` 命令，但设备上只有 `cmd device_config` 服务

**解决方案：**

1. **确认 CTS 版本兼容性**
   ```bash
   ./cts-tradefed --version
   adb shell getprop ro.build.version.release
   ```

2. **使用旧版命令验证**
   ```bash
   adb shell cmd device_config get media_solutions enable_screen_off_scanning
   ```

3. **如果 W 版本不支持，从测试计划中排除**
   ```xml
   <Module name="CtsMediaHostTestCases">
     <Test name="android.media.router.cts.MediaRouter2HostSideTest">
       <Exclude name="screenOffScan_onProxyRouter_allowedWithMediaContentControl"/>
       <Exclude name="screenOffScan_onLocalRouter_allowedWithMediaContentControl"/>
       <Exclude name="requestScan_withOnScreenScan_withScreenOff_doesNotScan"/>
     </Test>
   </Module>
   ```

## License

MIT
