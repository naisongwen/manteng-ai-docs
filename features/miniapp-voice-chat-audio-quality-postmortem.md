# 小程序 WebView 语音通话音质问题排查复盘

## 摘要

这次问题的表象是：同一套 voice-chat H5 页面，直接在浏览器访问时 agent 语音清晰稳定；通过微信小程序 `web-view` 打开时，部分 Android 手机会听到滋滋啦啦、断断续续的声音，iOS 小程序里甚至出现完全卡住、无声的情况。

最终确认，问题不在后端接口、agent 生成音频或 RTC 房间本身，而在微信小程序 WebView 环境下的 RTC 音频播放链路。部分手机微信内核对 SDK 默认的 WebAudio in-track 处理路径兼容不好，叠加 WebView/系统层音频处理后，会出现远端 agent 播放失真、卡顿或无声。

真正生效的修复是：在 miniapp 入口下绕开 WebAudio in-track 路径，并恢复 miniapp 专用的 RTC 采集、订阅、播放和优先级配置。

## 背景架构

当前语音通话链路由三部分组成：

1. 小程序页面：`packageChat/h5-chat/h5-chat` 通过 `web-view` 加载线上 H5。
2. H5 页面：`src/views/voice-chat/index.vue` 负责登录态同步、RTC 初始化、进房、麦克风采集、订阅 agent 远端流和播放。
3. 后端接口：`aigc-backend` 提供 voice-chat 会话创建、启动和停止能力，agent 进入同一个 RTC 房间后发布语音流。

由于直接访问 H5 页面音质正常，而小程序内访问异常，问题范围一开始就可以缩小到“小程序 WebView 环境 + H5 RTC SDK 运行方式”的差异上，而不是后端合成音频质量。

## 现象

问题集中出现在远端 agent 声音：

- 直接打开 `voice-chat` H5 页面，agent 音质正常。
- 小程序 `web-view` 打开同一页面，部分 Android 机型出现滋滋啦啦、爆音、断续。
- 某些微信/手机组合播放正常，某些组合异常，说明不是稳定的服务端音频数据损坏。
- iOS 小程序出现过播放完全卡住、无声音。
- 用户本地麦克风采集不是主要问题，主要问题是远端 agent 播放链路。

这个现象非常关键：如果后端推流或 agent 音频源本身坏了，直接访问 H5 也应该复现；如果所有手机都坏，可能是通用前端逻辑错误；但问题只在部分小程序 WebView 里出现，说明兼容性问题概率更高。

## 排查过程

### 1. 先排除后端和 agent 音频源

我们对比了两种入口：

- 浏览器直接访问 H5。
- 小程序 `web-view` 访问同一个 H5。

两者使用同一套后端接口、同一个 RTC SDK、同一个 agent 流。直接 H5 正常，小程序异常，因此后端接口返回、agent 生成音频、RTC 房间创建这些链路不是根因。

### 2. 尝试统一音频档位

最早怀疑是小程序 WebView 中 SDK 默认协商到了较低质量的音频档位，于是尝试显式设置：

```ts
await rtcEngine.setAudioProfile(getRtcConstant('AudioProfileType', 'standard'));
```

这个修改有必要，但不是根因。后续验证发现，“只设置 standard”无法彻底解决小程序内的滋滋啦啦和卡顿问题。

### 3. 尝试调整采集参数

接着我们尝试过不同的采集参数组合，包括采样率、降噪、自动增益、码率等。过程中发现 AGC 容易和 WebView/系统层音频处理叠加，造成音量被反复拉伸，听感上接近破音或断续。

最终保留下来的 miniapp 采集配置是：

```ts
await rtcEngine.setAudioCaptureConfig({
  sampleRate: 48000,
  channelCount: 1,
  echoCancellation: true,
  noiseSuppression: false,
  autoGainControl: false
});
```

