---
title: iOS 音频（一）
date: 2021-10-13 12:00:00
tags:
- iOS
- 音频
categories: iOS开发
---

# iOS 音频（一）

> 本篇是在做完公司的合拍需求后，对于音频处理的一些总结，如有错误的地方请指正

## 音频基础知识

目前常见的音频格可以大致分为两种：

1. 有损压缩，例如：`MP3`、`AAC`、`OGG`、`WMA`
2. 无损压缩，例如：`ALAC`、`APE`、`FLAC`、`WAV`

其中，对声音进行采样、量化过程被称为`脉冲编码调制`（Pulse Code Modulation），简称`PCM`。PCM数据是最原始的音频数据完全无损，但占用空间大，(`PCM`电脑上可以用`FFPlay` 设置采样率后播放)

<!--more-->  
**关于音频计算**

- 采样率：指**每秒钟取得声音样本的次数**。采样频率越高，声音的质量也就越好，声音的还原也就越真实，但同时它占的资源比较多。由于人耳的分辨率很有限，太高的频率并不能分辨出来。
- 采样位数：即**采样值**或取样值（声音的连续强度被数字表示后可以分为多少级，即将采样样本幅度量化）。它是用来衡量声音波动变化的一个参数，也可以说是声卡的分辨率。它的数值越大，分辨率也就越高，所发出声音的能力越强。
  - 8 bit -> 记录 256（2^8）个数，振幅划分成 256 个等级 
  - 16 bit -> 记录65536(2^16) 个数，振幅划分成 65536 个等级 
- 通道数：**声音的通道的数目**。常有单声道和立体声之分，单声道的声音只能使用一个喇叭发声（有的也处理成两个喇叭输出同一个声道的声音），立体声可以使两个喇叭都发声（一般左右声道有分工） ，更能感受到空间效果，当然还有更多的通道数。
  - 单声道 mono
  - 双声道 stereo （左右）
  - 2.1 
  - 5.1
  - 7.1
- 每秒数据大小：采样率 * 采样通道 * 位深度 / 8

以项目中为例：

> 采样率 = 44100，采样通道 = 1，位深度 =16，采样间隔 = 20ms
>
> 一秒钟50帧 ，每帧大小为： 88200 / 50 = 1764 byte == 882 short

## AVAudioSession

> 做项目的时候，合拍需求需要同时播放音视频，同时需要录制用户的声音。在做的过程中发现，不去设置AVAudioSession，则使用蓝牙耳机时，视频播放的声音仍然会通过扬声器播放，以及一些意想不到的问题。



`AVAudioSession` 会自带一些默认的设置，以便在没有设置的情况下，能够应对一些简单的场景，先来考虑两个常见的场景：

1. 语音通话时，手机的其他app正在播放音乐
2. 地图导航，当需要语音播报，手机的音乐会减少

### 作用



![图片来自官方](https://imagedatabase-1259343097.cos.ap-beijing.myqcloud.com/blogImage/ASPG_intro_2x.png)

简单来说，`AVAudioSession`的作用分为三点：

1. 协调多个App的音频播放
2. 告诉系统如何使用音频：录音、播放、边录边播
3. 设置输入输出设备



APP启动的时候会自动帮激活`AVAudioSession`，APP维护一个`AVAudioSession` 的单例

```swift
import AVFoundation

AVAudioSession.sharedInstance()
```

### AVAudioSession Category

目前`AVAudioSession` 所支持的`category` 共有 7 种：

| Category                                | 是否会被静音键或锁屏键静音 | 是否打断不支持混音播放的应用 | 是否允许音频输入/输出 | 解释                                                |
| --------------------------------------- | -------------------------- | ---------------------------- | --------------------- | --------------------------------------------------- |
| AVAudioSessionCategoryAmbient           | Yes                        | NO                           | 只输出                | 只播放音频，不会被打断                              |
| AVAudioSessionCategoryAudioProcessing   | NO                         | YES                          | 无输入和输出          | 音频处理                                            |
| AVAudioSessionCategoryMultiRoute        | NO                         | YES                          | 支持输入和输出        | 支持音频播放和录制（允许多条音频流的同步输入和输出) |
| AVAudioSessionCategoryPlayAndRecord     | NO                         | 默认 YES，可重写开关置为 NO  | 支持输入和输出        | 支持音频播放和录制。                                |
| AVAudioSessionCategoryPlayback          | NO                         | 默认 YES，可重写开关置为 NO  | 只输出                | 只播放音频，一般音乐播放器都会选择这个              |
| AVAudioSessionCategoryRecord            | NO（锁屏时依然保持录制）   | YES                          | 只输入                | 只支持音频录制                                      |
| AVAudioSessionCategorySoloAmbient(默认) | YES                        | YES                          | 只输出                | 只播放音频，会被打断                                |

