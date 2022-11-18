# RTMP for macOS

## 前言

作者之前参与了一个会议共享桌面的项目,这个是多人语音会议的一个子功能模块.
主要是通过屏幕录制+推拉流来实现.

[RTMP库](https://github.com/ossrs/librtmp/tree/master/librtmp)

实时消息协议（英语：Real-Time Messaging Protocol，缩写RTMP）也称实时消息传输协议，是最初由Macromedia为通过互联网在Flash播放器与一个服务器之间传输流媒体音频、视频和数据而开发的一个专有协议。Macromedia后被Adobe Systems收购，该协议也已发布了不完整的规范供公众使用。

具体如rtmp协议的说明,可以在网上搜索一下,有不少文章介绍的很详细.

以下作者只是从macOS项目实战角度出发描述一下代码流程.用来记录.

## 关于推流

### 初始化连接

初始化代码如下,硬汉直接读代码
最好自己管理一下线程,因为RTMP_ConnectStream是阻塞式的,具体可看下源码.

```objc
// 头文件
#import "rtmp.h"

// 初始化
RTMP *push = RTMP_Alloc();
RTMP_Init(push);

// 设置超时
pushRTMP->Link.timeout = Timeout;

// 设置推流地址
int ret = RTMP_SetupURL(push, (char*)[@"你的推流地址" cStringUsingEncoding:NSASCIIStringEncoding]);
if (ret < 0) {
    // 错误处理
    return;
}

//设置可写，即发布流，这个函数必须在连接前使用，否则无效
RTMP_EnableWrite(push);

// 连接服务器
if (RTMP_Connect(push, NULL) < 0) {
    // 失败处理
    return;
}
// 连接流
if (RTMP_ConnectStream(push, 0) < 0) {
    // 失败处理
    return;
}
```

最好在合适的时候做一下检查.如果都断开了,就别推了.

```objc
if (!RTMP_IsConnected(pushRTMP)) {
    // 错误处理
    return;
}
```


### 关闭销毁

```objc
RTMP_Close(push);
RTMP_Free(push);
```

### MetaData

连接成功以后,在你发送数据流之前的第一步,要先发送MetaData,告知服务器接下来你要传递的数据的一些基本信息.

先准备好一些宏和常量,这些后面创建数据包会用到,都是给Meta用的

```objc
#define RTMP_HEAD_SIZE (sizeof(RTMPPacket) + RTMP_MAX_HEADER_SIZE)
#define SAVC(x)    static const AVal av_##x = AVC(#x)
static const AVal av_setDataFrame = AVC("@setDataFrame");
SAVC(onMetaData);
SAVC(duration);
SAVC(width);
SAVC(height);
SAVC(videocodecid);
SAVC(videodatarate);
SAVC(framerate);
SAVC(audiocodecid);
SAVC(audiodatarate);
SAVC(audiosamplerate);
SAVC(audiosamplesize);
SAVC(audiochannels);
SAVC(stereo);
SAVC(encoder);
SAVC(av_stereo);
SAVC(fileSize);
SAVC(avc1);
SAVC(mp4a);
static const AVal av_SDKVersion = AVC("你的版本标识");

```

```objc
// 我们先记录一下开始推送的时间,这个后面正式推送数据时会用到
self.start_time = RTMP_GetTime();

// 创建数据包
RTMPPacket packet;
char pbuf[2048], *pend = pbuf + sizeof(pbuf);
packet.m_nChannel = 0x03;     // control channel (invoke)
packet.m_headerType = RTMP_PACKET_SIZE_LARGE;
packet.m_packetType = RTMP_PACKET_TYPE_INFO;
packet.m_nTimeStamp = 0;
packet.m_nInfoField2 = pushRTMP->m_stream_id;
packet.m_hasAbsTimestamp = TRUE;
packet.m_body = pbuf + RTMP_MAX_HEADER_SIZE;

// 写入body内容
char *enc = packet.m_body;
enc = AMF_EncodeString(enc, pend, &av_setDataFrame);
enc = AMF_EncodeString(enc, pend, &av_onMetaData);

*enc++ = AMF_OBJECT;

// 默认0
enc = AMF_EncodeNamedNumber(enc, pend, &av_duration,        0.0);
enc = AMF_EncodeNamedNumber(enc, pend, &av_fileSize,        0.0);

// videosize
enc = AMF_EncodeNamedNumber(enc, pend, &av_width,           width);
enc = AMF_EncodeNamedNumber(enc, pend, &av_height,          height);

// video
enc = AMF_EncodeNamedString(enc, pend, &av_videocodecid,    &av_avc1);

// 宽x高
enc = AMF_EncodeNamedNumber(enc, pend, &av_videodatarate,   width * height  / 1000.f *1.5f);

// fps
enc = AMF_EncodeNamedNumber(enc, pend, &av_framerate,       fps);

//    // audio
// 因为我们做共享桌面,走的PJSIP音频,所以这里暂时不用.
//    enc = AMF_EncodeNamedString(enc, pend, &av_audiocodecid,    &av_mp4a);
//    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiodatarate,   96000);
//
//    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiosamplerate, 44100);
//    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiosamplesize, 16.0);
//    enc = AMF_EncodeNamedBoolean(enc, pend, &av_stereo,     NO);

// sdk version
enc = AMF_EncodeNamedString(enc, pend, &av_encoder,         &av_SDKVersion);

*enc++ = 0;
*enc++ = 0;
*enc++ = AMF_OBJECT_END;

packet.m_nBodySize = (uint32_t)(enc - packet.m_body);

RTMP_SendPacket(push, &packet, FALSE);
```

### SPS,PPS

发送sps和pps数据

```objc
char *sps = (char *)spsData.bytes;
int spsLen = (int)spsData.length;
char *pps = (char *)ppsData.bytes;
int ppsLen = (int)ppsData.length;


int bodySize = spsLen + ppsLen + 16;
RTMPPacket rtmpPacket;
RTMPPacket_Alloc(&rtmpPacket, bodySize);
RTMPPacket_Reset(&rtmpPacket);

rtmpPacket.m_packetType = RTMP_PACKET_TYPE_VIDEO;
rtmpPacket.m_nBodySize = bodySize;
rtmpPacket.m_nTimeStamp = 0;
rtmpPacket.m_hasAbsTimestamp = 0;
rtmpPacket.m_nChannel = 0x04;//音频或者视频
rtmpPacket.m_headerType = RTMP_PACKET_SIZE_MEDIUM;
rtmpPacket.m_nInfoField2 = pushRTMP->m_stream_id;

char *body = rtmpPacket.m_body;

int i = 0;
//frame type(4bit)和CodecId(4bit)合成一个字节(byte)
//frame type 关键帧1  非关键帧2
//CodecId  7表示avc
body[i++] = 0x17;

//fixed 4byte
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;

//configurationVersion： 版本 1byte
body[i++] = 0x01;

//AVCProfileIndication：Profile 1byte  sps[1]
body[i++] = sps[1];

//compatibility：  兼容性 1byte  sps[2]
body[i++] = sps[2];

//AVCLevelIndication： ProfileLevel 1byte  sps[3]
body[i++] = sps[3];

//lengthSizeMinusOne： 包长数据所使用的字节数  1byte
body[i++] = 0xff;

//sps个数 1byte
body[i++] = 0xe1;

//sps长度 2byte
body[i++] = (spsLen >> 8) & 0xff;
body[i++] = spsLen & 0xff;

//sps data 内容
memcpy(&body[i], sps, spsLen);
i += spsLen;

//pps个数 1byte
body[i++] = 0x01;

//pps长度 2byte
body[i++] = (ppsLen >> 8) & 0xff;
body[i++] = ppsLen & 0xff;

//pps data 内容
memcpy(&body[i], pps, ppsLen);

RTMP_SendPacket(push, &rtmpPacket, FALSE);
RTMPPacket_Free(&rtmpPacket);
```


### Video data

发送视频数据

```objc
// data就是你传入的I帧和P帧数据,这里因为我们是做实时,所以没有B帧
int dataLen = (int)data.length;
int bodySize = dataLen + 9;

RTMPPacket rtmpPacket;
RTMPPacket_Alloc(&rtmpPacket, bodySize);
RTMPPacket_Reset(&rtmpPacket);

char *body = rtmpPacket.m_body;

int i = 0;
//frame type(4bit)和CodecId(4bit)合成一个字节(byte)
//frame type 关键帧1  非关键帧2
//CodecId  7表示avc
if (isKeyFrame) {
    body[i++] = 0x17;
} else {
    body[i++] = 0x27;
}

// fixed 4byte   0x01表示NALU单元
body[i++] = 0x01;
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;

// dataLen  4byte
body[i++] = (dataLen >> 24) & 0xff;
body[i++] = (dataLen >> 16) & 0xff;
body[i++] = (dataLen >> 8) & 0xff;
body[i++] = dataLen & 0xff;

// data
memcpy(&body[i], data.bytes, dataLen);

rtmpPacket.m_packetType = RTMP_PACKET_TYPE_VIDEO;
rtmpPacket.m_nBodySize = bodySize;

// 持续播放时间,还记的Meta的时候我们记录了一个时间吗?
// 这里用到了,主要是用作一个时间对齐.
uint32_t ts = RTMP_GetTime() - self.start_time;
rtmpPacket.m_nTimeStamp = ts;

// 进入直播播放开始时间
rtmpPacket.m_hasAbsTimestamp = 0;
rtmpPacket.m_nChannel = 0x04;   // 音频或者视频
rtmpPacket.m_headerType = RTMP_PACKET_SIZE_LARGE;
rtmpPacket.m_nInfoField2 = pushRTMP->m_stream_id;

RTMP_SendPacket(push, &rtmpPacket, FALSE);
RTMPPacket_Free(&rtmpPacket);
```



## 关于拉流

### 初始化

与推流一致,唯一区别就是不需要启动写 "RTMP_EnableWrite()"

### 拉流

这里主要是通过你自己控制的一个循环,去不停地拉流

我们在这里会获取到3个回调数据

- i帧数据
- sps+pps数据
- p帧数据

```objc
RTMPPacket packet = { 0 };
// ret 正常是返回1,如果不是1的话,可能要考虑跳出你的循环.
ret = RTMP_ReadPacket(pull, &packet);

if (RTMPPacket_IsReady(&packet)) {
    
    RTMP_ClientPacket(pull, &packet);
    
    if (packet.m_packetType == RTMP_PACKET_TYPE_INFO) {
        // 这里解析Meta
        NSData *d = [NSData dataWithBytes:packet.m_body length:packet.m_nBodySize];
        AMFObject obj;
        AMF_Decode(&obj, d.bytes, (int)d.length, FALSE);
        
    } else if (packet.m_packetType == RTMP_PACKET_TYPE_VIDEO) {
        
        // 这里获取视频数据
        NSData *d = [NSData dataWithBytes:packet.m_body length:packet.m_nBodySize];
        // 关键帧
        if (packet.m_body[0] == 0x17) {
            Byte type = packet.m_body[1];
            if (type==0) {
                // sps/pps,这里回调出去
                [self getSPSAndPPSDataWithData:d finish:^(NSData *sps, NSData *pps) {
                    
                    // 回调sps和pps数据...
                
                }];
            } else {
                // 视频数据,这里回调出去
                NSData *ivideo = [self getVideoWithData:d];
                
                // 回调i帧数据...
            }
        }
        // 非关键帧
        if (packet.m_body[0] == 0x27) {
            // 获取到非关键帧这里回调出去
            NSData *pvideo = [self getVideoWithData:d];
            
            // 回调p帧数据...
        }
    }
}
RTMPPacket_Free(&packet);
```

这里几个辅助方法,主要是获取去掉头,只要纯数据部分

```objc
typedef void(^GetSPSAndPPSData)(NSData *sps, NSData *pps);
- (void)getSPSAndPPSDataWithData:(NSData*)data finish:(GetSPSAndPPSData)finish {
    short spsLen = 0;
    [data getBytes:&spsLen range:NSMakeRange(11, 2)];
    HTONS(spsLen);
    NSData *spsData = [data subdataWithRange:NSMakeRange(13, spsLen)];
    
    short ppsLen = 0;
    [data getBytes:&ppsLen range:NSMakeRange(13+spsLen+1, 2)];
    HTONS(ppsLen);
    NSData *ppsData = [data subdataWithRange:NSMakeRange(13+spsLen+1+2, ppsLen)];
    
    if (finish) {
        finish(spsData,ppsData);
    }
}

- (NSData *)getVideoWithData:(NSData*)data {
    return [data subdataWithRange:NSMakeRange(5, data.length-5)];
}
```

当你拿到3个数据时,后面就是渲染播放的问题了

## 未完可能待续