# 苹果的VideoToolbox,H264编解码

## 前言

之前做macOS共享桌面的时候因为要实时同步,所以会用到H264编解码.苹果自带的库VideoToolbox正好可以解决这个问题.

### VideoToolbox

> VideoToolbox is a low-level framework that provides direct access to hardware encoders and decoders. It provides services for video compression and decompression, and for conversion between raster image formats stored in CoreVideo pixel buffers. These services are provided in the form of session objects (compression, decompression, and pixel transfer), which are vended as Core Foundation (CF) types. Apps that don’t need direct access to hardware encoders and decoders shouldn’t need to use VideoToolbox directly.

>  Work directly with hardware-accelerated video encoding and decoding capabilities.


> VideoToolbox 是一个低级框架，可提供对硬件编码器和解码器的直接访问。 它提供视频压缩和解压缩服务，以及存储在 CoreVideo 像素缓冲区中的光栅图像格式之间的转换。 这些服务以会话对象（压缩、解压缩和像素传输）的形式提供，它们作为 Core Foundation (CF) 类型出售。 不需要直接访问硬件编码器和解码器的应用不需要直接使用 VideoToolbox。

> 直接使用硬件加速视频编码和解码功能。

### AVSampleBufferDisplayLayer

> AVSampleBufferDisplayLayer is a subclass of CALayer that can decompress and display compressed or uncompressed video frames.

> AVSampleBufferDisplayLayer 是 CALayer 的子类，可以解压缩并显示压缩或未压缩的视频帧。

先说一下苹果的H264如何播放,我们使用AVSampleBufferDisplayLayer来显示,属于AVFoundation框架中的一员

- 初始化

```objc
@import AVFoundation;
@property (strong, nonatomic) AVSampleBufferDisplayLayer *avsLayer;
- (void)initUI {
    self.avsLayer = [[AVSampleBufferDisplayLayer alloc] init];
    self.avsLayer.videoGravity = AVLayerVideoGravityResizeAspect;
    self.avsLayer.opaque = YES;
    self.avsLayer.backgroundColor = 背景颜色;
    [self.layer addSublayer:self.avsLayer];
}
```

- 布局

```objc
- (void)layout
{
    [super layout];
    self.avsLayer.frame = self.bounds;
    self.avsLayer.position = CGPointMake(CGRectGetMidX(self.bounds), CGRectGetMidY(self.bounds));
}
```

- 传入 CMSampleBufferRef 数据, 播放

```objc
- (void)playVideoWithBuffer:(CMSampleBufferRef)sampleBuffer {
    /*
    如果 sampleBuffer 为 NULL 或调用了 CMSampleBufferInvalidate(sampleBuffer)，则返回 false，否则返回 true。
    */
    if (!CMSampleBufferIsValid(sampleBuffer)) {
        // 错误处理
        return;
    }
    
    /*
    isReadyForMoreMediaData 指示层准备好接受更多样本缓冲区。
    */
    if (![self.avsLayer isReadyForMoreMediaData]) {
        // 错误处理
        return;
    }
    
    /*
    发送样本缓冲区以供显示。
    */
    [self.avsLayer enqueueSampleBuffer:sampleBuffer];
}
```

- 清理

```objc
/*
指示层丢弃挂起的排队样本缓冲区并删除任何当前显示的图像。
*/
- (void)dismiss {
    [self.avsLayer flushAndRemoveImage];
}
```

## Encode

分为3个部分
- 初始化配置
- 开始编码
- 结束编码

### 初始化配置

```objc
@import VideoToolbox;

// 宽高,fps
@property (assign, nonatomic) int32_t width;
@property (assign, nonatomic) int32_t height;
@property (assign, nonatomic) int fps;

// 每一帧生成CMTime使用的frameID++，计数器
@property (nonatomic, assign) int64_t frameID;

// block回调
// data i帧,p帧数据
// pps 略
// sps 略
// isKeyFrame 标记是否为关键帧
typedef void(^H264DataBlock)(NSData *_Nullable data, NSData *_Nullable pps, NSData *_Nullable sps, BOOL isKeyFrame);
@property (nonatomic, copy) H264DataBlock h264DataBlock;

// 串行编码队列
@property (nonatomic) dispatch_queue_t encodeQueue;

// 编码会话
@property (nonatomic, assign) VTCompressionSessionRef compressionSession;
```

- 传入自定义的值

```objc
- (void)configWithVideoWidth:(int32_t)videoWidth videoHeight:(int32_t)videoHeight fps:(int)fps {
        // 采集视频设置的分辨率
    self.width = videoWidth;
    self.height = videoHeight;
    self.fps = fps;
    self.encodeQueue = dispatch_queue_create("com.qixin.encode.h264", DISPATCH_QUEUE_SERIAL);
    [self setupVideoToolBox];
}
```