**第一个坑：**

如果`AVAudioSession` 处于`inactive` 的状态，那么`setCategory` 会在激活时才发送，不会立即生效，若处于`active`状态，则立即生效。

```swift
try? AVAudioSession.sharedInstance().setCategory(.playAndRecord)
try? AVAudioSession.sharedInstance().setActive(true)
/// 建议在init方法里面就激活一下
```

若当前App激活了`AVAudioSession`，则其他App的`AVAudioSession` 会失效，若想让其他App重新也能被激活：

```swift
AVAudioSession.sharedInstance().setActive(true, options: .notifyOthersOnDeactivation)
```



**小结：**

到此为止。很清楚的可以看到，常见的：

若需求是单纯的音频播放，例如播放器等选择：AVAudioSessionCategoryPlayback

若需求是需要录制，例如录音机，录制视频等选择：AVAudioSessionCategoryRecord

若需求是需要录制同时播放声音，例如短视频合拍等选择：AVAudioSessionCategoryPlayAndRecord



### AVAudioSession Option

| Option                                       | 说明                 | 兼容的 Category                                              | 解释                                                         |
| -------------------------------------------- | -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| AVAudioSessionCategoryOptionMixWithOthers    | 允许和其他音频 mix   | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryPlayback AVAudioSessionCategoryMultiRoute | 例如：当前App播放的声音想与QQ音乐播放的声音并存              |
| AVAudioSessionCategoryOptionDuckOthers       | 智能调低冲突音频音量 | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryPlayback AVAudioSessionCategoryMultiRoute | 例如：导航时，语音播报并不会打断QQ音乐的声音，只是让其他App声音变小 |
| AVAudioSessionCategoryOptionAllowBluetooth   | 允许蓝牙音频输入     | AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayAndRecord | 若要支持蓝牙耳机，这个是必备的                               |
| AVAudioSessionCategoryOptionDefaultToSpeaker | 默认输出音频到扬声器 | AVAudioSessionCategoryPlayAndRecord                          | 将音频输出到扬声器                                           |

除此之外，在iOS9还提供了`AVAudioSessionCategoryOptionInterruptSpokenAudioAndMixWithOthers`最新的iOS10又新加了两个`AVAudioSessionCategoryOptionAllowBluetoothA2DP`    、`AVAudioSessionCategoryOptionAllowAirPlay`用来支持蓝牙A2DP耳机和AirPlay。

```swift
/// 通过这个方法设置option
func setCategory(_ category: AVAudioSession.Category, options: AVAudioSession.CategoryOptions = []) throws
```

第二个坑：

一般来说，所有的`Category`和`Option`都会遵循 `last in wins` 原则，即最后接入的音频设备作为输入或输出的主设备。同时用于播放音频的App都需要考虑到用户使用蓝牙耳机的情况，若不设置`AVAudioSessionCategoryOptionAllowBluetooth`,则可能出现虽然链接蓝牙耳机，但在`一边录制一边播放`的情况下，音频还是会从扬声器种播放出来。





### AVAudioSession Mode

以上的`Category`和`Option`基本上能够满足大部分的App使用，对于特定的情况，例如通话、游戏，`AVAudioSession`还有自己特殊的优化 `Mode`：