这里的重点不是“48kHz 单独解决问题”，而是 miniapp 环境下要明确接管 RTC 音频配置，避免 SDK 默认值、WebView 默认处理和系统音频策略互相叠加。微信 WebView/系统层本身可能已经做了音频处理，因此这里关闭浏览器层 `noiseSuppression` 和 `autoGainControl`，只保留回声消除。

### 4. 关注设备差异

问题不是所有手机都复现。有些 Android 微信上播放正常，有些 Android 有明显杂音；iOS 小程序又出现无声。这说明问题不太可能是应用层某个简单参数错误，而是 WebView 内核、系统音频设备、微信版本和 RTC SDK 音频输出链路之间的兼容性问题。

这个阶段我们把排查重点从“后端音频质量”转向“WebView 播放路径”。

### 5. 找到关键开关：SKIP_WEB_AUDIO_IN_TRACK

最终真正改变结果的是：

```ts
sdk.setParameter('SKIP_WEB_AUDIO_IN_TRACK', true);
```

这个参数的作用是让 RTC SDK 跳过 WebAudio in-track 处理路径。WebAudio 本身是浏览器提供的音频处理 API，可以用于混音、增益、滤波、分析和调度播放。但在微信小程序 WebView 中，WebAudio 相关链路在不同内核和设备上的行为并不完全一致。

当远端 agent 音频走到 WebAudio in-track 路径后，部分设备会出现播放调度不稳、额外处理叠加、音频块衔接异常等问题，表现为滋滋啦啦、断断续续或无声。跳过这条路径后，远端播放稳定性恢复。

### 6. 清理代码导致回归

问题修好后，我们曾经尝试清理之前的试探性代码。但清理时把真正起作用的 miniapp 专用逻辑也删掉了，包括：

- miniapp 下的音频采集配置。
- 远端播放音量设置。
- 手动订阅和播放远端 agent 音频。
- 远端用户优先级。
- `SKIP_WEB_AUDIO_IN_TRACK`。

清理后 Android 音质问题重新出现。把这些逻辑完整恢复后，iOS 和 Android 又恢复正常。这次回归反向证明：问题根因确实在 miniapp WebView 的 RTC 音频播放链路，而不是后端音频源。

## 最终修复

最终修复集中在 `src/views/voice-chat/index.vue`。

### 1. miniapp 下跳过 WebAudio in-track

```ts
if (isMiniappSource.value) {
  sdk.setParameter('SKIP_WEB_AUDIO_IN_TRACK', true);
}
await sdk.enableDevices({ audio: true, video: false });
```

这是本次修复的关键。它让小程序 WebView 避开兼容性不稳定的 WebAudio in-track 路径。这个参数需要在 `enableDevices` 等可能初始化音频链路的调用之前设置，避免部分进入场景已经走到默认 WebAudio 路径。

### 2. miniapp 下恢复独立 RTC 音频调优

```ts
async function applyMiniappRtcAudioTuning(rtcEngine: any) {
  if (!isMiniappSource.value || !rtcEngine) {
    return;
  }

  const standardProfile = getRtcAudioProfile('standard');
  if (standardProfile !== undefined) {
    await rtcEngine.setAudioProfile(standardProfile);
  }

  await rtcEngine.setAudioCaptureConfig({
    sampleRate: 48000,
    channelCount: 1,
    echoCancellation: true,
    noiseSuppression: false,
    autoGainControl: false
  });
}
```

这段逻辑不能被当成无用的试验代码删除。它和 `SKIP_WEB_AUDIO_IN_TRACK` 一起构成 miniapp WebView 的稳定音频链路。

### 3. 设置 miniapp 采集和播放音量

```ts
const miniappLocalCaptureVolume = 75;
const miniappRemotePlaybackVolume = 65;
```

麦克风开启后设置本地采集音量：

```ts
if (isMiniappSource.value) {
  engine.value.setCaptureVolume(
    getRtcConstant('StreamIndex', MAIN_STREAM_INDEX),
    miniappLocalCaptureVolume
  );
}
```

