# CTS MediaSession 失败用例分析 (AOSP W / Android 15)

## 失败用例总览

| 用例 | 失败行号 | 失败信息 |
|------|---------|---------|
| testSetOnVolumeKeyLongPressListener | 335 | expected to be true |
| testSetOnMediaKeyListener | 388 | expected to be true |
| testRemoteUserInfo | 465 | expected to be true |

**共同特征**：CountDownLatch 在 TIMEOUT_MS (3000ms) 内没有 countDown 到 0，意味着注册的 listener/callback 从未被回调。

---

## 用例 1: testSetOnVolumeKeyLongPressListener

### 执行路径

#### 阶段 1: Listener 注册

**文件**: `frameworks/base/media/java/android/media/session/MediaSessionManager.java`  
**行号**: 793-812

```java
public void setOnVolumeKeyLongPressListener(
        OnVolumeKeyLongPressListener listener, @Nullable Handler handler) {
    synchronized (mLock) {
        try {
            if (listener == null) {
                mOnVolumeKeyLongPressListener = null;
                mService.setOnVolumeKeyLongPressListener(null);
            } else {
                if (handler == null) {
                    handler = new Handler();
                }
                mOnVolumeKeyLongPressListener =
                        new OnVolumeKeyLongPressListenerImpl(listener, handler);
                mService.setOnVolumeKeyLongPressListener(mOnVolumeKeyLongPressListener);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to set volume key long press listener", e);
        }
    }
}
```

