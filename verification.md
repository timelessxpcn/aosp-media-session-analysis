# 快速验证方法

在不修改 AOSP 源码的情况下，快速验证 `input keyevent --longpress` 是否正确注入了长按事件。

---

## 方法 1: 开启 InputDispatcher 详细日志 (推荐)

最快的方法，不需要改代码，直接看系统日志：

```bash
# 1. 开启 InputDispatcher 详细日志
adb shell dumpsys input enable-log

# 2. 清 logcat
adb logcat -c

# 3. 注入长按
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN

# 4. 看所有相关日志
adb logcat -d | grep -iE "InputDispatcher|PhoneWindowManager|AudioService|MediaSession" | tail -50
```

**观察要点**：
- 如果看到 `InputDispatcher: dispatchKeyEventToInputTargets` → 事件到达了 InputDispatcher
- 如果看到 `PhoneWindowManager: interceptKeyBeforeQueueing` → 事件到达了 WindowManager
- 如果看到 `Volume key routed to AudioService directly` → TV 路由问题
- 如果完全没有 PhoneWindowManager 的日志 → 事件没到达 WindowManager

---

## 方法 2: dumpsys input 查看事件历史

```bash
# 注入长按
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN

# 立即查看 input 子系统状态
adb shell dumpsys input | grep -A 20 "Most Recent Events" | grep -i "volume\|long\|repeat\|flag"
```

**观察要点**：
- 查看事件的 `flags` 字段
- 查看 `repeatCount` 是否 > 0
- 查看是否有 `FLAG_LONG_PRESS` (0x80)

---

## 方法 3: 对比物理按键和注入按键

如果有物理音量键，对比两种方式的事件：

```bash
echo "=== 测试 1: input 命令注入长按 ==="
adb logcat -c
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN
adb logcat -d | grep -iE "volume|longpress|media|intercept" | head -20

echo ""
echo "=== 测试 2: 手动长按物理音量键 ==="
adb logcat -c
# 手动长按设备上的音量键 3 秒
adb logcat -d | grep -iE "volume|longpress|media|intercept" | head -20
```

**对比要点**：
- 事件序列是否一致
- repeatCount 是否相同
- flags 是否相同
- 系统处理路径是否相同

---

## 方法 4: 写一个 KeyLogger App (最准确)

如果能安装 app 到设备，这是最准确的方法：

**KeyLoggerActivity.java**:
```java
package com.example.keylogger;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.KeyEvent;

public class KeyLoggerActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        Log.d("KeyLogger", "keyCode=" + event.getKeyCode()
            + ", action=" + event.getAction()
            + ", repeatCount=" + event.getRepeatCount()
            + ", flags=0x" + Integer.toHexString(event.getFlags())
            + ", FLAG_LONG_PRESS=" + ((event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0)
            + ", FLAG_CANCELED=" + ((event.getFlags() & KeyEvent.FLAG_CANCELED) != 0));
        return super.dispatchKeyEvent(event);
    }
}
```

**运行测试**:
```bash
# 安装并启动 app
adb install keylogger.apk
adb shell am start -n com.example.keylogger/.KeyLoggerActivity

# 清日志
adb logcat -c

# 注入长按
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN

# 查看日志
adb logcat -d -s KeyLogger:*
```

**期望输出**:
```
D/KeyLogger: keyCode=25, action=0, repeatCount=0, flags=0x0, FLAG_LONG_PRESS=false
D/KeyLogger: keyCode=25, action=0, repeatCount=1, flags=0x80, FLAG_LONG_PRESS=true
D/KeyLogger: keyCode=25, action=0, repeatCount=2, flags=0x80, FLAG_LONG_PRESS=true
D/KeyLogger: keyCode=25, action=1, repeatCount=0, flags=0x0, FLAG_LONG_PRESS=false
```

**判断标准**：
- 如果 `FLAG_LONG_PRESS=true` 且 `repeatCount >= 1` → 注入正确
- 如果 `FLAG_LONG_PRESS=false` → 注入有问题
- 如果完全没有事件 → 事件被拦截了

---

## 方法 5: 手动模拟事件序列

如果怀疑 `input keyevent --longpress` 的实现有问题，可以手动发送：