- 配置

```objc
- (void)setupVideoToolBox {
    
    self.frameID = 0;
    // width->视频的宽 height->视频的高 编码格式H264->kCMVideoCodecType_H264
    // OC桥接对象 -> (__bridge void *)(self)，会话对象->compressionSession
    
    // 1. 创建编码器session
    OSStatus status = VTCompressionSessionCreate(NULL, self.width, self.height, kCMVideoCodecType_H264, NULL, NULL, NULL, didCompressionH264, (__bridge void *)(self), &_compressionSession);
    NSLog(@"VTCompressionSessionCreate status == %d", status);
    if (status != noErr) {
        NSLog(@"unable to create H264 session -> VTCompressionSessionCreate");
    }
    
    // 1.实时编码输出(避免延时)
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_RealTime, kCFBooleanTrue);
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_ProfileLevel, kVTProfileLevel_H264_Baseline_AutoLevel);
    
    // 2. 是否产生B帧 -> kCFBooleanFalse -> 丢弃B帧
    // 通常做法不产生B帧确保实时性(因为B帧在解码时并不是必要的,是可以抛弃B帧的)
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_AllowFrameReordering, kCFBooleanFalse);
    
    // 3. 设置关键帧(GOPsize)间隔
    int frameInterval = self.fps*2;
    CFNumberRef frameIntervalRef = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &frameInterval);
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_MaxKeyFrameInterval, frameIntervalRef);
    CFRelease(frameIntervalRef);
    
    // 4. 设置期望帧率
    int fps = self.fps;
    CFNumberRef fpsRef = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &fps);
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_ExpectedFrameRate, fpsRef);
    CFRelease(fpsRef);

    // 5. 设置码率均值
    int bitRate = 88473600;//self.width * self.height * 3 * 4 * 8;
    CFNumberRef bitRateRef = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &bitRate);
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_AverageBitRate, bitRateRef);
    CFRelease(bitRateRef);

    // 6. 设置码率上限
    int bigRateLimit = bitRate/8;
    CFNumberRef bigRateLimitRef = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &bigRateLimit);
    VTSessionSetProperty(self.compressionSession, kVTCompressionPropertyKey_DataRateLimits, bigRateLimitRef);
    CFRelease(bigRateLimitRef);

    
    //7. 设置切片最大大小
    int numberOfSlices = 1;
    CFStringRef numberOfSlicesKey = CFStringCreateWithCString(kCFAllocatorDefault, "NumberOfSlices", kCFStringEncodingUTF8);
    CFNumberRef numberOfSlicesRef = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &numberOfSlices);
    status = VTSessionSetProperty(self.compressionSession, numberOfSlicesKey, numberOfSlicesRef);
    if (status != noErr) {
        NSLog(@"NumberOfSlices Error (%d)",status);
    }
    CFRelease(numberOfSlicesKey);
    CFRelease(numberOfSlicesRef);
    
    // 8. 准备开始编码
    VTCompressionSessionPrepareToEncodeFrames(self.compressionSession);
}
```

- 编码器回调

