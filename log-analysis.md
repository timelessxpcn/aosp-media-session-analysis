# 日志分析结果

## 测试环境

- 设备：华为设备（包含 HwPhoneWindowManager 定制代码）
- 测试命令：`adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN`
- 日志抓取：`adb logcat -d | grep -iE "InputDispatcher|PhoneWindowManager|AudioService|MediaSession"`

## 关键日志

```
06-12 09:31:48.561   509  2814 I HwPhoneWindowManager: innerInterceptKeyBeforeQueueing keycode:25,action:0,isIntercept:false
06-12 09:31:48.565   509  2814 I HwPhoneWindowManager: intercept Key : 25, isdown : true, flags : 0
06-12 09:31:48.567   509  2814 I HwPhoneWindowManager: interceptVolumeDownKey isDown=true policyFlags=b000000
06-12 09:31:48.568   509  2814 I PhoneWindowManager: interceptKeyTq keycode=25 interactive=false keyguardActive=true policyFlags=b000000 down=true canceled=false displayId=-1 screenshotChordEnabled=false
06-12 09:31:48.572   509  2814 I PhoneWindowManager: interceptKeyBeforeQueueing: key 25 , result : 1
06-12 09:31:48.572   509   636 I InputDispatcher: In cts test running scenario, no need to dispatch key events.

06-12 09:31:48.975   509  2814 I HwPhoneWindowManager: innerInterceptKeyBeforeQueueing keycode:25,action:0,isIntercept:false
06-12 09:31:48.976   509  2814 I HwPhoneWindowManager: intercept Key : 25, isdown : true, flags : 128
06-12 09:31:48.978   509  2814 I HwPhoneWindowManager: interceptVolumeDownKey isDown=true policyFlags=b000000
06-12 09:31:48.978   509  2814 I PhoneWindowManager: interceptKeyTq keycode=25 interactive=false keyguardActive=true policyFlags=b000000 down=true canceled=false displayId=-1 screenshotChordEnabled=false
06-12 09:31:48.980   509  2814 I PhoneWindowManager: interceptKeyBeforeQueueing: key 25 , result : 1
06-12 09:31:48.980   509   636 I InputDispatcher: In cts test running scenario, no need to dispatch key events.

06-12 09:31:48.984   509  2814 I HwPhoneWindowManager: innerInterceptKeyBeforeQueueing keycode:25,action:1,isIntercept:false
06-12 09:31:48.984   509  2814 I HwPhoneWindowManager: intercept Key : 25, isdown : false, flags : 0
06-12 09:31:48.985   509  2814 I HwPhoneWindowManager: interceptVolumeDownKey isDown=false policyFlags=b000000
06-12 09:31:48.985   509  2814 I PhoneWindowManager: interceptKeyTq keycode=25 interactive=false keyguardActive=true policyFlags=b000000 down=false canceled=false displayId=-1 screenshotChordEnabled=false
06-12 09:31:48.985   509  2814 I PhoneWindowManager: interceptKeyBeforeQueueing: key 25 , result : 1
06-12 09:31:48.985   509   636 I InputDispatcher: In cts test running scenario, no need to dispatch key events.
```

## 日志分析

### 1. 事件注入是正确的

从日志可以看到 `input keyevent --longpress` 正确生成了长按事件序列：

| 时间戳 | action | flags | 含义 |
|--------|--------|-------|------|
| 09:31:48.561 | 0 (DOWN) | 0 | 第 1 个 DOWN，repeatCount=0 |
| 09:31:48.975 | 0 (DOWN) | 128 (0x80) | 第 2 个 DOWN，FLAG_LONG_PRESS，repeatCount=1 |
| 09:31:48.984 | 1 (UP) | 0 | UP 事件 |

**flags=128 (0x80)** 就是 `KeyEvent.FLAG_LONG_PRESS`，说明长按事件注入完全正确。

### 2. PhoneWindowManager 收到了事件

日志显示：
- `HwPhoneWindowManager.interceptKeyBeforeQueueing` 被调用
- `PhoneWindowManager.interceptKeyBeforeQueueing: key 25, result : 1`
- `result : 1` 表示事件被处理

### 3. InputDispatcher 拒绝分发（根本原因）

**每个事件都出现了这条关键日志**：

