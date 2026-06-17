# AOSP MediaSession & MediaRouter CTS 失败用例分析

基于 AOSP W (Android 15) 版本的 CTS 失败用例深度分析。

## 概述

本文档库包含以下 CTS 失败用例分析：

### 1. CtsMediaSessionTestCases（已完成）

分析了以下 3 个测试用例的执行路径和失败原因：

- `testSetOnVolumeKeyLongPressListener`
- `testSetOnMediaKeyListener`  
- `testRemoteUserInfo`

### 2. CtsMediaHostTestCases - MediaRouter2（已完成）

分析了以下 3 个 MediaRouter2 息屏扫描测试用例：

- `MediaRouter2HostSideTest#screenOffScan_onProxyRouter_allowedWithMediaContentControl`
- `MediaRouter2HostSideTest#screenOffScan_onLocalRouter_allowedWithMediaContentControl`
- `MediaRouter2HostSideTest#requestScan_withOnScreenScan_withScreenOff_doesNotScan`

### 3. CtsMediaAudioTestCases - AudioPlaybackCapture（新增）

分析了以下测试用例：

- `AudioPlaybackCaptureTest#testCaptureExcludeUid`

## 文档结构

### MediaSession 相关
- `analysis.md` - 完整的执行路径分析、失败原因和日志添加建议
- `verification.md` - 快速验证方法（不需要修改 AOSP 源码）
- `log-analysis.md` - 实际日志分析结果和根本原因定位

### MediaRouter2 相关
- `cts-media-router-analysis.md` - MediaRouter2 息屏扫描测试失败分析
  - 测试功能说明（Local Router vs Proxy Router）
  - 失败原因深度分析（服务名称不匹配问题）
  - 验证方法和解决方案

### ADB 验证工具
- `adb-flag-verification.md` - ADB 快速验证 Flag 能力指南
  - 检查 Flag 服务是否可用
  - 读取 Flag 值的各种方法
  - 一键验证脚本
  - 常见问题排查

### AudioPlaybackCapture 相关（新增）
- `audio-playback-capture-analysis.md` - AudioPlaybackCaptureTest#testCaptureExcludeUid 失败分析
  - playback capture 动态 AudioPolicy 注册失败
  - excludeUid(0) 参数校验问题
  - MediaProjection 权限检查分析

### 通用
- `README.md` - 本文件

## 适用版本

- AOSP W (Android 15.0.0)
- 涉及仓库: `frameworks/base`

## 关键发现

### MediaSession 测试
- **根本原因：华为 InputDispatcher CTS 过滤**
  - 日志：`"In cts test running scenario, no need to dispatch key events."`
  - InputDispatcher 检测到 CTS 测试运行时丢弃所有按键事件
  - 影响所有依赖按键注入的 CTS 测试
- 关键文件：MediaSessionManager, MediaSessionService, PhoneWindowManager

### MediaRouter2 测试
- **失败原因：CTS 工具链与设备服务名称不匹配**
  - CTS 使用 `cmd flag` 命令
  - 设备实际服务名：`cmd device_config`
  - 导致测试在 setup 阶段就失败，测试逻辑从未执行
- **不是系统功能缺失，是命名不匹配问题**

### AudioPlaybackCapture 测试（新增）
- **失败原因：playback capture 动态 AudioPolicy 注册失败**
  - 异常：`UnsupportedOperationException: Error: could not register audio policy`
  - 最可疑原因：设备侧把 `excludeUid(0)` 中的 UID 0 误判为非法参数
  - 需要区分是 AudioService 权限拒绝还是 native registerPolicyMixes 失败
- 关键文件：AudioRecord, AudioService, AudioPolicyManager

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

# 检查 InputDispatcher CTS 过滤
adb logcat -c
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN
adb logcat -d | grep -i "cts test running"
```

### MediaRouter2 测试

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

### AudioPlaybackCapture 测试（新增）

```bash
# 抓取相关日志
adb logcat -b all -v threadtime | grep -iE \
"registerAudioPolicy|policy denied|connectMixes|registerPolicyMixes|remote_submix|submix|RULE_EXCLUDE_UID|exclude uid|uid 0|MediaProjection"

# 检查 remote_submix 模块
adb shell dumpsys media.audio_policy | grep -i submix

# 检查 MediaProjection 权限
adb shell dumpsys media_projection
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

### MediaRouter2
| 文件 | 路径 |
|------|------|
| MediaRouter2 | `media/java/android/media/MediaRouter2.java` |
| MediaRouter2HostSideTest | `cts/hostsidetests/media/src/android/media/router/cts/MediaRouter2HostSideTest.java` |
| MediaRouter2DeviceTest | `cts/hostsidetests/media/app/MediaRouterWithBTPermissionsApp/src/.../MediaRouter2DeviceTest.java` |
| DeviceConfigService | `packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java` |
| aconfig flags | `media/java/android/media/flags/media_better_together.aconfig` |

### AudioPlaybackCapture（新增）
| 文件 | 路径 |
|------|------|
| AudioRecord | `media/java/android/media/AudioRecord.java` |
| AudioPlaybackCaptureConfiguration | `media/java/android/media/AudioPlaybackCaptureConfiguration.java` |
| AudioService | `services/core/java/com/android/server/audio/AudioService.java` |
| AudioPolicyManager (native) | `services/audiopolicy/managerdefault/AudioPolicyManager.cpp` |

## 解决方案

### MediaSession 测试

**问题：** 华为 InputDispatcher 的 CTS 检测逻辑丢弃了所有按键事件

**解决方案：**
1. 修改 InputDispatcher，区分物理按键和注入事件
2. 通过系统属性禁用 CTS 过滤（如果支持）
3. 联系华为获取补丁

### MediaRouter2 测试

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

### AudioPlaybackCapture 测试（新增）

**问题：** playback capture 动态 AudioPolicy 注册失败，最可能是 `excludeUid(0)` 被误判为非法参数

**解决方案：**

1. **确认失败位置**
   - 如果日志显示 `policy denied`：问题在 AudioService 权限/MediaProjection 判断
   - 如果日志显示 `connectMixes failed`：问题在 native registerPolicyMixes()
   - 如果日志显示 `Unable to find audio module for submix`：remote_submix 配置缺失

2. **检查 UID 0 校验逻辑**
   - 搜索设备侧代码中是否有 `uid <= 0` 或 `uid == 0` 的非法判断
   - AOSP 允许 UID 0 作为 RULE_EXCLUDE_UID 的合法值

3. **如果其他 playback capture case 正常，仅该 case 失败**
   - 重点排查 `excludeUid(0)` 的参数校验
   - 检查 AudioMixingRule 中 UID 0 的处理逻辑

## License

MIT