```objc
// 2. 回调函数->解析出SPS PPS NALU 单元
// 编码完成回调函数-> VTCompressionSessionCreate -> didCompressionH264
void (didCompressionH264)(void * CM_NULLABLE outputCallbackRefCon, void * CM_NULLABLE sourceFrameRefCon, OSStatus status, VTEncodeInfoFlags infoFlags, CM_NULLABLE CMSampleBufferRef sampleBuffer) {
    
    // 1. 状态判断
    if (status != noErr) {
        NSLog(@"didCompressionH264 error");
        return;
    }
    
    // 2. 判断CMSampleBufferRef是否准备OK
    if (!CMSampleBufferDataIsReady(sampleBuffer)) {
        NSLog(@"CMSampleBufferRef is not ready");
        return;
    }
    
    // 3.OC桥接对象转换
    VideoH264Encoder * encoder = (__bridge VideoH264Encoder*)outputCallbackRefCon;
    
    // 4. 判断是否是关键帧 ->是关键帧获取pps 和 sps 数据
    bool isKeyFrame = !CFDictionaryContainsKey(CFArrayGetValueAtIndex(CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, true), 0), kCMSampleAttachmentKey_NotSync);
    
    if(isKeyFrame) {
        // 5. 获取编码后的信息，存储于CMFormatDescriptionRef中
        CMFormatDescriptionRef formatDescription = CMSampleBufferGetFormatDescription(sampleBuffer);
        
        // index sps = 0, pps = 1
        const uint8_t * spsParameterSet;
        size_t spsParameterSetSize, spsParameterCount;
        // 6.获取SPS数据 格式描述 parameterSetPointer指针 size count
        OSStatus status =  CMVideoFormatDescriptionGetH264ParameterSetAtIndex(formatDescription, 0, &spsParameterSet, &spsParameterSetSize, &spsParameterCount, 0);
        if (status != noErr) {
            NSLog(@"CMVideoFormatDescriptionGetH264ParameterSetAtIndex get SPS failed");
        }
        
        // index sps = 0, pps = 1
        // 7.获取PPS数据 格式描述 parameterSetPointer指针 size count
        const uint8_t * ppsParmeterSet;
        size_t ppsParameterSetSize, ppsParameterCount;
        CMVideoFormatDescriptionGetH264ParameterSetAtIndex(formatDescription, 1, &ppsParmeterSet, &ppsParameterSetSize, &ppsParameterCount, 0);
        if (status != noErr) {
            NSLog(@"CMVideoFormatDescriptionGetH264ParameterSetAtIndex get PPS failed");
        }
        
        // 8. SPS PPS 封装成NSData
        NSData * spsData = [NSData dataWithBytes:spsParameterSet length:spsParameterSetSize];
        NSData * ppsData = [NSData dataWithBytes:ppsParmeterSet length:ppsParameterSetSize];
        
        if (encoder.h264DataBlock)
        {
            encoder.h264DataBlock(nil, ppsData, spsData, NO);
        }
    }
    
    // 10. 获取编码后的视频数据CMBlockBuffer
    CMBlockBufferRef blockBuffer = CMSampleBufferGetDataBuffer(sampleBuffer);
    size_t length, totalLength;
    char * dataPointer;
    
    OSStatus statusCodeRet = CMBlockBufferGetDataPointer(blockBuffer, 0, &length, &totalLength, &dataPointer);
    if (statusCodeRet == noErr) {
        size_t bufferOffset = 0;
        // 返回的nalu数据前四个字节不是0001的startcode，而是大端模式的帧长度length
        static const int AVCCHeaderLength = 4;
        while (bufferOffset < totalLength - AVCCHeaderLength) {
            uint32_t NALUUnitLength = 0;
            
            // 1. 读取前4个字节，返回帧长度 保存在NALUUnitLength里，Read the NAL unit length
            memcpy(&NALUUnitLength, dataPointer + bufferOffset, AVCCHeaderLength);
            
            // 2. 大端转系统端 -> iOS MacOS小端存储
            NALUUnitLength = CFSwapInt32BigToHost(NALUUnitLength);
            
            // 3.读取CMBlockBufferRef数据
            // 移动指针从dataPointer + bufferOffset + AVCCHeaderLength位置读取NALUUnitLength长度
            NSData * blockData = [[NSData alloc] initWithBytes:(dataPointer + bufferOffset + AVCCHeaderLength) length:NALUUnitLength];
            
            
            uint8_t type = 0;
            [blockData getBytes:&type length:1];
            
            if ((type%16) == 5)//关键帧
            {
                if (encoder.h264DataBlock)
                {
                    encoder.h264DataBlock(blockData, nil, nil, isKeyFrame);
                }
            }
            else if ((type%16) == 1)//非关键帧
            {
                if (encoder.h264DataBlock)
                {
                    encoder.h264DataBlock(blockData, nil, nil, isKeyFrame);
                }
            }

            
            // 4. 移动到下一个块，转成NALU单元 //移动下标，继续读取下一个数据
            bufferOffset += NALUUnitLength + AVCCHeaderLength;
        }
    }
}
```

### 开始编码

```objc
- (void)encodeCMSampleBuffer:(CMSampleBufferRef)sampleBuffer h264DataBlock:(H264DataBlock)h264DataBlock
{
    CFRetain(sampleBuffer);

    dispatch_async(self.encodeQueue, ^{
        if (!self.compressionSession) {
            CFRelease(sampleBuffer);
            return;
        }
        // 1.保存block回调
        self.h264DataBlock = h264DataBlock;
        
        // 2. CMSampleBufferRef -> CVImageBufferRef
        CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
        
        // 3. 根据当前帧数，创建CMTime的时间
        CMTime presentationTimeStamp = CMTimeMake(self.frameID++, 1000);
        VTEncodeInfoFlags flags;
        
        // 4. 开始编码该帧数据
        // 参数 1.compressionSession会话 2.imageBuffer视频数据 3.presentationTimeStamp时间戳
        // 4.(__bridge void *)(self)OC桥接对象
        OSStatus status = VTCompressionSessionEncodeFrame(self.compressionSession, imageBuffer, presentationTimeStamp, kCMTimeInvalid, NULL, (__bridge void *)(self), &flags);
        if (status != noErr) {
            NSLog(@"VTCompressionSessionEncodeFrame failed");
        }

        CFRelease(sampleBuffer);
    });
}
```