→ Binder 调用到 system_server

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionService.java`  
**行号**: 2002-2059

```java
public void setOnVolumeKeyLongPressListener(IOnVolumeKeyLongPressListener listener) {
    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        // 权限检查
        if (mContext.checkPermission(
                android.Manifest.permission.SET_VOLUME_KEY_LONG_PRESS_LISTENER, pid, uid)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Must hold the SET_VOLUME_KEY_LONG_PRESS_LISTENER"
                    + " permission.");
        }

        synchronized (mLock) {
            int userId = UserHandle.getUserHandleForUid(uid).getIdentifier();
            FullUserRecord user = getFullUserRecordLocked(userId);
            if (user == null || user.mFullUserId != userId) {
                Log.w(TAG, "Only the full user can set the volume key long-press listener"
                        + ", userId=" + userId);
                return;
            }
            if (user.mOnVolumeKeyLongPressListener != null
                    && user.mOnVolumeKeyLongPressListenerUid != uid) {
                Log.w(TAG, "The volume key long-press listener cannot be reset"
                        + " by another app , mOnVolumeKeyLongPressListener="
                        + user.mOnVolumeKeyLongPressListenerUid
                        + ", uid=" + uid);
                return;
            }

            user.mOnVolumeKeyLongPressListener = listener;
            user.mOnVolumeKeyLongPressListenerUid = uid;

            Log.d(TAG, "The volume key long-press listener "
                    + listener + " is set by " + getCallingPackageName(uid));

            // linkToDeath 注册
            if (user.mOnVolumeKeyLongPressListener != null) {
                try {
                    user.mOnVolumeKeyLongPressListener.asBinder().linkToDeath(...);
                } catch (RemoteException e) {
                    Log.w(TAG, "Failed to set death recipient "
                            + user.mOnVolumeKeyLongPressListener);
                    user.mOnVolumeKeyLongPressListener = null;
                }
            }
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

#### 阶段 2: 按键注入与分发

测试代码:
```java
injectInputEvent(KeyEvent.KEYCODE_VOLUME_DOWN, true);
// 执行: adb shell input keyevent --longpress 25
```

InputManager 注入事件序列:
- ACTION_DOWN (repeatCount=0)
- ACTION_DOWN (repeatCount=1, FLAG_LONG_PRESS)
- ACTION_DOWN (repeatCount=2, FLAG_LONG_PRESS)
- ...
- ACTION_UP

**文件**: `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`  
**行号**: 2395-2398 (初始化时读取配置)

```java
mUseTvRouting = AudioSystem.getPlatformType(mContext) == AudioSystem.PLATFORM_TELEVISION;

mHandleVolumeKeysInWM = mContext.getResources().getBoolean(
        com.android.internal.R.bool.config_handleVolumeKeysInWindowManager);
```

**行号**: 3820-3828 (interceptKeyBeforeQueueing 中处理音量键)

```java
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_MUTE:
    if (mUseTvRouting || mHandleVolumeKeysInWM) {
        // On TVs or when the configuration is enabled, volume keys never
        // go to the foreground app.
        dispatchDirectAudioEvent(event);
        return true;  // ← 事件被消费，不传递给应用
    }
    // ... 否则 break，继续往下走
```

**行号**: 5344-5412 (interceptKeyBeforeQueueing 中音量键的详细处理)

```java
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_MUTE: {
    // ... 各种检查 (来电、通话模式等)
    
    if (mUseTvRouting || mHandleVolumeKeysInWM) {
        // Defer special key handlings to interceptKeysBeforeDispatching().
        result |= ACTION_PASS_TO_USER;
    } else if ((result & ACTION_PASS_TO_USER) == 0) {
        // If we aren't passing to the user and no one else
        // handled it send it to the session manager to
        // figure out.
        MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
                event, AudioManager.USE_DEFAULT_STREAM_TYPE, true);
    }
    break;
}
```

**行号**: 5931-5952 (dispatchDirectAudioEvent)

```java
private void dispatchDirectAudioEvent(KeyEvent event) {
    // ... HDMI 检查
    try {
        getAudioService().handleVolumeKey(event, mUseTvRouting,
                mContext.getOpPackageName(), TAG);
    } catch (Exception e) {
        Log.e(TAG, "Error dispatching volume key in handleVolumeKey for event:"
                + event, e);
    }
}
```

**文件**: `frameworks/base/services/core/java/com/android/server/audio/AudioService.java`  
**行号**: 3633-3675 (handleVolumeKey)

```java
public void handleVolumeKey(@NonNull KeyEvent event, boolean isOnTv,
        @NonNull String callingPackage, @NonNull String caller) {
    int keyEventMode = AudioDeviceVolumeManager.ADJUST_MODE_NORMAL;
    if (isOnTv) {
        if (event.getAction() == KeyEvent.ACTION_DOWN) {
            keyEventMode = AudioDeviceVolumeManager.ADJUST_MODE_START;
        } else {
            keyEventMode = AudioDeviceVolumeManager.ADJUST_MODE_END;
        }
    } else if (event.getAction() != KeyEvent.ACTION_DOWN) {
        return;  // ← 非 TV 模式下，只处理 ACTION_DOWN
    }

    int flags = AudioManager.FLAG_SHOW_UI | AudioManager.FLAG_PLAY_SOUND
            | AudioManager.FLAG_FROM_KEY;

    switch (event.getKeyCode()) {
        case KeyEvent.KEYCODE_VOLUME_UP:
            adjustSuggestedStreamVolume(AudioManager.ADJUST_RAISE, ...);
            break;
        case KeyEvent.KEYCODE_VOLUME_DOWN:
            adjustSuggestedStreamVolume(AudioManager.ADJUST_LOWER, ...);
            break;
        // ...
    }
}
```

#### 阶段 3: MediaSessionService 的长按检测 (仅在非 TV 路由时)

如果 mUseTvRouting=false 且 mHandleVolumeKeysInWM=false，音量键会走到:

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionService.java`  
**行号**: 2143-2179 (dispatchVolumeKeyEvent)

```java
public void dispatchVolumeKeyEvent(String packageName, String opPackageName,
        boolean asSystemService, KeyEvent keyEvent, int stream, boolean musicOnly) {
    // ... 参数检查
    try {
        synchronized (mLock) {
            if (isGlobalPriorityActiveLocked()) {
                dispatchVolumeKeyEventLocked(...);
            } else {
                // ← 走这个分支
                mVolumeKeyEventHandler.handleVolumeKeyEventLocked(packageName, pid, uid,
                        asSystemService, keyEvent, opPackageName, stream, musicOnly);
            }
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

**行号**: 2934-2939 (handleVolumeKeyEventLocked)

```java
void handleVolumeKeyEventLocked(String packageName, int pid, int uid,
        boolean asSystemService, KeyEvent keyEvent, String opPackageName, int stream,
        boolean musicOnly) {
    handleKeyEventLocked(packageName, pid, uid, asSystemService, keyEvent, false,
            opPackageName, stream, musicOnly);
}
```

**行号**: 2941-3022 (handleKeyEventLocked - 核心逻辑)

```java
void handleKeyEventLocked(String packageName, int pid, int uid,
        boolean asSystemService, KeyEvent keyEvent, boolean needWakeLock,
        String opPackageName, int stream, boolean musicOnly) {
    if (keyEvent.isCanceled()) {
        return;
    }

    int overriddenKeyEvents = 0;
    if (mCustomMediaKeyDispatcher != null
            && mCustomMediaKeyDispatcher.getOverriddenKeyEvents() != null) {
        overriddenKeyEvents = mCustomMediaKeyDispatcher.getOverriddenKeyEvents()
                .get(keyEvent.getKeyCode());
    }
    cancelTrackingIfNeeded(...);
    
    if (!needTracking(keyEvent, overriddenKeyEvents)) {
        // ← 如果没有 listener，直接分发音量事件
        if (mKeyType == KEY_TYPE_VOLUME) {
            dispatchVolumeKeyEventLocked(...);
        } else {
            dispatchMediaKeyEventLocked(...);
        }
        return;
    }

    if (isFirstDownKeyEvent(keyEvent)) {
        mTrackingFirstDownKeyEvent = keyEvent;
        mIsLongPressing = false;
        return;  // ← 第1个 ACTION_DOWN，记录后返回
    }

    // Long press is always overridden here
    if (isFirstLongPressKeyEvent(keyEvent)) {
        mIsLongPressing = true;
    }
    if (mIsLongPressing) {
        handleLongPressLocked(keyEvent, needWakeLock, overriddenKeyEvents);
        return;
    }
    // ... 后续处理
}
```

**行号**: 3077-3096 (needTracking)

```java
private boolean needTracking(KeyEvent keyEvent, int overriddenKeyEvents) {
    if (!isFirstDownKeyEvent(keyEvent)) {
        if (mTrackingFirstDownKeyEvent == null) {
            return false;
        } else if (mTrackingFirstDownKeyEvent.getDownTime() != keyEvent.getDownTime()
                || mTrackingFirstDownKeyEvent.getKeyCode() != keyEvent.getKeyCode()) {
            return false;
        }
    }
    if (overriddenKeyEvents == 0) {
        if (mKeyType == KEY_TYPE_VOLUME) {
            if (mCurrentFullUserRecord.mOnVolumeKeyLongPressListener == null) {
                return false;  // ← 如果没有 listener，不需要跟踪
            }
        } else if (!isVoiceKey(keyEvent.getKeyCode())) {
            return false;
        }
    }
    return true;
}
```

**行号**: 3109-3139 (handleLongPressLocked)

```java
private void handleLongPressLocked(KeyEvent keyEvent, boolean needWakeLock,
        int overriddenKeyEvents) {
    if (mCustomMediaKeyDispatcher != null
            && isLongPressOverridden(overriddenKeyEvents)) {
        // 自定义分发器处理
        mCustomMediaKeyDispatcher.onLongPress(keyEvent);
        // ...
    } else {
        if (mKeyType == KEY_TYPE_VOLUME) {
            if (isFirstLongPressKeyEvent(keyEvent)) {
                dispatchVolumeKeyLongPressLocked(mTrackingFirstDownKeyEvent);
            }
            dispatchVolumeKeyLongPressLocked(keyEvent);  // ← 回调 listener
        } else if (isFirstLongPressKeyEvent(keyEvent)
                && isVoiceKey(keyEvent.getKeyCode())) {
            startVoiceInput(needWakeLock);
            resetLongPressTracking();
        }
    }
}
```

**行号**: 1172-1181 (dispatchVolumeKeyLongPressLocked)

```java
private void dispatchVolumeKeyLongPressLocked(KeyEvent keyEvent) {
    if (mCurrentFullUserRecord.mOnVolumeKeyLongPressListener == null) {
        return;
    }
    try {
        mCurrentFullUserRecord.mOnVolumeKeyLongPressListener.onVolumeKeyLongPress(keyEvent);
        // ← Binder 回调到测试进程
    } catch (RemoteException e) {
        Log.w(TAG, "Failed to send " + keyEvent + " to volume key long-press listener");
    }
}
```

#### 阶段 4: 回调到测试进程

**文件**: `frameworks/base/media/java/android/media/session/MediaSessionManager.java`  
(OnVolumeKeyLongPressListenerImpl 内部类)

```java
private class OnVolumeKeyLongPressListenerImpl extends IOnVolumeKeyLongPressListener.Stub {
    private final OnVolumeKeyLongPressListener mListener;
    private final Handler mHandler;

    OnVolumeKeyLongPressListenerImpl(OnVolumeKeyLongPressListener listener, Handler handler) {
        mListener = listener;
        mHandler = handler;
    }

    @Override
    public void onVolumeKeyLongPress(KeyEvent event) {
        mHandler.post(() -> mListener.onVolumeKeyLongPress(event));
        // → 测试代码中的 VolumeKeyLongPressListener.onVolumeKeyLongPress()
        // → CountDownLatch.countDown()
    }
}
```

### 可能失败原因

**原因 A (最可能): mUseTvRouting = true**
- PhoneWindowManager line 2395:
  ```java
  mUseTvRouting = AudioSystem.getPlatformType(mContext) == PLATFORM_TELEVISION
  ```
- 如果设备被识别为 TV 平台，音量键直接走 AudioService.handleVolumeKey()
- 永远不会到达 MediaSessionService 的长按检测逻辑
- 测试只检查了 FEATURE_LEANBACK，但 mUseTvRouting 不依赖 FEATURE_LEANBACK
- 某些设备可能是 TV 平台但没有声明 LEANBACK feature

**原因 B: 前台 Activity 消费了音量键**
- 如果 CTS 测试的 Activity 重写了 onKeyDown 并消费了音量键
- PhoneWindowManager 会将 result &= ~ACTION_PASS_TO_USER
- 但在 line 5405 的 else if 分支中才会发给 MediaSessionLegacyHelper
- 如果 Activity 消费了事件，就不会走到 MediaSessionService

**原因 C: 权限静默失败**
- 虽然 privapp-permissions-platform.xml 中 shell 有这些权限
- 但如果设备厂商修改了该配置，shell 可能没有 SET_VOLUME_KEY_LONG_PRESS_LISTENER
- SecurityException 在 Binder 传输中变为 RemoteException
- 客户端 MediaSessionManager line 808-810 捕获并 log，不抛出
- listener 实际未注册成功，后续回调自然不会来

**原因 D: input keyevent --longpress 事件序列问题**
- 注入的事件可能没有正确设置 FLAG_LONG_PRESS
- isFirstLongPressKeyEvent() 要求 FLAG_LONG_PRESS && repeatCount==1
- 如果 FLAG_LONG_PRESS 未被设置，长按检测不会触发

---

## 用例 2: testSetOnMediaKeyListener

### 执行路径

#### 阶段 1: Listener 注册

**文件**: `frameworks/base/media/java/android/media/session/MediaSessionManager.java`  
**行号**: 829-846

```java
public void setOnMediaKeyListener(OnMediaKeyListener listener, @Nullable Handler handler) {
    synchronized (mLock) {
        try {
            if (listener == null) {
                mOnMediaKeyListener = null;
                mService.setOnMediaKeyListener(null);
            } else {
                if (handler == null) {
                    handler = new Handler();
                }
                mOnMediaKeyListener = new OnMediaKeyListenerImpl(listener, handler);
                mService.setOnMediaKeyListener(mOnMediaKeyListener);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to set media key listener", e);
        }
    }
}
```

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionService.java`  
**行号**: 2062-2115

```java
public void setOnMediaKeyListener(IOnMediaKeyListener listener) {
    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        // 权限检查
        if (mContext.checkPermission(
                android.Manifest.permission.SET_MEDIA_KEY_LISTENER, pid, uid)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Must hold the SET_MEDIA_KEY_LISTENER permission.");
        }

        synchronized (mLock) {
            int userId = UserHandle.getUserHandleForUid(uid).getIdentifier();
            FullUserRecord user = getFullUserRecordLocked(userId);
            if (user == null || user.mFullUserId != userId) {
                Log.w(TAG, "Only the full user can set the media key listener"
                        + ", userId=" + userId);
                return;
            }
            if (user.mOnMediaKeyListener != null && user.mOnMediaKeyListenerUid != uid) {
                Log.w(TAG, "The media key listener cannot be reset by another app. "
                        + ", mOnMediaKeyListenerUid=" + user.mOnMediaKeyListenerUid
                        + ", uid=" + uid);
                return;
            }

            user.mOnMediaKeyListener = listener;
            user.mOnMediaKeyListenerUid = uid;

            Log.d(TAG, "The media key listener " + user.mOnMediaKeyListener
                    + " is set by " + getCallingPackageName(uid));

            // linkToDeath 注册
            if (user.mOnMediaKeyListener != null) {
                try {
                    user.mOnMediaKeyListener.asBinder().linkToDeath(...);
                } catch (RemoteException e) {
                    Log.w(TAG, "Failed to set death recipient " + user.mOnMediaKeyListener);
                    user.mOnMediaKeyListener = null;
                }
            }
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

#### 阶段 2: 按键注入与分发

测试代码:
```java
injectInputEvent(KeyEvent.KEYCODE_HEADSETHOOK, false);
// 执行: adb shell input keyevent 79
```

InputManager 注入事件序列:
- ACTION_DOWN (repeatCount=0)
- ACTION_UP (repeatCount=0)

**文件**: `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`  
**行号**: 5540-5570 (interceptKeyBeforeQueueing 中处理媒体键)

```java
case KeyEvent.KEYCODE_MEDIA_PLAY:
case KeyEvent.KEYCODE_MEDIA_PAUSE:
case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
case KeyEvent.KEYCODE_HEADSETHOOK:  // ← 这个
case KeyEvent.KEYCODE_MEDIA_STOP:
case KeyEvent.KEYCODE_MEDIA_NEXT:
case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
case KeyEvent.KEYCODE_MEDIA_REWIND:
case KeyEvent.KEYCODE_MEDIA_RECORD:
case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD:
case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK: {
    notifyKeyGestureCompletedOnActionUp(event,
            KeyGestureEvent.KEY_GESTURE_TYPE_MEDIA_KEY);
    if (MediaSessionLegacyHelper.getHelper(mContext).isGlobalPriorityActive()) {
        // If the global session is active pass all media keys to it
        // instead of the active window.
        result &= ~ACTION_PASS_TO_USER;
    }
    if ((result & ACTION_PASS_TO_USER) == 0) {
        // Only do this if we would otherwise not pass it to the user.
        mBroadcastWakeLock.acquire();
        Message msg = mHandler.obtainMessage(MSG_DISPATCH_MEDIA_KEY_WITH_WAKE_LOCK,
                new KeyEvent(event));
        msg.setAsynchronous(true);
        msg.sendToTarget();
    }
    break;
}
```

**行号**: 5966-5990 (dispatchMediaKeyWithWakeLock)

```java
void dispatchMediaKeyWithWakeLock(KeyEvent event) {
    // ... 取消重复按键
    dispatchMediaKeyWithWakeLockToAudioService(event);
    // ... 处理重复按键
}
```

**行号**: 5991-5993 (dispatchMediaKeyWithWakeLockToAudioService)

```java
void dispatchMediaKeyWithWakeLockToAudioService(KeyEvent event) {
    if (event.getAction() == KeyEvent.ACTION_UP) {
        MediaSessionLegacyHelper.getHelper(mContext).sendMediaButtonEvent(event, true);
    }
}
```

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionLegacyHelper.java`  
**行号**: 172-177 (sendMediaButtonEvent)

```java
public void sendMediaButtonEvent(KeyEvent keyEvent, boolean needWakeLock) {
    mCommunicationManager.dispatchMediaKeyEvent(keyEvent, needWakeLock);
}
```

**文件**: `frameworks/base/media/java/android/media/session/MediaSessionManager.java`  
**行号**: 589-590 (dispatchMediaKeyEvent)

```java
public void dispatchMediaKeyEvent(@NonNull KeyEvent keyEvent, boolean needWakeLock) {
    dispatchMediaKeyEventInternal(keyEvent, /*asSystemService=*/false, needWakeLock);
}
```

**行号**: 608-613 (dispatchMediaKeyEventInternal)

```java
private void dispatchMediaKeyEventInternal(KeyEvent keyEvent, boolean asSystemService,
        boolean needWakeLock) {
    try {
        mService.dispatchMediaKeyEvent(mContext.getPackageName(), asSystemService, keyEvent,
                needWakeLock);
    } catch (RemoteException e) {
        Log.e(TAG, "Failed to send media key event.", e);
    }
}
```

#### 阶段 3: MediaSessionService 分发媒体键

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionService.java`  
**行号**: 1774-1834 (dispatchMediaKeyEvent)

```java
public void dispatchMediaKeyEvent(String packageName, boolean asSystemService,
        KeyEvent keyEvent, boolean needWakeLock) {
    if (keyEvent == null || !KeyEvent.isMediaSessionKey(keyEvent.getKeyCode())) {
        Log.w(TAG, "Attempted to dispatch null or non-media key event.");
        return;
    }

    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        if (DEBUG) {
            Log.d(TAG, "dispatchMediaKeyEvent, pkg=" + packageName + " pid=" + pid
                    + ", uid=" + uid + ", asSystem=" + asSystemService + ", event="
                    + keyEvent);
        }
        if (!isUserSetupComplete()) {
            // ===== 关键检查点 =====
            Log.i(TAG, "Not dispatching media key event because user "
                    + "setup is in progress.");
            return;  // ← 如果用户设置未完成，直接返回
        }

        synchronized (mLock) {
            boolean isGlobalPriorityActive = isGlobalPriorityActiveLocked();
            if (isGlobalPriorityActive && uid != Process.SYSTEM_UID) {
                Log.i(TAG, "Only the system can dispatch media key event "
                        + "to the global priority session.");
                return;
            }
            if (!isGlobalPriorityActive) {
                // ===== 关键分支: mOnMediaKeyListener =====
                if (mCurrentFullUserRecord.mOnMediaKeyListener != null) {
                    if (DEBUG_KEY_EVENT) {
                        Log.d(TAG, "Send " + keyEvent + " to the media key listener");
                    }
                    try {
                        mCurrentFullUserRecord.mOnMediaKeyListener.onMediaKey(keyEvent,
                                new MediaKeyListenerResultReceiver(packageName, pid, uid,
                                        asSystemService, keyEvent, needWakeLock));
                        // ← Binder 回调到测试进程
                        return;  // ← 如果 listener 处理了，直接返回
                    } catch (RemoteException e) {
                        Log.w(TAG, "Failed to send " + keyEvent
                                + " to the media key listener");
                    }
                }
            }
            // 如果 listener 没处理或不存在，走后续分发逻辑
            if (isGlobalPriorityActive) {
                dispatchMediaKeyEventLocked(...);
            } else {
                mMediaKeyEventHandler.handleMediaKeyEventLocked(...);
            }
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

### 可能失败原因

**原因 A (最可能): 权限静默失败 (同用例1的原因C)**
- shell 可能没有 SET_MEDIA_KEY_LISTENER 权限
- listener 注册失败但被静默捕获
- mOnMediaKeyListener 实际为 null

**原因 B: isUserSetupComplete() 返回 false**
- MediaSessionService line 1790-1796:
  ```java
  if (!isUserSetupComplete()) {
      Log.i(TAG, "Not dispatching media key event because user setup is in progress.");
      return;
  }
  ```
- 如果设备未完成用户设置（如首次开机向导未完成），所有媒体键分发被跳过

**原因 C: 前台 Activity 消费了 HEADSETHOOK 键**
- PhoneWindowManager line 5553-5569:
  如果 isGlobalPriorityActive 为 false，且 Activity 处理了该键
  (result & ACTION_PASS_TO_USER) != 0
  → 不会走到 dispatchMediaKeyWithWakeLock
  → 事件不会到达 MediaSessionService

**原因 D: Utils.assertMediaPlaybackStarted 未正确设置**
- 测试依赖一个正在运行的媒体播放
- 如果该辅助方法未能正确启动媒体播放
- 可能影响后续的媒体键分发逻辑

**原因 E: isGlobalPriorityActive 为 true**
- 如果有全局优先级的 session 正在运行
- line 1800-1806: 非 SYSTEM_UID 的调用被拒绝
- line 1807: mOnMediaKeyListener 路径被跳过
- 直接走 dispatchMediaKeyEventLocked 路径

---

## 用例 3: testRemoteUserInfo

### 执行路径

这个用例的复杂之处在于它混合了多种分发方式:

1. injectInputEvent(KEYCODE_HEADSETHOOK, false) - 通过 adb 注入
2. session.getController().dispatchMediaButtonEvent(event) - 通过 API 调用
3. controller.dispatchMediaButtonEvent(event) - 通过 API 调用

前者的路径与用例 2 相同。后两者走的是不同的路径:

**文件**: `frameworks/base/media/java/android/media/session/MediaController.java`  
(dispatchMediaButtonEvent)

```java
public boolean dispatchMediaButtonEvent(@NonNull KeyEvent keyEvent) {
    try {
        return mSessionBinder.dispatchMediaButtonEvent(keyEvent);
    } catch (RemoteException e) {
        Log.e(TAG, "Failed to send media button event.", e);
    }
    return false;
}
```

**文件**: `frameworks/base/services/core/java/com/android/server/media/MediaSessionRecord.java`  
(dispatchMediaButtonEvent)

```java
public boolean dispatchMediaButtonEvent(KeyEvent keyEvent) {
    // ... 权限检查
    // 最终调用 session 的 callback
    mCbStub.dispatchOnMediaButtonEvent(keyEvent);
    return true;
}
```

### 可能失败原因

**原因 A (最可能): 媒体键事件根本未到达 MediaSession**
- 与用例2相同的原因：权限、isUserSetupComplete、Activity消费等
- 如果 injectInputEvent 的事件未能到达 session
- 且 dispatchMediaButtonEvent 的 API 调用也未能触发回调
- CountDownLatch 不会 countDown

**原因 B: dispatchMediaButtonEvent 路径问题**
- MediaController.dispatchMediaButtonEvent() 最终调用
  MediaSessionService.dispatchMediaKeyEvent()
- 如果 isGlobalPriorityActive 为 true 且调用者非 SYSTEM_UID
- 事件被拒绝 (line 1800-1806)

**原因 C: Session 未正确激活**
- session.setActive(true) 必须在 session 注册完成后调用
- 如果 session 未被 MediaSessionService 正确注册
- 媒体键不会被分发到该 session

**原因 D: 与 testSetOnMediaKeyListener 的级联影响**
- 如果前一个测试设置了 mOnMediaKeyListener 且未清理
- 该 listener 可能消费了所有媒体键事件
- 导致 session 的 callback 收不到事件
- (但 tearDown 应该会清理)

---

## 日志添加建议

### 用例 1: testSetOnVolumeKeyLongPressListener

#### 位置 1: PhoneWindowManager.java line 3823

```java
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_MUTE:
    // ===== 添加日志 =====
    Log.d(TAG, "Volume key intercepted: keyCode=" + keyCode 
        + ", mUseTvRouting=" + mUseTvRouting 
        + ", mHandleVolumeKeysInWM=" + mHandleVolumeKeysInWM
        + ", action=" + event.getAction()
        + ", repeatCount=" + event.getRepeatCount());
    // ====================
    
    if (mUseTvRouting || mHandleVolumeKeysInWM) {
        // ===== 添加日志 =====
        Log.d(TAG, "Volume key routed to AudioService directly (TV/WM mode)");
        // ====================
        dispatchDirectAudioEvent(event);
        return true;
    }
    // ... 后续代码
```

#### 位置 2: PhoneWindowManager.java line 5401-5411

```java
if (mUseTvRouting || mHandleVolumeKeysInWM) {
    result |= ACTION_PASS_TO_USER;
} else if ((result & ACTION_PASS_TO_USER) == 0) {
    // ===== 添加日志 =====
    Log.d(TAG, "Volume key passed to MediaSessionLegacyHelper");
    // ====================
    MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
            event, AudioManager.USE_DEFAULT_STREAM_TYPE, true);
}
```

#### 位置 3: MediaSessionService.java line 2002-2059

```java
public void setOnVolumeKeyLongPressListener(IOnVolumeKeyLongPressListener listener) {
    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        // ===== 添加日志 =====
        Log.d(TAG, "setOnVolumeKeyLongPressListener called, uid=" + uid 
            + ", pid=" + pid + ", listener=" + listener);
        // ====================
        
        if (mContext.checkPermission(
                android.Manifest.permission.SET_VOLUME_KEY_LONG_PRESS_LISTENER, pid, uid)
                != PackageManager.PERMISSION_GRANTED) {
            // ===== 添加日志 =====
            Log.w(TAG, "setOnVolumeKeyLongPressListener: Permission denied for uid=" + uid);
            // ====================
            throw new SecurityException("Must hold the SET_VOLUME_KEY_LONG_PRESS_LISTENER"
                    + " permission.");
        }

        synchronized (mLock) {
            int userId = UserHandle.getUserHandleForUid(uid).getIdentifier();
            FullUserRecord user = getFullUserRecordLocked(userId);
            // ===== 添加日志 =====
            Log.d(TAG, "setOnVolumeKeyLongPressListener: userId=" + userId 
                + ", user=" + (user != null ? "found" : "null")
                + ", mFullUserId=" + (user != null ? user.mFullUserId : "N/A"));
            // ====================
            
            if (user == null || user.mFullUserId != userId) {
                Log.w(TAG, "Only the full user can set the volume key long-press listener"
                        + ", userId=" + userId);
                return;
            }
            // ... 后续代码
            
            user.mOnVolumeKeyLongPressListener = listener;
            user.mOnVolumeKeyLongPressListenerUid = uid;

            Log.d(TAG, "The volume key long-press listener "
                    + listener + " is set by " + getCallingPackageName(uid));
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

#### 位置 4: MediaSessionService.java line 2941-3022

```java
void handleKeyEventLocked(String packageName, int pid, int uid,
        boolean asSystemService, KeyEvent keyEvent, boolean needWakeLock,
        String opPackageName, int stream, boolean musicOnly) {
    // ===== 添加日志 =====
    Log.d(TAG, "handleKeyEventLocked: keyCode=" + keyEvent.getKeyCode()
        + ", action=" + keyEvent.getAction()
        + ", repeatCount=" + keyEvent.getRepeatCount()
        + ", flags=" + keyEvent.getFlags()
        + ", mOnVolumeKeyLongPressListener=" 
        + (mCurrentFullUserRecord.mOnVolumeKeyLongPressListener != null ? "set" : "null"));
    // ====================
    
    if (keyEvent.isCanceled()) {
        return;
    }

    int overriddenKeyEvents = 0;
    // ... 检查 overriddenKeyEvents
    
    cancelTrackingIfNeeded(...);
    
    if (!needTracking(keyEvent, overriddenKeyEvents)) {
        // ===== 添加日志 =====
        Log.d(TAG, "handleKeyEventLocked: needTracking=false, dispatching directly");
        // ====================
        if (mKeyType == KEY_TYPE_VOLUME) {
            dispatchVolumeKeyEventLocked(...);
        } else {
            dispatchMediaKeyEventLocked(...);
        }
        return;
    }

    if (isFirstDownKeyEvent(keyEvent)) {
        // ===== 添加日志 =====
        Log.d(TAG, "handleKeyEventLocked: First down key event, tracking started");
        // ====================
        mTrackingFirstDownKeyEvent = keyEvent;
        mIsLongPressing = false;
        return;
    }

    if (isFirstLongPressKeyEvent(keyEvent)) {
        // ===== 添加日志 =====
        Log.d(TAG, "handleKeyEventLocked: First long press key event detected");
        // ====================
        mIsLongPressing = true;
    }
    if (mIsLongPressing) {
        // ===== 添加日志 =====
        Log.d(TAG, "handleKeyEventLocked: Long pressing, calling handleLongPressLocked");
        // ====================
        handleLongPressLocked(keyEvent, needWakeLock, overriddenKeyEvents);
        return;
    }
    // ... 后续代码
}
```

#### 位置 5: MediaSessionService.java line 3109-3139

```java
private void handleLongPressLocked(KeyEvent keyEvent, boolean needWakeLock,
        int overriddenKeyEvents) {
    // ===== 添加日志 =====
    Log.d(TAG, "handleLongPressLocked: keyCode=" + keyEvent.getKeyCode()
        + ", mKeyType=" + mKeyType
        + ", isFirstLongPressKeyEvent=" + isFirstLongPressKeyEvent(keyEvent));
    // ====================
    
    if (mCustomMediaKeyDispatcher != null
            && isLongPressOverridden(overriddenKeyEvents)) {
        mCustomMediaKeyDispatcher.onLongPress(keyEvent);
        // ...
    } else {
        if (mKeyType == KEY_TYPE_VOLUME) {
            if (isFirstLongPressKeyEvent(keyEvent)) {
                // ===== 添加日志 =====
                Log.d(TAG, "handleLongPressLocked: Dispatching long press for first key event");
                // ====================
                dispatchVolumeKeyLongPressLocked(mTrackingFirstDownKeyEvent);
            }
            // ===== 添加日志 =====
            Log.d(TAG, "handleLongPressLocked: Dispatching long press for current key event");
            // ====================
            dispatchVolumeKeyLongPressLocked(keyEvent);
        }
        // ...
    }
}
```

#### 位置 6: MediaSessionService.java line 1172-1181

```java
private void dispatchVolumeKeyLongPressLocked(KeyEvent keyEvent) {
    // ===== 添加日志 =====
    Log.d(TAG, "dispatchVolumeKeyLongPressLocked: listener=" 
        + (mCurrentFullUserRecord.mOnVolumeKeyLongPressListener != null ? "set" : "null")
        + ", keyCode=" + keyEvent.getKeyCode()
        + ", action=" + keyEvent.getAction()
        + ", repeatCount=" + keyEvent.getRepeatCount());
    // ====================
    
    if (mCurrentFullUserRecord.mOnVolumeKeyLongPressListener == null) {
        return;
    }
    try {
        mCurrentFullUserRecord.mOnVolumeKeyLongPressListener.onVolumeKeyLongPress(keyEvent);
        // ===== 添加日志 =====
        Log.d(TAG, "dispatchVolumeKeyLongPressLocked: Callback sent successfully");
        // ====================
    } catch (RemoteException e) {
        Log.w(TAG, "Failed to send " + keyEvent + " to volume key long-press listener");
    }
}
```

### 用例 2: testSetOnMediaKeyListener

#### 位置 1: MediaSessionService.java line 2062-2115

```java
public void setOnMediaKeyListener(IOnMediaKeyListener listener) {
    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        // ===== 添加日志 =====
        Log.d(TAG, "setOnMediaKeyListener called, uid=" + uid 
            + ", pid=" + pid + ", listener=" + listener);
        // ====================
        
        if (mContext.checkPermission(
                android.Manifest.permission.SET_MEDIA_KEY_LISTENER, pid, uid)
                != PackageManager.PERMISSION_GRANTED) {
            // ===== 添加日志 =====
            Log.w(TAG, "setOnMediaKeyListener: Permission denied for uid=" + uid);
            // ====================
            throw new SecurityException("Must hold the SET_MEDIA_KEY_LISTENER permission.");
        }

        synchronized (mLock) {
            int userId = UserHandle.getUserHandleForUid(uid).getIdentifier();
            FullUserRecord user = getFullUserRecordLocked(userId);
            // ===== 添加日志 =====
            Log.d(TAG, "setOnMediaKeyListener: userId=" + userId 
                + ", user=" + (user != null ? "found" : "null")
                + ", mFullUserId=" + (user != null ? user.mFullUserId : "N/A"));
            // ====================
            
            if (user == null || user.mFullUserId != userId) {
                Log.w(TAG, "Only the full user can set the media key listener"
                        + ", userId=" + userId);
                return;
            }
            // ... 后续代码
            
            user.mOnMediaKeyListener = listener;
            user.mOnMediaKeyListenerUid = uid;

            Log.d(TAG, "The media key listener " + user.mOnMediaKeyListener
                    + " is set by " + getCallingPackageName(uid));
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

#### 位置 2: MediaSessionService.java line 1774-1834

```java
public void dispatchMediaKeyEvent(String packageName, boolean asSystemService,
        KeyEvent keyEvent, boolean needWakeLock) {
    // ===== 添加日志 =====
    Log.d(TAG, "dispatchMediaKeyEvent: keyCode=" + keyEvent.getKeyCode()
        + ", action=" + keyEvent.getAction()
        + ", packageName=" + packageName
        + ", asSystemService=" + asSystemService
        + ", isUserSetupComplete=" + isUserSetupComplete()
        + ", mOnMediaKeyListener=" 
        + (mCurrentFullUserRecord.mOnMediaKeyListener != null ? "set" : "null")
        + ", isGlobalPriorityActive=" + isGlobalPriorityActiveLocked());
    // ====================
    
    if (keyEvent == null || !KeyEvent.isMediaSessionKey(keyEvent.getKeyCode())) {
        Log.w(TAG, "Attempted to dispatch null or non-media key event.");
        return;
    }

    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    final long token = Binder.clearCallingIdentity();
    try {
        if (!isUserSetupComplete()) {
            // ===== 添加日志 =====
            Log.w(TAG, "dispatchMediaKeyEvent: BLOCKED - user setup not complete");
            // ====================
            Log.i(TAG, "Not dispatching media key event because user "
                    + "setup is in progress.");
            return;
        }

        synchronized (mLock) {
            boolean isGlobalPriorityActive = isGlobalPriorityActiveLocked();
            if (isGlobalPriorityActive && uid != Process.SYSTEM_UID) {
                // ===== 添加日志 =====
                Log.w(TAG, "dispatchMediaKeyEvent: BLOCKED - global priority active, uid=" 
                    + uid);
                // ====================
                Log.i(TAG, "Only the system can dispatch media key event "
                        + "to the global priority session.");
                return;
            }
            if (!isGlobalPriorityActive) {
                if (mCurrentFullUserRecord.mOnMediaKeyListener != null) {
                    // ===== 添加日志 =====
                    Log.d(TAG, "dispatchMediaKeyEvent: Sending to mOnMediaKeyListener");
                    // ====================
                    if (DEBUG_KEY_EVENT) {
                        Log.d(TAG, "Send " + keyEvent + " to the media key listener");
                    }
                    try {
                        mCurrentFullUserRecord.mOnMediaKeyListener.onMediaKey(keyEvent,
                                new MediaKeyListenerResultReceiver(packageName, pid, uid,
                                        asSystemService, keyEvent, needWakeLock));
                        // ===== 添加日志 =====
                        Log.d(TAG, "dispatchMediaKeyEvent: mOnMediaKeyListener callback sent");
                        // ====================
                        return;
                    } catch (RemoteException e) {
                        Log.w(TAG, "Failed to send " + keyEvent
                                + " to the media key listener");
                    }
                } else {
                    // ===== 添加日志 =====
                    Log.d(TAG, "dispatchMediaKeyEvent: mOnMediaKeyListener is null, "
                        + "falling through to normal dispatch");
                    // ====================
                }
            }
            // ... 后续代码
        }
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

#### 位置 3: PhoneWindowManager.java line 5540-5570

```java
case KeyEvent.KEYCODE_HEADSETHOOK:
// ... 其他媒体键
{
    // ===== 添加日志 =====
    Log.d(TAG, "Media key intercepted: keyCode=" + keyCode
        + ", action=" + event.getAction()
        + ", isGlobalPriorityActive=" 
        + MediaSessionLegacyHelper.getHelper(mContext).isGlobalPriorityActive()
        + ", result=" + Integer.toHexString(result));
    // ====================
    
    notifyKeyGestureCompletedOnActionUp(event,
            KeyGestureEvent.KEY_GESTURE_TYPE_MEDIA_KEY);
    if (MediaSessionLegacyHelper.getHelper(mContext).isGlobalPriorityActive()) {
        result &= ~ACTION_PASS_TO_USER;
    }
    if ((result & ACTION_PASS_TO_USER) == 0) {
        // ===== 添加日志 =====
        Log.d(TAG, "Media key will be dispatched to MediaSessionLegacyHelper");
        // ====================
        mBroadcastWakeLock.acquire();
        Message msg = mHandler.obtainMessage(MSG_DISPATCH_MEDIA_KEY_WITH_WAKE_LOCK,
                new KeyEvent(event));
        msg.setAsynchronous(true);
        msg.sendToTarget();
    } else {
        // ===== 添加日志 =====
        Log.d(TAG, "Media key will be passed to foreground activity");
        // ====================
    }
    break;
}
```

### 用例 3: testRemoteUserInfo

#### 位置: MediaSessionRecord.java (dispatchMediaButtonEvent)

```java
public boolean dispatchMediaButtonEvent(KeyEvent keyEvent) {
    // ===== 添加日志 =====
    Log.d(TAG, "dispatchMediaButtonEvent: keyCode=" + keyEvent.getKeyCode()
        + ", action=" + keyEvent.getAction()
        + ", session=" + mSessionId
        + ", caller=" + getPackageName());
    // ====================
    
    // ... 权限检查
    // 最终调用 session 的 callback
    mCbStub.dispatchOnMediaButtonEvent(keyEvent);
    
    // ===== 添加日志 =====
    Log.d(TAG, "dispatchMediaButtonEvent: Callback sent to session");
    // ====================
    return true;
}
```

---

## 综合诊断流程

添加上述日志后，按以下流程诊断:

1. **运行测试前清理日志**:
   ```bash
   adb logcat -c
   ```

2. **运行测试**:
   ```bash
   adb shell am instrument -w -e class android.media.session.cts.MediaSessionManagerTest#testSetOnVolumeKeyLongPressListener com.android.cts.media/androidx.test.runner.AndroidJUnitRunner
   ```

3. **抓取日志**:
   ```bash
   adb logcat -d | grep -E "MediaSessionService|PhoneWindowManager" > /tmp/cts_debug.log
   ```

4. **分析日志**:

**对于 testSetOnVolumeKeyLongPressListener**:
- 如果看到 "Volume key routed to AudioService directly (TV/WM mode)"
  → 设备是 TV 平台，mUseTvRouting=true
  → 音量键不经过 MediaSessionService 的长按检测
  
- 如果看到 "setOnVolumeKeyLongPressListener: Permission denied"
  → shell 没有 SET_VOLUME_KEY_LONG_PRESS_LISTENER 权限
  
- 如果看到 "setOnVolumeKeyLongPressListener: ... listener=null"
  → listener 注册失败
  
- 如果看到 "Volume key intercepted" 但没有 "dispatchVolumeKeyLongPressLocked"
  → 按键到达了 PhoneWindowManager 但未到达 MediaSessionService
  → 检查是否有 "Volume key passed to MediaSessionLegacyHelper"
  
- 如果看到 "handleKeyEventLocked: needTracking=false"
  → mOnVolumeKeyLongPressListener 为 null
  → listener 未成功注册

**对于 testSetOnMediaKeyListener**:
- 如果看到 "setOnMediaKeyListener: Permission denied"
  → shell 没有 SET_MEDIA_KEY_LISTENER 权限
  
- 如果看到 "dispatchMediaKeyEvent: BLOCKED - user setup not complete"
  → user_setup_complete 未设置
  
- 如果看到 "dispatchMediaKeyEvent: BLOCKED - global priority active"
  → 有全局优先级 session 正在运行
  
- 如果看到 "Media key intercepted" 但没有 "Media key will be dispatched to MediaSessionLegacyHelper"
  → 前台 Activity 消费了按键事件
  
- 如果看到 "mOnMediaKeyListener is null, falling through to normal dispatch"
  → listener 未成功注册

**对于 testRemoteUserInfo**:
- 检查 injectInputEvent 的事件是否到达了 MediaSessionService
- 检查 dispatchMediaButtonEvent 是否被调用
- 检查 session 的 callback 是否被触发

5. **根据日志确定根因后，针对性修复**:
- 如果是 TV 平台问题: 修改测试代码，增加对 mUseTvRouting 的检查
- 如果是权限问题: 检查 privapp-permissions-platform.xml 配置
- 如果是 user_setup_complete 问题: 确保设备完成用户设置
- 如果是全局优先级问题: 检查是否有其他应用创建了全局优先级 session

---

## 快速诊断命令

在添加日志之前，可以先用这些命令快速检查:

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

# 检查 config_handleVolumeKeysInWindowManager
adb shell cmd uiautomator dump /dev/null 2>&1
(需要通过代码检查该配置值)
```

---

## 最可能的根因排序

1. **设备为 TV 平台 (mUseTvRouting=true) 但未声明 FEATURE_LEANBACK**
   → 音量键直接走 AudioService，不经过 MediaSessionService 长按检测

2. **shell 权限被厂商定制移除**
   → listener 注册静默失败

3. **前台 Activity 消费了按键事件**
   → 事件未到达 MediaSessionService

4. **user_setup_complete 未设置**
   → 媒体键分发被完全跳过