订阅远端 agent 后设置播放音量：

```ts
if (isMiniappSource.value) {
  engine.value.setPlaybackVolume(
    event.userId,
    getRtcConstant('StreamIndex', MAIN_STREAM_INDEX),
    miniappRemotePlaybackVolume
  );
}
```

音量不是根因，但它避免 WebView/系统层过度放大导致破音，也让不同设备上的听感更一致。

### 4. 手动订阅远端音频并设置优先级

```ts
if (isMiniappSource.value && typeof engine.value.setRemoteUserPriority === 'function') {
  engine.value.setRemoteUserPriority(
    event.userId,
    getRtcConstant('RemoteUserPriority', 'HIGH')
  );
}

await engine.value.subscribeStream(event.userId, audioMediaType);
```

小程序 WebView 中不能完全依赖自动订阅和默认播放行为。显式订阅远端 agent 音频、设置高优先级，可以降低弱网或设备调度差异带来的播放抖动。

### 5. 保证 publish/unpublish 的异步顺序

```ts
await engine.value.publishStream(getRtcConstant('MediaType', 'AUDIO'));
await engine.value.unpublishStream(getRtcConstant('MediaType', 'AUDIO'));
```

这不是音质根因，但能减少进出房、开关麦时的状态竞争，避免下一次会话继承异常状态。

## 根因结论

真正的问题原因是：

> 微信小程序 WebView 环境下，RTC SDK 默认的 WebAudio in-track 音频处理路径在部分 iOS/Android 微信内核上兼容性不好，导致远端 agent 音频播放出现失真、断续或卡死。清理代码时删除 miniapp 专用音频链路后，问题重新出现；完整恢复后问题消失。

这也解释了几个关键现象：

- 直接访问 H5 正常：普通浏览器的音频链路没有触发同样的 WebView 兼容问题。
- 只有部分手机异常：微信内核、系统音频栈和设备硬件差异影响 WebAudio 播放稳定性。
- 后端无须改动：同一个 agent 流在直接 H5 中正常播放。
- `standard` 单独无效：音频档位只能改善编码质量，无法绕开 WebAudio 兼容问题。
- 清理后复现：被删掉的不是无效试验代码，而是 miniapp 环境的必要兼容层。

## 复发补充：WebView 生命周期残留

后续出现过“修好后过一段时间重新打开小程序又有问题”的情况。这次复发和前面的 WebAudio 音频链路问题不同，关键诱因是小程序 `web-view` 的生命周期：

- 用户从小程序离开或小程序进入后台时，H5 页面不一定立刻触发 Vue 组件卸载。
- 原代码主要依赖 `onBeforeUnmount` 清理 RTC 会话，但微信 WebView 可能只是挂起或缓存页面。
- 如果旧的 RTC engine、麦克风采集、远端 agent 会话没有及时释放，下一次重新打开就可能继承异常状态，表现为无声、播放失败、远端流状态错乱或再次出现不稳定。
- 清理流程如果先等待后端 `stopVoiceChat` 网络请求，再执行本地 `leaveRoom` 和 `destroyEngine`，在 WebView 很快被冻结的场景下，本地 RTC 释放也可能来不及执行。

补充修复集中在 `src/views/voice-chat/index.vue`：

- 增加 `pagehide`、`beforeunload`、`visibilitychange(hidden)` 监听，在 miniapp 入口下主动触发 RTC 清理。
- 将 `cleanupRtcSession` 改为幂等，避免重复退房、重复销毁 engine 互相打架。
- 清理时先启动本地 `unpublishStream`、`stopAudioCapture`、`leaveRoom`、`destroyEngine`，再调用后端 `stopVoiceChat`，优先保证 WebView 挂起前本地 RTC 资源释放。
- 新建连接前如果仍有清理任务进行中，先等待清理完成，避免新旧 engine 交叉。若页面在 `joinRoom` 过程中进入隐藏态，先挂起清理；如果很快恢复可见则取消清理并恢复播放，如果连接结束后仍处于隐藏态再执行清理，否则会把未完成的连接直接打断，表现为“建立语音连接失败”。