```bash
# 方法 A: 使用 input swipe 模拟长按（不推荐，会生成触摸事件）
adb shell input swipe 100 100 100 100 2000

# 方法 B: 使用 sendevent 直接注入（需要知道设备节点）
# 先找到输入设备
adb shell getevent -pl

# 然后手动发送事件（示例）
adb shell sendevent /dev/input/event0 1 114 1  # KEY_VOLUMEDOWN down
adb shell sleep 1
adb shell sendevent /dev/input/event0 1 114 0  # KEY_VOLUMEDOWN up

# 方法 C: 使用 instrument 注入（CTS 实际使用的方式）
adb shell am instrument -w -e class android.media.session.cts.MediaSessionManagerTest#testSetOnVolumeKeyLongPressListener com.android.cts.media/androidx.test.runner.AndroidJUnitRunner
```

---

## 方法 6: 检查 input 命令的实现

查看 `input` 命令的源码，确认 `--longpress` 的实现：

```bash
# 在 AOSP 源码中
find frameworks/base -name "InputShell.java" -o -name "Input.java" | xargs grep -A 20 "longpress"
```

**关键代码** (frameworks/base/cmds/input/src/com/android/commands/input/Input.java):
```java
private void sendKeyEvent(int keyCode, boolean longpress) {
    // ...
    if (longpress) {
        // 发送 ACTION_DOWN
        event = new KeyEvent(downTime, when, KeyEvent.ACTION_DOWN, keyCode, 0);
        InputManager.getInstance().injectInputEvent(event, ...);
        
        // 发送 ACTION_DOWN with repeat
        when += 500;  // 长按延迟
        event = new KeyEvent(downTime, when, KeyEvent.ACTION_DOWN, keyCode, 1);
        event.setFlags(KeyEvent.FLAG_LONG_PRESS);
        InputManager.getInstance().injectInputEvent(event, ...);
        
        // 发送 ACTION_UP
        when += 100;
        event = new KeyEvent(downTime, when, KeyEvent.ACTION_UP, keyCode, 0);
        InputManager.getInstance().injectInputEvent(event, ...);
    }
}
```

**验证点**：
- `repeatCount` 是否正确设置为 1
- `FLAG_LONG_PRESS` 是否正确设置
- 事件间隔是否合理

---

## 快速诊断脚本

把上面的方法整合成一个脚本：

```bash
#!/bin/bash

echo "=========================================="
echo "CTS MediaSession 快速诊断"
echo "=========================================="
echo ""

# 清理
adb logcat -c
adb shell dumpsys input enable-log > /dev/null 2>&1

echo "1. 注入长按事件..."
adb shell input keyevent --longpress KEYCODE_VOLUME_DOWN

echo ""
echo "2. 检查 InputDispatcher 日志..."
adb logcat -d | grep -iE "InputDispatcher.*volume|dispatchKeyEventToInputTargets" | tail -5

echo ""
echo "3. 检查 PhoneWindowManager 日志..."
adb logcat -d | grep -iE "PhoneWindowManager.*volume|interceptKeyBeforeQueueing" | tail -5

echo ""
echo "4. 检查 AudioService 日志..."
adb logcat -d | grep -iE "AudioService.*volume|handleVolumeKey" | tail -5

echo ""
echo "5. 检查 MediaSessionService 日志..."
adb logcat -d | grep -iE "MediaSessionService.*volume|dispatchVolumeKeyEvent|setOnVolumeKeyLongPressListener" | tail -5

echo ""
echo "6. 检查 dumpsys input 事件历史..."
adb shell dumpsys input | grep -A 30 "Most Recent Events" | grep -i "volume" | tail -10

echo ""
echo "=========================================="
echo "诊断完成"
echo "=========================================="
```

保存为 `diagnose.sh`，然后：
```bash
chmod +x diagnose.sh
./diagnose.sh
```

---

## 判断标准总结

| 现象 | 结论 |
|------|------|
| 完全没有 InputDispatcher 日志 | 事件注入失败 |
| 有 InputDispatcher 日志，但没有 PhoneWindowManager 日志 | 事件被其他组件拦截 |
| 有 PhoneWindowManager 日志，显示 "routed to AudioService directly" | TV 路由问题 (mUseTvRouting=true) |
| 有 PhoneWindowManager 日志，显示 "passed to MediaSessionLegacyHelper" | 事件到达了 MediaSessionService |
| 有 MediaSessionService 日志，显示 "listener=null" | listener 注册失败（权限问题） |
| 有 MediaSessionService 日志，显示 "BLOCKED - user setup not complete" | user_setup_complete 未设置 |
| 有 MediaSessionService 日志，显示 "BLOCKED - global priority active" | 有全局优先级 session |
| 有 MediaSessionService 日志，显示 "Callback sent successfully" | 回调成功，问题在客户端 |

根据这些判断标准，快速定位问题所在。
