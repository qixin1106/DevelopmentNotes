# macOS屏幕录制

## 前言

之前参与的多人语音会议项目中,这里要实现类似腾讯会议的共享桌面功能.
其中主要需要实现以下几个功能:

- 选择窗口/屏幕
- 实现屏幕录制并返回buff(CMSampleBufferRef)


### 获取窗口/屏幕模型

我们先自定义定义一个需要展示用的Model

```objc
/// 类型enum
typedef NS_ENUM(NSUInteger, DisplayType) {
    DisplayType_Screen = 0,
    DisplayType_Window = 1,
};
@interface XYSelectedScreenModel : NSObject
/// 缩略图
@property (strong, nonatomic) NSImage *image;
/// 标题
@property (copy, nonatomic) NSString *title;        
/// 类型标记
@property (assign, nonatomic) DisplayType type; 
/// 分配的id, 这个很重要,录制屏幕时需要用到
@property (assign, nonatomic) uint32_t objId;

/// 获取model数组方法
+ (NSArray*)getDisplayModels;
@end
```


- 获取窗口和屏幕的model

```objc
+ (NSArray*)getDisplayModels;
{
    NSMutableArray<XYSelectedScreenModel*> *temp = @[].mutableCopy;
    // 获取屏幕模型
    [temp addObjectsFromArray:[XYSelectedScreenModel getDisplayScreen]];
    // 获取窗口模型 , 暂时没有用到.
    //[temp addObjectsFromArray:[XYSelectedScreenModel getAllShowWindow]];
    return [temp copy];
}
```

- 获取所有屏幕model

```objc
+ (NSArray*)getDisplayScreen
{
    CGDirectDisplayID dspyIDArray[10];
    uint32_t dspyIDCount = 0;
    if (CGGetActiveDisplayList(10, dspyIDArray, &dspyIDCount) != kCGErrorSuccess)
    {
        return @[];
    }
    
    NSMutableArray<XYSelectedScreenModel*> *temp = @[].mutableCopy;
    
    for (uint32_t i = 0; i < dspyIDCount; i++)
    {
        XYSelectedScreenModel *model = [[XYSelectedScreenModel alloc] init];
        model.type = DisplayType_Screen;
        if (dspyIDCount>1)
        {
            model.title = STR_FORMAT(@"屏幕%d",i+1);
        }
        else
        {
            model.title = @"屏幕";
        }
        {
            CGDirectDisplayID mainID = dspyIDArray[i];
            // 根据Quartz分配给显示器的id，生成显示器mainID的截图
            CGImageRef mainCGImage = CGDisplayCreateImage(mainID);
            
            NSImage *image = [[NSImage alloc] initWithCGImage: mainCGImage size:NSZeroSize];
            CGImageRelease(mainCGImage);
            model.image = image;
            model.objId = mainID;
        }
        [temp addObject:model];
    }

    return [temp copy];
}
```
- 获取所有窗口model

```objc
+ (NSArray*)getAllShowWindow
{
    NSMutableArray<XYSelectedScreenModel*> *temp = @[].mutableCopy;

    NSArray *windows = (NSArray*)CFBridgingRelease(CGWindowListCopyWindowInfo(kCGWindowListOptionOnScreenOnly, kCGNullWindowID));
    [windows enumerateObjectsUsingBlock:^(NSDictionary *_Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        
        NSInteger kCGWindowLayer = [obj[@"kCGWindowLayer"] integerValue];
        NSString *kCGWindowOwnerName = obj[@"kCGWindowOwnerName"];
        uint32_t kCGWindowNumber = [obj[@"kCGWindowNumber"] unsignedIntValue];
        
        if (kCGWindowLayer==0)
        {
            XYSelectedScreenModel *model = [[XYSelectedScreenModel alloc] init];
            model.type = DisplayType_Window;
            model.objId = kCGWindowNumber;
            model.title = kCGWindowOwnerName;
            
            // 生成缩略图
            CGImageRef screenShot = CGWindowListCreateImage(CGRectNull, kCGWindowListOptionIncludingWindow, kCGWindowNumber, kCGWindowImageBoundsIgnoreFraming|
            kCGWindowImageNominalResolution);
            NSImage *image = [[NSImage alloc] initWithCGImage:screenShot size:NSZeroSize];
            CGImageRelease(screenShot);
            model.image = image;
            
            [temp addObject:model];
        }
        
    }];
    
    return [temp copy];
}
```

关于UI部分暂时略,我们使用NSCollectionView来展示.

### 录制屏幕

只记录一些核心方法,细节方法待补充.
首先我们需要定义一些用到的属性,和需要实现的协议
我们主要利用AVFoundation提供的屏幕录制API

```objc
@import AVFoundation;
/// public
@property (assign, nonatomic, readonly) CGDirectDisplayID displayId;
@property (assign, nonatomic, readonly) CGFloat width;
@property (assign, nonatomic, readonly) CGFloat height;
@property (assign, nonatomic, readonly) int fps;

/// private
@property (strong, nonatomic) AVCaptureSession *session;
@property (strong, nonatomic) AVCaptureScreenInput *input;
@property (strong, nonatomic) AVCaptureDeviceInput *audioInput;
@property (strong, nonatomic) AVCaptureVideoDataOutput *output;
@property (copy, nonatomic) GetScreenSampleBuffer block;
```
```objc
/// 实现<AVCaptureVideoDataOutputSampleBufferDelegate>协议
- (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
{
    if (self.block) {
        self.block(sampleBuffer);
    }
}
```

