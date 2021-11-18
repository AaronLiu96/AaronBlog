---
title: iOS 音频（一）
date: 2021-10-13 12:00:00
tags:
- iOS
- 音频
categories: iOS开发
---

## AVAudioSession

> 做项目的时候，合拍需求需要同时播放音视频，同时需要录制用户的声音。在做的过程中发现，不去设置AVAudioSession，则使用蓝牙耳机时，视频播放的声音仍然会通过扬声器播放，以及一些意想不到的问题。


`AVAudioSession` 会自带一些默认的设置，以便在没有设置的情况下，能够应对一些简单的场景

### 作用

![图片来自官方](https://imagedatabase-1259343097.cos.ap-beijing.myqcloud.com/blogImage/ASPG_intro_2x.png)

简单来说，`AVAudioSession`的作用分为三点：

1. 协调多个App的音频播放
2. 告诉系统如何使用音频：录音、播放、边录边播
3. 设置输入输出设备

<!--more-->  

APP启动的时候会自动帮激活`AVAudioSession`，APP维护一个`AVAudioSession` 的单例

```swift
import AVFoundation

AVAudioSession.sharedInstance()
```

### AVAudioSession Category

目前`AVAudioSession` 所支持的`category` 共有 7 种：

| Category                                | 是否会被静音<br>键或锁屏键静音 | 是否打断不支持混音<br>播放的应用 | 是否允许音频<br>输入/输出 | <div style="width: 150pt">解释</div> |
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

| Option                                       | 说明                 | 兼容的 Category                                              | <div style="width: 150pt">解释</div> |
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

一般来说，所有的`Category`和`Option`都会遵循 `last in wins` 原则，即最后接入的音频设备作为输入或输出的主设备。同时用于播放音频的App都需要考虑到用户使用蓝牙耳机的情况，若不设置`AVAudioSessionCategoryOptionAllowBluetooth`,则可能出现虽然链接蓝牙耳机，但在`一边录制一边播放`的情况下，音频还是会从扬声器中播放出来。


### AVAudioSession Mode

以上的`Category`和`Option`基本上能够满足大部分的App使用，对于特定的情况，例如通话、游戏，`AVAudioSession`还有自己特殊的优化 `Mode`：

| Mode                             | 兼容的 Category                                              | <div style="width: 150pt">说明</div> |
| -------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| AVAudioSessionModeDefault        | All                                                          | 默认格式                                                 |
| AVAudioSessionModeVoiceChat      | AVAudioSessionCategoryPlayAndRecord                          | VoIP 类型的应用                                          |
| AVAudioSessionModeGameChat       | AVAudioSessionCategoryPlayAndRecord                          | 适用于游戏类应用                                         |
| AVAudioSessionModeVideoRecording | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord | 适用于使用摄像头采集视频的应用，搭配AVCaptureSession使用 |
| AVAudioSessionModeMoviePlayback  | AVAudioSessionCategoryPlayback                               | 适用于播放视频的应用                                     |
| AVAudioSessionModeMeasurement    | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayback | 最小化系统（？？？ 不是很清楚）                          |
| AVAudioSessionModeVideoChat      | AVAudioSessionCategoryPlayAndRecord                          | 视频聊天类型应用                                         |
**几个需要注意的地方**
- `AVAudioSessionModeVoiceChat` 适用于VoIP（基于IP的语音传输英语：Voice over Internet Protocol，缩写为VoIP）这个模式下，会自动配置上`AVAudioSessionCategoryOptionAllowBluetooth` 和 `AVAudioSessionCategoryOptionDefaultToSpeaker`。系统将自动选择最佳的麦克风组合来支持视频聊天
- `AVAudioSessionModeVideoRecording` 适用于使用摄像头采集视频的应用, 与`AVCaptureSession` 结合来用可以更好地控制音视频的输入输出路径。(例如，设置 `automaticallyConfiguresApplicationAudioSession` 属性，系统会自动选择最佳输出路径.)

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

## 其他的坑

### PlayAndRecord
在一般情况下，当没有接入任何音频设备时，声音会通过**扬声器**来播放，但是一旦设置了`PlayAndRecord`,会将默认的输出设备转为**听筒**。

解决办法：
```swift
// 第一种：通过overrideOutputAudioPort 方法设置
AVAudioSession.sharedInstance().overrideOutputAudioPort(AVAudioSession.PortOverride.speaker)
// 第二种：设置AVAudioSessionCategoryOptionDefaultToSpeaker
AVAudioSession.sharedInstance().setCategory(.playAndRecord, options: .defaultToSpeaker)
```
第三种：[MPVolumeView](https://developer.apple.com/documentation/mediaplayer/mpvolumeview) 让用户自己选择

### AirPods
AirPods在不同系统上的表现
AirPods 系列耳机，对于系统的要求不一样，具体要求可以去[官网](https://support.apple.com/zh-cn/HT207974)查看
![图片来自苹果官网](https://imagedatabase-1259343097.cos.ap-beijing.myqcloud.com/blogImage/WX20211026-204244%402x.png)
例如：
AirPods在iOS10.2以下表现和普通的蓝牙耳机类似，能手动通过蓝牙连接上手机。
AirPods在iOS10.2以上的能支持双击操作，双击播放音乐，双击停止音乐；分别对应远程线控中的`UIEventSubtypeRemoteControlPlay`、`UIEventSubtypeRemoteControlStop`等事件。

判断是否是AirPod
```swift
func isAirPods() -> Bool {
        let des = AVAudioSession.sharedInstance().currentRoute.outputs
        for de in des {
            if de.portName.contains("AirPods") {
                return true
            }
        }
        return false
    }
```
**1. 远程控制的坑**

在使用AirPods 的情况下，`AVAudioSession` 的category 设置为`AVAudioSessionCategoryPlayback`，此时APP只用于播放音频，能够**自动适配远程控制**。**但是**如果是`AVAudioSessionCategoryPlayAndRecord`，此时APP既需要播放音频，也需要录制音频。**不能自动适配远程控制** 
解决办法：
```swift
// 设置option .alllowBluetooth  PS: 设备需要的系统不满足使用
try? AVAudioSession.sharedInstance().setCategory(.playAndRecord, options: .allowBluetooth)
// 设置option .allowBluetoothA2DP  PS: 设备需要的系统满足使用 且 iOS10 以后才有这个选项
try? AVAudioSession.sharedInstance().setCategory(.playAndRecord, options: .allowBluetoothA2DP)
```
**2. 音频输出的问题**

对于一般情况：
 - `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeDefault` + `AVAudioSessionCategoryOptionDefaultToSpeaker` 声音会通过扬声器播放出来
 - `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeDefault` 声音会通过听筒播放出来


**但是**在AirPod的情况下:

- `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeDefault` + `AVAudioSessionCategoryOptionAllowBluetoothA2DP` 能连接上，并能远程控制。

- `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeDefault` + `AVAudioSessionCategoryOptionAllowBluetooth` 能连接上，但是不能远程控制。

如果是设置的其它模式，比如设置了模式，坑坑坑坑

- `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeVoiceChat`
- `AVAudioSessionCategoryPlayAndRecord` + `AVAudioSessionModeDefault` + `AVAudioSessionCategoryOptionDefaultToSpeaker`

不能连上Airpods了，并且在控制中心面板中也没有AirPods选项。

**3. 录制播放声音不清晰**

打Log发现，默认的`AVAudioSessionCategoryPlayback` 模式下，耳机使用的模式是`BluetoothA2DPOutput`

**但是** 在`AVAudioSessionCategoryPlayAndRecord` 模式下，耳机使用的是`BluetoothHFP`
> - HeadsetPro-file（HSP）代表耳机功能，提供手机与耳机之间通信所需的基本功能。
> - HandProfile（HFP）则代表免提功能，HFP在HSP的基础上增加了某些扩展功能。
> - Advanced Audio Distribution Profile（A2DP），指的是蓝牙音频传输模型协定。
> 
> HFP格式的蓝牙耳机支持手机功能比较完整，用户可在耳机上操作手机设定好的重拨、来电保留、来电拒听等免提选项功能。A2DP是高级音频传送规格，允许传输立体声音频信号，相比用于 HSP 和 HFP 的单声道加密，质量要好得多。
> https://www.jianshu.com/p/04a1dc879c13

在Apple 官方描述中：
> "If an application uses the setPreferredInput:error: method to select a Bluetooth HFP input, the output will automatically be changed to the Bluetooth HFP output. **Moreover, selecting a Bluetooth HFP output using the MPVolumeView's route picker will automatically change the input to the Bluetooth HFP input. Therefore both the input and output will always end up on the Bluetooth HFP device even though only the input or output was set individually.**"

大众的解决方案：

- 维持原状的方案，就是继续采用HFP来进行音频的输入和输出，这种方案可以保证输入音频由支架的麦克风来提供，输出继续由支架来进行转发。但是，问题就是会导致音乐播放的音质差.

- 采用A2DP来进行高质量的蓝牙音频输出，保证歌曲、声音的播放质量，采用手机麦克风来进行音频输入而不通过蓝牙来采集音频。





## 总结

以上就是对于一些音频知识的总结，本篇主要是针对`AVAudioSession`做的分析，若有问题，请还望多包涵和指正。下一小节将会去介绍音频处理相关的内容。

