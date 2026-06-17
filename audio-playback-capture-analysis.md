# CTS AudioPlaybackCaptureTest#testCaptureExcludeUid 失败分析

## 问题现象

CTS 模块：

```text
arm64-v8a CtsMediaAudioTestCases
android.media.audio.cts.AudioPlaybackCaptureTest#testCaptureExcludeUid
```

失败栈：

```text
java.lang.UnsupportedOperationException: Error: could not register audio policy
    at android.media.AudioRecord$Builder.buildAudioPlaybackCaptureRecord(AudioRecord.java:812)
    at android.media.AudioRecord$Builder.build(AudioRecord.java:986)
    at android.media.audio.cts.AudioPlaybackCaptureTest.createPlaybackCaptureRecord(...)
    at android.media.audio.cts.AudioPlaybackCaptureTest.testCaptureExcludeUid(...)
```

这个异常说明 `AudioRecord` 还没有创建成功，失败发生在 playback capture 动态 `AudioPolicy` 注册阶段，而不是录音开始后读取数据失败。

## CTS 用例在测什么

`AudioPlaybackCaptureTest#testCaptureExcludeUid` 会构造两类排除 UID 的场景：

```java
mAPCTestConfig.excludeUids = new int[]{ 0 };
testPlaybackCapture(OPT_IN, AudioAttributes.USAGE_GAME, EXPECT_DATA);

mAPCTestConfig.excludeUids = new int[]{ mUid };
testPlaybackCapture(OPT_IN, AudioAttributes.USAGE_GAME, EXPECT_SILENCE);
```

第一段是本次失败最相关的场景。

`excludeUid(0)` 的语义不是非法参数，而是排除 UID 0/root 的播放流。CTS 自己播放音频的 UID 是测试应用 UID，不是 0，所以即使排除了 UID 0，也仍然应该捕获到测试音频数据。

因此，`excludeUid(0)` 这一轮的预期是 `EXPECT_DATA`。

## 从 CTS 到 framework 的调用链

CTS 通过 `APCTestConfig.build()` 创建 `AudioPlaybackCaptureConfiguration`，其中会调用：

```java
AudioPlaybackCaptureConfiguration.Builder builder =
        new AudioPlaybackCaptureConfiguration.Builder(mediaProjection);

builder.excludeUid(uid);
```

然后 CTS 创建 playback capture 用的 `AudioRecord`：

```java
new AudioRecord.Builder()
        .setAudioPlaybackCaptureConfig(config)
        .setAudioFormat(format)
        .build();
```

进入 framework 后，关键调用在 `AudioRecord.Builder.buildAudioPlaybackCaptureRecord()`：

```java
AudioMix audioMix = config.createAudioMix(format);
AudioPolicy audioPolicy = new AudioPolicy.Builder()
        .setMediaProjection(config.getMediaProjection())
        .addMix(audioMix)
        .build();

int error = AudioManager.registerAudioPolicyStatic(audioPolicy);
if (error != 0) {
    throw new UnsupportedOperationException("Error: could not register audio policy");
}
```

所以 CTS 栈里的异常直接对应：

```text
AudioManager.registerAudioPolicyStatic(audioPolicy) != SUCCESS
```

## AudioPlaybackCaptureConfiguration 做了什么

`AudioPlaybackCaptureConfiguration.createAudioMix()` 会把 capture 规则转换成一个 `AudioMix`。

playback capture 使用的 mix route 通常是：

```java
ROUTE_FLAG_LOOP_BACK | ROUTE_FLAG_RENDER
```

含义是：

- `RENDER`：音频仍正常播放到输出设备。
- `LOOP_BACK`：同时复制一份给 playback capture 的录音端。

`excludeUid(0)` 会进入 `AudioMixingRule`，规则类型是：

```java
RULE_EXCLUDE_UID
```

AOSP 预期这里允许 UID 0 作为过滤条件存在。

## AudioManager 到 AudioService

`AudioManager.registerAudioPolicyStatic()` 会调用 system_server 中的：

```java
AudioService.registerAudioPolicy(...)
```

如果 `AudioService` 返回的 registration id 是 `null`，`AudioManager` 会返回 `ERROR`，然后 `AudioRecord` 抛出：

```text
UnsupportedOperationException: Error: could not register audio policy
```

也就是说，真正失败点有两大类：

1. Java 层 `AudioService` 拒绝注册 policy。
2. Java 层允许了，但 native audio policy mix 注册失败。

