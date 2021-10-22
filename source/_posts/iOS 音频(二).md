---
title: iOS 音频（二）
date: 2021-10-22 19:20:32
tags:
- iOS
- 音频
categories: iOS开发
---

> 接上一节的音频，本章主要将音频处理问题

## iOS上的PCM

通常来说，第一时间肯定是想的是格式之间的转换。现在网上有很多`PCM` 转换到各种类型的方案。例如：

- PCM -> wav [pcm 转 wav代码](https://www.jianshu.com/p/007fd82b0940)
- pcm -> mp3 [LAME](https://lame.sourceforge.io/)
- 。。。

> 但是，我几乎没有找到其他格式能直接转换为`PCM`的代码(有可能是没有找到），若是有这一块需求的同学，建议联系sdk去做处理
有一点值得提的是，在iOS上，通过`AVAudioRecorder`录制的音频格式默认为PCM，其初始化的方法为：
<!--more-->  

```objective-c
- (nullable instancetype)initWithURL:(NSURL *)url settings:(NSDictionary<NSString *, id> *)settings error:(NSError **)outError;
```

在这其中，iOS7 中。默认的setting是：

```objective-c
{ AVFormatIDKey = kAudioFormatLinearPCM;  
AVLinearPCMBitDepthKey = 16;
AVLinearPCMIsBigEndianKey = 0;
AVLinearPCMIsFloatKey = 0; 
AVLinearPCMIsNonInterleaved = 0;
AVNumberOfChannelsKey = 2; 
AVSampleRateKey = 44100;}
```

其中，比较重要的几个参数：

- AVFormatIDKey： 音频格式的标识，具体的分类的种类见下面的代码
- AVLinearPCMBitDepthKey: 位宽
- AVNumberOfChannelsKey： 通道数
- AVSampleRateKey： 采样率

```objective-c
CF_ENUM(AudioFormatID)
{
    kAudioFormatLinearPCM               = 'lpcm',
    kAudioFormatAC3                     = 'ac-3',
    kAudioFormat60958AC3                = 'cac3',
    kAudioFormatAppleIMA4               = 'ima4',
    kAudioFormatMPEG4AAC                = 'aac ',
    kAudioFormatMPEG4CELP               = 'celp',
    kAudioFormatMPEG4HVXC               = 'hvxc',
    kAudioFormatMPEG4TwinVQ             = 'twvq',
    kAudioFormatMACE3                   = 'MAC3',
    kAudioFormatMACE6                   = 'MAC6',
    kAudioFormatULaw                    = 'ulaw',
    kAudioFormatALaw                    = 'alaw',
    kAudioFormatQDesign                 = 'QDMC',
    kAudioFormatQDesign2                = 'QDM2',
    kAudioFormatQUALCOMM                = 'Qclp',
    kAudioFormatMPEGLayer1              = '.mp1',
    kAudioFormatMPEGLayer2              = '.mp2',
    kAudioFormatMPEGLayer3              = '.mp3',
    kAudioFormatTimeCode                = 'time',
    kAudioFormatMIDIStream              = 'midi',
    kAudioFormatParameterValueStream    = 'apvs',
    kAudioFormatAppleLossless           = 'alac',
    kAudioFormatMPEG4AAC_HE             = 'aach',
    kAudioFormatMPEG4AAC_LD             = 'aacl',
    kAudioFormatMPEG4AAC_ELD            = 'aace',
    kAudioFormatMPEG4AAC_ELD_SBR        = 'aacf',
    kAudioFormatMPEG4AAC_ELD_V2         = 'aacg',
    kAudioFormatMPEG4AAC_HE_V2          = 'aacp',
    kAudioFormatMPEG4AAC_Spatial        = 'aacs',
    kAudioFormatMPEGD_USAC              = 'usac',
    kAudioFormatAMR                     = 'samr',
    kAudioFormatAMR_WB                  = 'sawb',
    kAudioFormatAudible                 = 'AUDB',
    kAudioFormatiLBC                    = 'ilbc',
    kAudioFormatDVIIntelIMA             = 0x6D730011,
    kAudioFormatMicrosoftGSM            = 0x6D730031,
    kAudioFormatAES3                    = 'aes3',
    kAudioFormatEnhancedAC3             = 'ec-3',
    kAudioFormatFLAC                    = 'flac',
    kAudioFormatOpus                    = 'opus'
};

```

关于`AVAudioRecorder`的使用方法具体就不贴了,网上一找一大堆，关于控制的话，`AVAudioRecorder`提供了一个`AVAudioRecorderDelegate` 作为回调，方便我们控制。

```objective-c
/* A protocol for delegates of AVAudioRecorder */
API_AVAILABLE(macos(10.7), ios(3.0), watchos(4.0)) API_UNAVAILABLE(tvos)
@protocol AVAudioRecorderDelegate <NSObject>
@optional 

/* audioRecorderDidFinishRecording:successfully: is called when a recording has been finished or stopped. This method is NOT called if the recorder is stopped due to an interruption. */
- (void)audioRecorderDidFinishRecording:(AVAudioRecorder *)recorder successfully:(BOOL)flag;

/* if an error occurs while encoding it will be reported to the delegate. */
- (void)audioRecorderEncodeErrorDidOccur:(AVAudioRecorder *)recorder error:(NSError * __nullable)error;

#if TARGET_OS_IPHONE

/* AVAudioRecorder INTERRUPTION NOTIFICATIONS ARE DEPRECATED - Use AVAudioSession instead. */

/* audioRecorderBeginInterruption: is called when the audio session has been interrupted while the recorder was recording. The recorded file will be closed. */
- (void)audioRecorderBeginInterruption:(AVAudioRecorder *)recorder NS_DEPRECATED_IOS(2_2, 8_0);

/* audioRecorderEndInterruption:withOptions: is called when the audio session interruption has ended and this recorder had been interrupted while recording. */
/* Currently the only flag is AVAudioSessionInterruptionFlags_ShouldResume. */
- (void)audioRecorderEndInterruption:(AVAudioRecorder *)recorder withOptions:(NSUInteger)flags NS_DEPRECATED_IOS(6_0, 8_0);

- (void)audioRecorderEndInterruption:(AVAudioRecorder *)recorder withFlags:(NSUInteger)flags NS_DEPRECATED_IOS(4_0, 6_0);

/* audioRecorderEndInterruption: is called when the preferred method, audioRecorderEndInterruption:withFlags:, is not implemented. */
- (void)audioRecorderEndInterruption:(AVAudioRecorder *)recorder NS_DEPRECATED_IOS(2_2, 6_0);

#endif // TARGET_OS_IPHONE

@end
```

AVAudioRecorderDelegate总共有6个回调，在iOS6.0中废弃掉两个，然后在iOS8.0又废掉两个。在以前的版本中被抛弃的回调接口，主要也是控制播放器在播放的过程中收到中断

比如，收到中断：

```
- (void)audioRecorderBeginInterruption:(AVAudioRecorder *)recorder
```

## 音视频编辑

在iOS 中，音频编辑离不开`AVMutableComposition`，`AVMutableComposition` 提供了接口来插入或者删除轨道，也可以调整这些轨道的顺序。一个 `composition` 可以简单的认为是一组轨道（tracks）的集合，这些轨道可以是来自不同媒体资源（asset）。

> 前提知识：一个工程文件中有很多轨道，如音频轨道1，音频轨道2，音频轨道3，视频轨道1，视频轨道2等等，每个轨道里有许多素材，对于每个视频素材，它可以进行缩放、旋转等操作，素材库中的视频拖到轨道中会分为视频轨和音频轨两个轨道。[参考](https://www.jianshu.com/p/ee868d0ff7c4)
>
> ```objectivec
> AVAsset：素材库里的素材； 
> AVAssetTrack：素材的轨道； 
> AVMutableComposition ：一个用来合成视频的工程文件； 
> AVMutableCompositionTrack ：工程文件中的轨道，有音频轨、视频轨等，里面可以插入各种对应的素材； 
> AVMutableVideoCompositionLayerInstruction：视频轨道中的一个视频，可以缩放、旋转等； 
> AVMutableVideoCompositionInstruction：一个视频轨道，包含了这个轨道上的所有视频素材； 
> AVMutableVideoComposition：管理所有视频轨道，可以决定最终视频的尺寸，裁剪需要在这里进行； 
> AVAssetExportSession：配置渲染参数并渲染。
> ```

一般来说，类似一个简单的mp3文件，它就只会有音频轨道。而对于一个mp4文件，它会有音频轨道、视频轨道（也可能没有，在写代码时，尤其需要注意），其次，PCM格式的文件是不存在轨道这类信息的，所以如果想要将PCM添加到视频中，一种比较笨的方法是，将PCM 先转换为mp3格式，再通过得到的mp3文件提取出音频轨道。

以一个例子来具体解释，需求很简单，我想将一个视频的音频替换成一个pcm文件，并保存为，mov的格式：

首先：初始化AVMutableComposition

```objective-c
AVMutableComposition *composition = [AVMutableComposition composition];
```

其次，我需要获取两个资源文件，并且提取音轨/视频轨，并插入到`composition`。

```objective-c
AVURLAsset *videoAsset = [AVURLAsset assetWithURL:[NSURL fileURLWithPath:videoPath]];
AVURLAsset *mp3Asset = [AVURLAsset assetWithURL:mp3Path];
// 音频轨道提取
AVMutableCompositionTrack *compositionAudioTrack = [composition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];
        [compositionAudioTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, mp3Asset.duration) ofTrack:[[mp3Asset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0] atTime:kCMTimeZero error:nil];
// 视频轨道提取
AVMutableCompositionTrack *compositionVideoTrack = [composition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
    [compositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, videoAsset.duration) ofTrack:[[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0] atTime:kCMTimeZero error:nil];
```

导出工程，使用`AVAssetExportSession`:

```objective-c
 AVAssetExportSession* assetExport = [[AVAssetExportSession alloc]initWithAsset:composition presetName:AVAssetExportPresetPassthrough];
    assetExport.outputFileType = AVFileTypeQuickTimeMovie;
    assetExport.outputURL = exportUrl;
    assetExport.shouldOptimizeForNetworkUse = YES;
    [assetExport exportAsynchronouslyWithCompletionHandler:
     ^(void ) {
        // 导出结果
    }];
```
注意，我当前导出的格式为`mov`, 可选择的格式可以去 `AVMediaFormat` 中 `AVFileType` 去查看。
## 小结

本章主要是对音频处理这块，有一个简单的补充，可能后续遇到问题，会继续补充。如果有不对的地方，也欢迎及时纠正我，谢谢～