## 复发补充：自动进房缺少 H5 用户手势

再次复现的现象是：多次进入小程序，有时声音正常，有时不正常。这类随机性和微信 WebView 的自动播放策略高度相关。

小程序原生页面点击后进入 `web-view`，并不等同于 H5 页面内部发生了一次可用于解锁音频播放的用户手势。原逻辑在 `mode=phone` 时会在 H5 `onMounted` 后自动 `enterPhoneMode()`，然后自动进 RTC 房间并播放远端 agent 音频。这个流程在部分微信 WebView 中可能被判定为非用户手势触发，导致远端音频播放有时成功、有时失败。

补充修复：

- 不改变电话页 UI 和原有进入方式，miniapp `mode=phone` 仍按原流程进入电话页并自动连接。
- 按 RTC SDK 要求，在远端音频 `play()` 前也先通过 `setRemoteVideoPlayer` 绑定隐藏播放器，避免 agent 只发布音频时缺少播放器绑定导致播放随机失败。
- 远端音频播放封装为可重试流程，记录已发布远端音频用户，在 `onAutoplayFailed`、远端流发布后、播放器 `pause`、页面重新可见和用户后续已有操作时尝试恢复播放。
- 当 miniapp 首次远端音频稳定播放后，自动重启一次麦克风采集，模拟用户“关麦再开麦”后音频恢复的效果，把被 WebView 打乱的本地音频管线重新拉直。这个重启不能和 `joinRoom` 并发执行，必须在连接稳定后延迟触发；如果还在 `joining`，要顺延而不是立即重启，否则会出现“建立语音连接失败”的回归。
- 这次修复保留原有 `SKIP_WEB_AUDIO_IN_TRACK`、miniapp 音频参数、手动订阅、播放音量和远端优先级逻辑，不替代前面的音质兼容修复。

## 经验教训

1. WebView 问题不要只用桌面浏览器验证。
   语音、视频、摄像头、播放设备这类能力必须覆盖微信小程序 iOS、Android 和不同微信版本。

2. 试探性代码在清理前要先分类。
   对音频链路来说，“看起来像 workaround”的代码可能正是兼容性修复。删除前必须有设备回归结果。

3. 对 miniapp WebView 保留独立音频策略。
   直接 H5 和小程序 WebView 不能完全视为同一个运行环境。小程序需要独立的音频参数、播放路径和订阅策略。

4. 根因判断要结合反向验证。
   本次清理后问题复现、恢复后问题消失，是确认根因的重要证据。

## 后续建议

1. 增加语音诊断日志。
   记录入口来源、设备平台、微信版本、是否启用 `SKIP_WEB_AUDIO_IN_TRACK`、音频 profile、采集配置、订阅状态和播放失败事件。

2. 建立设备回归矩阵。
   至少覆盖 iPhone + 微信、主流 Android + 微信、直接 H5 浏览器三类入口。

3. 给 miniapp 音频兼容代码加注释。
   说明这些逻辑是已验证的兼容修复，避免后续优化时再次被误删。

4. 保持后端和前端问题边界清晰。
   当直接 H5 正常、小程序异常时，优先检查 WebView、SDK 参数、浏览器能力和设备策略。

## 最终状态

当前修复已经恢复并验证正常：

- iOS 小程序播放恢复。
- Android 小程序播放恢复。
- 直接 H5 页面继续保持正常。
- 后端 voice-chat 接口无需修改。

这次问题的本质不是“音频质量参数调大或调小”，而是“小程序 WebView 中要避开不稳定的 WebAudio 处理路径，并显式接管 RTC 远端播放链路”。