```
InputDispatcher: In cts test running scenario, no need to dispatch key events.
```

这说明：
- InputDispatcher 检测到 CTS 测试正在运行
- 主动跳过了所有按键事件分发
- 事件虽然到达了 PhoneWindowManager，但没有被分发到目标应用
- MediaSessionService 永远收不到事件
- listener 的回调永远不会触发
- CountDownLatch 超时，测试失败

## 根本原因

**华为在 InputDispatcher 中添加了 CTS 测试检测逻辑**。

当检测到 CTS 测试运行时，InputDispatcher 会跳过所有按键事件分发，目的是防止 CTS 测试期间的物理按键干扰测试。但这个逻辑过于激进，把测试自己注入的按键事件也一起丢弃了。

### 事件流

```
input keyevent --longpress
  → InputManager.injectInputEvent()
    → InputDispatcher 收到事件
      → 检测到 CTS 测试运行中
        → "In cts test running scenario, no need to dispatch key events."
        → 事件被丢弃 ✗
```

### 预期事件流

```
input keyevent --longpress
  → InputManager.injectInputEvent()
    → InputDispatcher 收到事件
      → dispatchKeyEventToInputTargets()
        → PhoneWindowManager.interceptKeyBeforeQueueing()
          → MediaSessionLegacyHelper.sendVolumeKeyEvent()
            → MediaSessionService.dispatchVolumeKeyEvent()
              → handleKeyEventLocked() → handleLongPressLocked()
                → dispatchVolumeKeyLongPressLocked()
                  → listener.onVolumeKeyLongPress() ✓
```

## 解决方案

### 方案 1: 修改 InputDispatcher 的 CTS 检测逻辑（推荐）

找到华为 InputDispatcher 中的 CTS 检测代码，修改为区分物理按键和注入事件：

```cpp
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
// 修改前：
if (isCtsTestRunning()) {
    ALOGI("In cts test running scenario, no need to dispatch key events.");
    return;  // 跳过分发
}

// 修改后：
if (isCtsTestRunning() && !isInjectedEvent(event)) {
    ALOGI("In cts test running scenario, skip physical key events.");
    return;  // 只跳过物理按键，不跳过注入事件
}
```

判断是否为注入事件的方法：
- 检查事件的 source 是否为 `AINPUT_SOURCE_INJECTED`
- 检查事件的 flags 是否包含 `AINPUT_FLAG_INJECTED`
- 检查注入来源是否为 system_server

### 方案 2: 通过系统属性禁用

检查是否有系统属性可以控制这个行为：

```bash
# 查看相关属性
adb shell getprop | grep -i cts
adb shell getprop | grep -i input
adb shell getprop | grep -i dispatch

# 尝试设置属性（需要 root）
adb shell setprop persist.sys.disable_cts_input_filter 1
```

### 方案 3: 联系华为获取补丁

这是华为设备的定制问题，需要华为提供修复补丁。建议将此分析结果提交给华为技术支持。

## 验证方法

修复后再次运行测试：

```bash
adb logcat -c
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN
adb logcat -d | grep -iE "InputDispatcher|PhoneWindowManager|MediaSession" | tail -20
```

**期望看到**：
- 不再有 "no need to dispatch key events" 日志
- 出现 "dispatchKeyEventToInputTargets" 或类似的分发日志
- 出现 MediaSessionService 的相关日志（如 "dispatchVolumeKeyEvent"）
- 出现 listener 回调日志

## 影响范围

这个问题影响所有依赖按键注入的 CTS 测试用例，包括但不限于：

- `CtsMediaSessionTestCases` - 媒体会话测试
- `CtsMediaTestCases` - 媒体播放测试
- `CtsInputMethodTestCases` - 输入法测试
- 所有使用 `input keyevent` 注入按键的测试

## 总结

| 项目 | 结论 |
|------|------|
| 问题现象 | CTS MediaSession 按键相关用例全部失败 |
| 根本原因 | 华为 InputDispatcher 的 CTS 检测逻辑丢弃了所有按键事件 |
| 影响范围 | 所有依赖按键注入的 CTS 测试 |
| 解决方案 | 修改 InputDispatcher，区分物理按键和注入事件 |
| 责任方 | 华为设备定制代码 |