## 第一类失败：AudioService 权限或 MediaProjection 拒绝

`AudioService.registerAudioPolicy()` 会先检查调用方是否允许注册这个 policy。

对 playback capture 这种 `LOOP_BACK | RENDER` 的 mix，正常路径需要一个有效的 `MediaProjection`，并且该 projection 必须允许音频投射：

```java
projectionService.isValidMediaProjection(projection)
projection.canProjectAudio()
```

如果这里失败，`AudioService` 会直接返回 `null`，不会进入 native `registerPolicyMixes()`。

这种情况下问题通常是：

- CTS MediaProjection 授权流程异常；
- projection token 失效；
- `canProjectAudio()` 返回 false；
- 设备侧改坏了 `isPolicyRegisterAllowed()` 的权限判断。

## 第二类失败：native registerPolicyMixes 失败

如果 Java 权限检查通过，`AudioService` 会创建 `AudioPolicyProxy`。构造过程中会调用：

```java
int status = connectMixes();
if (status != AudioSystem.SUCCESS) {
    throw new IllegalStateException("Could not connect mix, error: " + status);
}
```

`connectMixes()` 最终会进入：

```java
AudioSystem.registerPolicyMixes(mMixes, true)
```

native 侧对应到 `AudioPolicyManager::registerPolicyMixes()`。

对 playback capture 来说，native 侧会检查：

- mix type 是否是 `MIX_TYPE_PLAYERS`；
- route flag 是否匹配 loopback/render；
- 是否能找到 `remote_submix` audio module；
- policy mix 是否能注册到 `mPolicyMixes`；
- remote submix device connection 是否能建立。

如果 `remote_submix` 配置缺失，经常会看到类似日志：

```text
Unable to find audio module for submix
```

如果这里失败，Java 最终也只会表现为：

```text
Error: could not register audio policy
```

## 为什么重点怀疑 excludeUid(0)

这个 CTS case 的特殊点是第一轮使用：

```java
excludeUid(0)
```

在 AOSP 语义里，UID 0 是合法的排除条件，不应该导致 policy 注册失败。

因此，如果现象是：

- 只有 `testCaptureExcludeUid` 失败；
- 其他 playback capture case 可以通过；
- 或者只有 `excludeUid(0)` 这一轮失败；

那么最值得怀疑的是设备侧/vendor 改动把 `uid == 0` 错误当成非法参数处理了，例如：

```cpp
if (uid <= 0) {
    return BAD_VALUE;
}
```

或者把 0 当成 clear/invalid/reserved value，导致 `RULE_EXCLUDE_UID` 中的 UID 0 无法通过 Java 或 native 参数校验。

正确行为应该是：

- policy 注册成功；
- capture filter 中包含 `RULE_EXCLUDE_UID: 0`；
- 测试 app 自己的播放流不被排除；
- `EXPECT_DATA` 成功。

## 推荐日志

复现时建议抓：

```bash
adb logcat -b all -v threadtime | grep -iE \
"registerAudioPolicy|policy denied|connectMixes|registerPolicyMixes|remote_submix|submix|RULE_EXCLUDE_UID|exclude uid|uid 0|MediaProjection"
```

判读方式：

- 看到 `policy denied` 或 permission 相关日志：问题还在 Java `AudioService` 权限/MediaProjection 判断。
- 看到 `connectMixes failed` 或 `Could not connect mix`：Java 已通过，失败在 native `registerPolicyMixes()`。
- 看到 `Unable to find audio module for submix`：`remote_submix` 配置缺失或 audio policy configuration 不完整。
- 只在 `excludeUid(0)` 时失败：重点查 `RULE_EXCLUDE_UID`、`uid <= 0`、`uid == 0` 的设备侧非法判断。

## 结论

本 CTS 失败不是数据断言问题，而是 playback capture 的动态 audio policy 注册失败。

从 AOSP 调用链看，`UnsupportedOperationException: Error: could not register audio policy` 只是最外层包装，真正原因需要在两处区分：

1. `AudioService.registerAudioPolicy()` 是否因为 MediaProjection/权限返回 `null`。
2. `AudioPolicyProxy.connectMixes()` 是否因为 native `registerPolicyMixes()` 返回错误。

结合 `testCaptureExcludeUid` 的测试内容，若其他 playback capture case 正常，仅该 case 失败，则最可疑的设备侧问题是把 `excludeUid(0)` 中的 UID 0 误判为非法，导致 audio policy mix 注册提前失败。