| Mode                             | 兼容的 Category                                              | 说明                                                     |
| -------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| AVAudioSessionModeDefault        | All                                                          | 默认格式                                                 |
| AVAudioSessionModeVoiceChat      | AVAudioSessionCategoryPlayAndRecord                          | VoIP 类型的应用                                          |
| AVAudioSessionModeGameChat       | AVAudioSessionCategoryPlayAndRecord                          | 适用于游戏类应用                                         |
| AVAudioSessionModeVideoRecording | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord | 适用于使用摄像头采集视频的应用，搭配AVCaptureSession使用 |
| AVAudioSessionModeMoviePlayback  | AVAudioSessionCategoryPlayback                               | 适用于播放视频的应用                                     |
| AVAudioSessionModeMeasurement    | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayback | 最小化系统（？？？ 不是很清楚）                          |
| AVAudioSessionModeVideoChat      | AVAudioSessionCategoryPlayAndRecord                          | 视频聊天类型应用                                         |

## 状态监听

### 耳机

这部分直接上代码吧：

```swift
/// 是否有耳机
    private func isUseHeadphones() -> Bool {
        let isUse = AVAudioSession.sharedInstance().currentRoute.outputs.first {
            $0.portType.rawValue == AVAudioSession.Port.headphones.rawValue ||
            $0.portType.rawValue == AVAudioSession.Port.bluetoothA2DP.rawValue ||
            $0.portType.rawValue == AVAudioSession.Port.bluetoothHFP.rawValue ||
            $0.portType.rawValue == AVAudioSession.Port.bluetoothLE.rawValue ||
            $0.portType.rawValue == AVAudioSession.Port.airPlay.rawValue
        } != nil
        return isUse
    }
```



判断耳机种类：

```swift
func updateHeadPhonesStatus() {
        guard let des = AVAudioSession.sharedInstance().currentRoute.outputs.first else {return}
        if des.portType.rawValue == AVAudioSession.Port.headphones.rawValue {
            // 有线耳机
            return
        } else if (des.portType.rawValue == AVAudioSession.Port.bluetoothA2DP.rawValue ||
                    des.portType.rawValue == AVAudioSession.Port.bluetoothHFP.rawValue ||
                    des.portType.rawValue == AVAudioSession.Port.bluetoothLE.rawValue ||
                    des.portType.rawValue == AVAudioSession.Port.airPlay.rawValue) {
            //  蓝牙耳机
            return
        }
        //外放扬声器
        context?.controller?.handleChangeDeviceStatus(to: .loudSpeaker)
    }
```



### 外设状态



首先需要注册一个`AVAudioSession.routeChangeNotification`:

```swift
observers.append(NotificationCenter.default.addObserver(forName: AVAudioSession.routeChangeNotification, object: nil, queue: .main) { [weak self] notification in
            self?.handleSessionRouteChange(notification)
        })
        UIApplication.shared.beginReceivingRemoteControlEvents()
```

目前状态是一个枚举，主要分为8种：

| 枚举值                                                    | 意义                         |
| --------------------------------------------------------- | ---------------------------- |
| AVAudioSessionRouteChangeReasonUnknown                    | 未知原因                     |
| AVAudioSessionRouteChangeReasonNewDeviceAvailable         | 有新设备可用                 |
| AVAudioSessionRouteChangeReasonOldDeviceUnavailable       | 老设备不可用                 |
| AVAudioSessionRouteChangeReasonCategoryChange             | 类别改变了                   |
| AVAudioSessionRouteChangeReasonOverride                   | App重置了输出设置            |
| AVAudioSessionRouteChangeReasonWakeFromSleep              | 从睡眠状态呼醒               |
| AVAudioSessionRouteChangeReasonNoSuitableRouteForCategory | 当前Category下没有合适的设备 |
| AVAudioSessionRouteChangeReasonRouteConfigurationChange   | Rotuer的配置改变了           |



可以通过判断返回的枚举值，去做相应的处理，例如拔出耳机时，停止播放，更新耳机状体等等。


## 总结

以上就是对于一些音频知识的总结，最开始的两个场景解决办法应该自然而然出来了，本篇主要是针对`AVAudioSession`做的分析，若有问题，请还望多包涵和指正。下一小节将会去介绍音频处理相关的内容。

