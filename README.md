# AOSP MediaSession CTS 失败用例分析

基于 AOSP W (Android 15) 版本的 CtsMediaSessionTestCases 失败用例深度分析。

## 概述

本文档分析了以下 3 个 CTS 测试用例的执行路径和失败原因：

- `testSetOnVolumeKeyLongPressListener`
- `testSetOnMediaKeyListener`  
- `testRemoteUserInfo`

## 文档结构

- `analysis.md` - 完整的执行路径分析、失败原因和日志添加建议
- `verification.md` - 快速验证方法（不需要修改 AOSP 源码）
- `README.md` - 本文件

## 适用版本

- AOSP W (Android 15.0.0)
- 涉及仓库: `frameworks/base`

## 关键文件

| 文件 | 路径 |
|------|------|
| MediaSessionManager | `media/java/android/media/session/MediaSessionManager.java` |
| MediaSessionService | `services/core/java/com/android/server/media/MediaSessionService.java` |
| PhoneWindowManager | `services/core/java/com/android/server/policy/PhoneWindowManager.java` |
| AudioService | `services/core/java/com/android/server/audio/AudioService.java` |
| MediaSessionLegacyHelper | `media/java/android/media/session/MediaSessionLegacyHelper.java` |

## 快速诊断

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

## License

MIT