- 初始化+配置

```objc
- (void)configWithWindowDisplayId:(CGDirectDisplayID)displayId width:(CGFloat)width height:(CGFloat)height fps:(int)fps;
{
    _displayId = displayId;
    _width = width;
    _height = height;
    _fps = fps;
    
    
    
    _session = [[AVCaptureSession alloc] init];
    
    _input = [[AVCaptureScreenInput alloc] initWithDisplayID:displayId];
    
    _output = [[AVCaptureVideoDataOutput alloc] init];
    // kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange 表示输出的视频格式为NV12
    
    CGSize size = [ScreenRecorder fitSizeWithRealSize:[ScreenRecorder getScreenSizeWithDisplayId:displayId]];
    NSDictionary *videoSetting = [NSDictionary dictionaryWithObjectsAndKeys:
                                  [NSNumber numberWithInt:kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange], kCVPixelBufferPixelFormatTypeKey,
                                  @(size.width),kCVPixelBufferWidthKey,@(size.height),kCVPixelBufferHeightKey,
                                  nil];
    [_output setVideoSettings:videoSetting];
    // ---  设置输出串行队列和数据回调  ---
    dispatch_queue_t outputQueue = dispatch_queue_create("com.qixin.screen.recorder", DISPATCH_QUEUE_SERIAL);
    // ---  设置代理  ---
    [_output setSampleBufferDelegate:self queue:outputQueue];
    // ---  丢弃延迟的帧  ---
    _output.alwaysDiscardsLateVideoFrames = YES;
    
    
    
    if ([_session canAddInput:_input])
    {
        [_session addInput:_input];
        [_input setCapturesCursor:YES];
        [_input setMinFrameDuration:CMTimeMake(1, fps)];
    }
    
    if ([_session canAddOutput:_output])
    {
        [_session addOutput:_output];
    }
}
```

如果你需要录制音频的话,至少我们没有这个需求...

```objc
- (void)recordAudio
{
    if (!_audioInput) {
        AVCaptureDevice *audioDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
        _audioInput = [AVCaptureDeviceInput deviceInputWithDevice:audioDevice error:nil];

        if ([_session canAddInput:_audioInput]) {
            [_session addInput:_audioInput];
        }
    }
}
```


下面是一些附加的配置能力,但是我们基本都使用默认的即可.如果您有特殊需要可以设置


默认情况下，AVCaptureScreenInput 不会在其捕获的输出中突出显示鼠标点击。 如果此属性设置为 YES，则在捕获的输出中突出显示鼠标单击（在单击期间在鼠标周围绘制一个圆圈）。

```objc
- (void)setCapturesMouseClicks:(BOOL)capturesMouseClicks
{
    [_input setCapturesMouseClicks:capturesMouseClicks];
}
```

默认情况下，AVCaptureScreenInput 在其捕获的输出中绘制光标。 如果此属性设置为 NO，则捕获的输出仅包含屏幕上的窗口。

```objc
- (void)setCapturesCursor:(BOOL)capturesCursor
{
    [_input setCapturesCursor:capturesCursor];
}
```

默认情况下，AVCaptureScreenInput 捕获与其关联的 displayID 的整个区域。 要将捕获矩形限制为屏幕的一部分，请设置 cropRect 属性，该属性在屏幕坐标系中定义屏幕的较小部分。 原点 (0,0) 是屏幕的左下角。

```objc
- (void)setCropRect:(CGRect)cropRect
{
    [_input setCropRect:cropRect];
}
```

AVCaptureScreenInput 的 minFrameDuration 是其最大帧速率的倒数。 此属性可用于请求输入产生视频帧的最大帧速率。 由于总带宽的原因，请求的速率可能无法实现，因此实际帧速率可能会更低。

```objc
- (void)setFrameRate:(int)framerRate
{
    [_input setMinFrameDuration:CMTimeMake(1, (int32_t)framerRate)];
}
```

别忘了设置我们自己定义的回调Block,我们就是为了获取这个CMSampleBufferRef来的

```objc
// typedef void(^GetScreenSampleBuffer)(CMSampleBufferRef sampleBuffer);
- (void)outputSampleBuffer:(GetScreenSampleBuffer)block
{
    self.block = block;
}
```


实现start和stop方法

```objc
- (void)start
{
    [_session startRunning];
}

- (void)stop
{
    [_session stopRunning];
}
```

一些辅助方法,您可能需要获取所选屏幕的frame

```objc
+ (CGRect)getScreenFrameWithDisplayId:(CGDirectDisplayID)displayId
{
    __block CGRect frame = CGRectZero;
    [NSScreen.screens enumerateObjectsUsingBlock:^(NSScreen * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        
        NSDictionary *description = [obj deviceDescription];
        CGDirectDisplayID curId = [[description objectForKey:@"NSScreenNumber"] unsignedIntValue];
        if (curId==displayId)
        {
            frame = obj.frame;
            *stop = YES;
        }
        
    }];
    
    return frame;
}
```