### 结束编码

```objc
- (void)stopEncode
{
    // 1.停止编码
    VTCompressionSessionCompleteFrames(self.compressionSession, kCMTimeInvalid);
    
    // 2.使得会话无效
    VTCompressionSessionInvalidate(self.compressionSession);
    
    // 3.释放会话资源
    if (self.compressionSession) {
        CFRelease(self.compressionSession);
        self.compressionSession = NULL;
    }
}
```

## Decode

```objc
@import VideoToolbox;

// 串行编码队列
@property (nonatomic) dispatch_queue_t decodeQueue;

// 编码会话
@property (nonatomic, assign) VTDecompressionSessionRef decoderSession;
@property (nonatomic, assign) CMVideoFormatDescriptionRef decoderFormatDescription;

// 缓存sps和pps数据
@property (nonatomic, strong) NSData *sps;
@property (nonatomic, strong) NSData *pps;

```

```objc
self.decoder.decodeQueue = dispatch_queue_create("com.qixin.decode.h264", DISPATCH_QUEUE_SERIAL);
```

- 解码,最终会通过block返回sampleBuffer

```objc
- (void)decodeWithData2:(NSData*)data sps:(NSData*)sps pps:(NSData*)pps decodeFinish:(H264DecodeBlock2)decodeFinish
{
    dispatch_async(self.decodeQueue, ^{
        
        if (sps && pps)
        {
            self.sps = sps;
            self.pps = pps;
        }
        else if (data)
        {
            if (!self.decoderSession)
            {
                const uint8_t* const parameterSetPointers[2] = { self.sps.bytes, self.pps.bytes };
                const size_t parameterSetSizes[2] = {(size_t)(self.sps.length), (size_t)(self.pps.length)};
                OSStatus status = CMVideoFormatDescriptionCreateFromH264ParameterSets(kCFAllocatorDefault,
                                                                                      2, //param count
                                                                                      parameterSetPointers,
                                                                                      parameterSetSizes,
                                                                                      4, //nal start code size
                                                                                      &self->_decoderFormatDescription);
                
                if(status == noErr)
                {
                    CFDictionaryRef attrs = NULL;
                    const void *keys[] = { kCVPixelBufferPixelFormatTypeKey };
                    uint32_t v = kCVPixelFormatType_420YpCbCr8BiPlanarFullRange;
                    const void *values[] = { CFNumberCreate(NULL, kCFNumberSInt32Type, &v) };
                    attrs = CFDictionaryCreate(NULL, keys, values, 1, NULL, NULL);
                    
                    VTDecompressionOutputCallbackRecord callBackRecord;
                    callBackRecord.decompressionOutputCallback = didDecompress;
                    callBackRecord.decompressionOutputRefCon = NULL;
                    
                    status = VTDecompressionSessionCreate(kCFAllocatorDefault,
                                                          self->_decoderFormatDescription,
                                                          NULL, attrs,
                                                          &callBackRecord,
                                                          &self->_decoderSession);
                    CFRelease(attrs);
                }
                else
                {
                    NSLog(@"IOS8VT: reset decoder session failed status=%d", (int)status);
                }
            }
            
            
            CMBlockBufferRef blockBuffer = NULL;
            CMSampleBufferRef sampleBuffer = NULL;
            
            OSStatus status = CMBlockBufferCreateWithMemoryBlock(kCFAllocatorDefault,
                                                                 (void*)data.bytes, data.length,
                                                                 kCFAllocatorNull,
                                                                 NULL, 0, data.length,
                                                                 0, &blockBuffer);
            if (status == kCMBlockBufferNoErr)
            {
                const size_t sampleSizeArray[] = {data.length};
                
                status = CMSampleBufferCreateReady(kCFAllocatorDefault,
                                                   blockBuffer,
                                                   self->_decoderFormatDescription,
                                                   1, 0, NULL, 1, sampleSizeArray,
                                                   &sampleBuffer);
                CFRelease(blockBuffer);
                CFArrayRef attachments = CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, YES);
                CFMutableDictionaryRef dict = (CFMutableDictionaryRef)CFArrayGetValueAtIndex(attachments, 0);
                CFDictionarySetValue(dict, kCMSampleAttachmentKey_DisplayImmediately, kCFBooleanTrue);
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (decodeFinish)
                    {
                        decodeFinish(sampleBuffer);
                    }
                });
            }
        }
    });
}
```

获取到sampleBuffer之后,您可以使用AVSampleBufferDisplayLayer来显示.


## 未完可能待续

