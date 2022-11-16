# PJSIP for macOS/iOS

### 前言

这里对前几年做的app软电话功能的一个简单梳理.自从公司被收购之后,因为新主体已经有同类软件了,所以这个就被下架了,我也就不藏着掖着了.

#### 关于PJSIP

[PJSIP网址](https://www.pjsip.org)

PJSIP 是一个免费的开源多媒体通信库，用 C 语言编写，实现基于标准的协议，如 SIP、SDP、RTP、STUN、TURN 和 ICE。 它将信令协议 (SIP) 与丰富的多媒体框架和 NAT 穿越功能结合到高级 API 中，这种 API 是可移植的，适用于几乎任何类型的系统，从台式机、嵌入式系统到手机。

PJSIP 既紧凑又功能丰富。 它支持音频、视频、状态和即时消息，并具有大量文档。 PJSIP 非常便携。 在移动设备上，它抽象了系统相关的功能，并且在许多情况下能够利用设备的本机多媒体功能。

自 2005 年以来，PJSIP 由一个专门为该项目工作的小团队开发，有来自世界各地的数百名开发人员参与，并自 2007 年以来在 SIP 互操作性活动 (SIPit) 上进行例行测试。

#### 我们要做什么?

那是几年前,一个阳光明媚的下午,产品会上正激烈的讨论,要实现app语音通话,不仅要支持app内软通话,还要可以给座机和手机打电话......balabala......

咳~!

说回正题,我们在调研了PJSIP和Linphone后,最终选择了PJSIP.

### 编译库

[我用的编译配置,Github](https://github.com/nesterenkodm/pjsip)

当然这个配置是别人写好的编译脚本,如果自己使用的话,还需要根据情况,自行修改一下.

编译完成后会生成以下几个库

- pjmedia

> PJMEDIA 是一个功能齐全的媒体堆栈，在开源/GPL 条款下分发，具有占用空间小、良好的可扩展性和出色的可移植性。 以下是 PJMEDIA 优势的一些简要概述。

- pjsip
    - pjsua

> PJSIP 是一个开源 SIP 协议栈，旨在占用空间非常小，具有高性能且非常灵活。
> PJSUA 用于构建 SIP UA 应用程序的非常高级的 API。
> 通常我们使用PJSUA这些API来开发,除非您希望修改源码,并做到更深层的定制化,或者解决一些bug。

- pjlib

> PJLIB 是一个用 C 编写的开源小型框架库，用于制作可扩展的应用程序。 由于占用空间小，它可以用于嵌入式应用程序（我们希望如此！），但该库还旨在促进高性能协议栈的创建。

- pjlib-util

> PJLIB-UTIL 是一个为 PJLIB 提供附加功能的辅助库。

- third_patry

> 第三方库需要使用的

- pjnath

> PJNATH（PJSIP NAT Helper）是一个开源库，通过使用基于标准的协议（例如 STUN、TURN 和 ICE）提供 NAT 遍历功能。

- opus

> Opus 是一种完全开放、免版税、用途广泛的音频编解码器。 Opus 在 Internet 上的交互式语音和音乐传输方面无与伦比，但也适用于存储和流媒体应用。 它被互联网工程任务组 (IETF) 标准化为 RFC 6716，它结合了 Skype 的 SILK 编解码器和 Xiph.Org 的 CELT 编解码器的技术。

- openssl

> 用于通用加密和安全通信

### 开发-工作流程

以下代码只说明一些核心的功能,不可能全部完整的列举出来.

#### 获取配置

首要要获取需要的一些配置,这个由后台接口返回,我们的app由后台分配权限,如果user拥有语音通话的权限,则会下发PJSIP所需要的配置.

```objc
// 地址和端口
NSString *tcphost;
int tcpport;        
// 你的账号
NSString *account;
// 你的密码
NSString *password;
// 你的账号所分配的key
NSString *ffskey;
// 编码优先级控制
NSString *ffscodec;
// 是否启动TLS
BOOL ffstls;
// 支持TLS的地址和端口
NSString *tlshost;
int tlsport;

// RTP类型
typedef NS_ENUM(int, FFSRTP) {
    FFSRTP_RTP = 0,
    FFSRTP_SRTP = 1,
    FFSRTP_ZRTP = 2,
};
FFSRTP ffsrtp;

// 是否启动选项
typedef NS_ENUM(int, SRTP_USE) {
    SRTP_USE_DISABLE = 0,   //禁用
    SRTP_USE_OPTIONAL = 1,  //可选
    SRTP_USE_MANDATORY = 2, //强制
};
SRTP_SIGN srtpsign;

/*
* 0: SRTP 不需要安全信令
* 1: SRTP 需要 TLS 等安全传输
* 2: SRTP 需要安全的端到端传输 (SIPS)
*/
  typedef NS_ENUM(int, SRTP_SIGN) {
    SRTP_SIGN_NOT_REQUIRE = 0,
    SRTP_SIGN_TLS = 1,
    SRTP_SIGN_END_TO_END = 2,
};
SRTP_USE srtprule;
```
获取后我们保存到本地的数据库,之后每次同步配置的时候,会刷新本地的配置.方便后台方便控制.

#### 初始化运行PJSIP

- 创建

```objc
pj_status_t status = pjsua_create();
if (status != PJ_SUCCESS) {
    return;
}
```

- 各种配置
    - pjsua_config
    - pjsua_media_config
    - pjsua_logging_config

```objc
// 创建一个默认的配置
pjsua_config cfg;
pjsua_config_default(&cfg);
// 最大呼叫支持（默认值：4）。 
// 此处指定的值必须小于编译时最大设置 PJSUA_MAX_CALLS，默认情况下为 32。
// 要增加此限制，必须使用新的 PJSUA_MAX_CALLS 值重新编译库。
cfg.max_calls = PJSUA_MAX_CALLS;

// 配置useragent,主要用于用户身份识别,我已将规则隐去.脱敏.
NSString *secret = 按规则拼接
NSString *userAgent = 按规则拼接
cfg.user_agent = pj_str((char*)userAgent.UTF8String);

// 我们使用的是SRTP,线上版本开启,测试环境不需要.
if (self.ffsrtp == FFSRTP_SRTP) {
    cfg.use_srtp = self.srtprule;
    cfg.srtp_secure_signaling = self.srtpsign;
}


// 回调函数配置
cfg.cb.on_call_media_state = &on_call_media_state; // 媒体状态回调（通话建立后，要播放RTP流）
cfg.cb.on_call_state = &on_call_state; // 电话状态回调
cfg.cb.on_call_tsx_state = &on_call_tsx_state;
cfg.cb.on_reg_state = &on_reg_state;//注册状态回调
cfg.cb.on_incoming_call = &on_incoming_call;//来电通知
cfg.cb.on_call_transfer_status = &on_call_transfer_status;//传输通道状态
cfg.cb.on_call_media_event = &on_call_media_event;//
cfg.cb.on_media_event = &on_media_event;
cfg.cb.on_create_media_transport = &on_create_media_transport;


// 媒体相关配置
pjsua_media_config media_cfg;
pjsua_media_config_default(&media_cfg);
media_cfg.clock_rate = 16000;
media_cfg.snd_clock_rate = 16000;
media_cfg.ec_tail_len = PJSUA_DEFAULT_EC_TAIL_LEN;
media_cfg.no_vad = PJ_TRUE;
media_cfg.ec_options = PJMEDIA_ECHO_WEBRTC|PJMEDIA_ECHO_USE_NOISE_SUPPRESSOR|PJMEDIA_ECHO_AGGRESSIVENESS_AGGRESSIVE;
media_cfg.quality = 10;
media_cfg.on_aud_prev_play_frame = &on_aud_prev_play_frame;
media_cfg.on_aud_prev_rec_frame = &on_aud_prev_rec_frame;


// 日志相关配置
pjsua_logging_config log_cfg;
pjsua_logging_config_default(&log_cfg);

#ifdef DEBUG
log_cfg.msg_logging = PJ_TRUE;
log_cfg.console_level = 4;
log_cfg.level = 5;
#else
log_cfg.msg_logging = PJ_FALSE;
log_cfg.console_level = 0;
log_cfg.level = 0;
#endif
// 如果你需要自己处理log,可以实现这个
//log_cfg.cb = &log_callback;
```

- 初始化,将上一步的3个配置传入

```objc
status = pjsua_init(&cfg, &log_cfg, &media_cfg);
if (status != PJ_SUCCESS) {
    return;
}
```

- 传输配置

```objc
pjsua_transport_config cfg;
pjsua_transport_config_default(&cfg);
NSString *path = "你的pem证书"
cfg.tls_setting.ca_list_file = pj_str((char*)path.UTF8String);

// 传输类型配置
if (self.ffstls) {
    cfg.tls_setting.verify_server = PJ_TRUE;
    status = pjsua_transport_create(PJSIP_TRANSPORT_TLS, &cfg, NULL);
} else {
    cfg.tls_setting.verify_server = PJ_FALSE;
    status = pjsua_transport_create(PJSIP_TRANSPORT_TCP, &cfg, NULL);
}

if (status != PJ_SUCCESS) {
    return ;
}
```

- 启动

```objc
status = pjsua_start();
if (status != PJ_SUCCESS) {
    return;
}
```

#### 修改MediaCodec优先级

```objc
//记录的格式
NSString *ffscodec = 配置下发的参数;
// 将下发的配置转为数组,按优先级排列好顺序
NSArray *arrCodecPriority = [self codecPriorityWithCodec:ffscodec];


pjsua_codec_info codes[16] = {0};
unsigned int count = 16;
pjsua_enum_codecs(codes, &count);

// 遍历自定义的优先级数组
for (NSDictionary *dicPriority in arrCodecPriority) {
    
    // 取出后端配置的编码及对应的优先级,优先级最大为255,视为最高级
    NSString *codec = dicPriority[@"codec"];
    int priority = [dicPriority[@"priority"] intValue];
    
    for (int i = 0; i < count; i++) {
        pjsua_codec_info codec_info = codes[i];
        NSString *codec_id_string = [[NSString alloc] initWithBytes:codec_info.codec_id.ptr length:codec_info.codec_id.slen encoding:NSUTF8StringEncoding];
        
        // 如果配置中的编码,在当前设备中支持的话,则修改它的优先级.
        if (kIsStringValid(codec) && [codec.lowercaseString isEqualToString:codec_id_string.lowercaseString]) {
            //设置为优先级
            pj_str_t codec_id = codec_info.codec_id;
            pjsua_codec_set_priority(&codec_id, priority);
        }
    }
}
```
> 这里主要是因为,我们的硬话机支持的编码格式比较少,而app软电话支持的比较丰富,因此在软通话时,可以使用opus格式,语音质量好,抗干扰比较牛B.

到这里为止我们的PJSIP就初始化运行起来了,但是这只是基础的准备工作,目前它还不能接打电话,不过在这之前我们还有个重要的步骤,就是注册.

#### 注册/注销

注册是标准流程中比较重要的一环,因为后台如果想把两边的设备联通在一起,需要你在必须先在后台注册过,它会返回你一个acc_id.后面需要地方需要用到这个.

有人可能会疑惑,如果我设备未激活,比较iOS设备,app处于后台杀掉状态,怎么弄注册呢? 

实际上,这里我们需要用到iOS的VOIP服务,android据我了解,也是后台开了保活的服务监听.

*但是iOS在某个版本之后已经不允许使用VOIP推送通知来做单纯的后台激活了.*

VOIP必须要和CallKit绑定在一起使用,这里我们后来也确实对接了CallKit,如果你不对接CallKit,在几次异常之后,苹果VOIP推送会给你的app锁住,即你再也无法收到VOIP推送,我们当时的测试的解决办法是,删除app重新安装.当时是这样的.

macOS到是比较简单一些,因为同类的app并没有实现在app不启动的时候来做接听提示.一旦app启动了,各种监听,相对iOS平台来说还是更简单一些的.

废话不多说,上代码

##### 注册

```objc
pjsua_state state = pjsua_get_state();
if (state != PJSUA_STATE_RUNNING) {
    // 一些错误处理
    return;
}

NSString *account = 账号
NSString *password = 密码
NSString *host = [self currentHost];
NSInteger port = [self currentPort];

if (!kIsStringValid(account) ||
    !kIsStringValid(password) ||
    !kIsStringValid(host) ||
    port==0)
{
    // 一些错误处理
    return;
}


// 账号配置
pjsua_acc_config acc_config;
pjsua_acc_config_default(&acc_config);

//(版本号)<sip:(账号/号码)@(ip地址):(端口)>
// 这里版本号是我们自己加的,用来区分版本和平台.
acc_config.id = pj_str((char *)[NSString stringWithFormat:@"%@<sip:%@@%@>",[XPMacDevice currentDevice].appBuild,account,[NSString stringWithFormat:@"%@:%ld",host,(long)port]].UTF8String);

// 如果你使用了TLS
if (self.ffstls) {
    // transport=tls
    acc_config.reg_uri = pj_str((char *)[NSString stringWithFormat:@"sip:%@", [NSString stringWithFormat:@"%@:%ld;transport=tls",host,(long)port]].UTF8String);
} else {
    // transport=tcp
    acc_config.reg_uri = pj_str((char *)[NSString stringWithFormat:@"sip:%@", [NSString stringWithFormat:@"%@:%ld;transport=tcp",host,(long)port]].UTF8String);
}

// 这些配置看着比较简单,能直接读出意思
acc_config.reg_retry_interval = 3;
acc_config.reg_first_retry_interval = 3;
acc_config.reg_retry_random_interval = 0;
acc_config.reg_timeout = kTimeOut;

// 这里就是填写你账号的配置
acc_config.cred_count = 1;
acc_config.cred_info[0].realm = pj_str("*");
acc_config.cred_info[0].username = pj_str((char *)account.UTF8String);
acc_config.cred_info[0].data_type = PJSIP_CRED_DATA_PLAIN_PASSWD;
acc_config.cred_info[0].data = pj_str((char *)password.UTF8String);
// 设置当你添加账号时,自动注册,当然你也可以选择手动注册.
acc_config.register_on_acc_add = PJ_TRUE;

// 线上我们使用SRTP,这玩意跟后端配置走,但是也就是SRTP/RTP切换
if (self.ffsrtp == FFSRTP_SRTP) {
    // 这块直接取从后端下发的配置参数
    acc_config.use_srtp = self.srtprule;
    acc_config.srtp_secure_signaling = self.srtpsign;
}


// 先获取你缓存的acc_id,这块第一次可能没有,没有也没事,后面验证时会false,也有它自己的逻辑
pjsua_acc_id acc_id = 你的acc_id;
pj_bool_t is_valid = pjsua_acc_is_valid(acc_id);


// 如果有效的话,说明你之前可能注册过,但是也许会过期.
// 举个🌰,比如你程序直接后台杀掉了,这时服务器端并不知道你注册的账号已经无效了.
// 那么当你下次再想使用的时候,你需要重新刷新.
// 既然你调用的方法是注册,那么我们可以先删除之前的账号,再重新注册,相当于刷新了
if (is_valid) {
    pj_status_t status = pjsua_acc_del(acc_id);
    if (status != PJ_SUCCESS) {
        // 一些错误处理
    }
}

// 重新注册
pj_status_t status = pjsua_acc_add(&acc_config, PJ_TRUE, &acc_id);

if (status != PJ_SUCCESS) {
    // 一些错误处理
}
```

以上就是注册里的一些流程Coding,这是你主动发起了注册,那么还记得之前初始化时,你注册了一堆的回调吗?

```objc
cfg.cb.on_reg_state = &on_reg_state;//注册状态回调
```

其中有一个是注册回调,实现如下


```objc
 static void on_reg_state(pjsua_acc_id acc_id) {
    pjsua_acc_info acc_info;
    pjsua_acc_get_info(acc_id, &acc_info);
    if (acc_info.status == PJSIP_SC_OK) {
        // 这里需要你维护一下acc_id,因为很多操作都需要这个.
    } else {
        // 一些错误处理...
    }
}
```
以上就是注册的流程了,实际使用中还有很多细节处理,这里不再多述.


##### 注销

```objc
pjsua_state state = pjsua_get_state();
if (state != PJSUA_STATE_RUNNING) {
    // 一些错误处理
    return;
}

pjsua_acc_id acc_id = 你的acc_id;
pj_bool_t is_valid = pjsua_acc_is_valid(acc_id);
if (is_valid) {
    pj_status_t status = pjsua_acc_del(acc_id);
    if (status != PJ_SUCCESS) {
        // 一些错误处理
    }
}
```


#### 拨打电话

因为前面初始化,启动,注册都确保完成了,那么我们可以尝试去主动拨打一个电话试试看.

上代码:

```objc
pjsua_state state = pjsua_get_state();
if (state != PJSUA_STATE_RUNNING) {
    // 一些错误处理
    return;
}

pjsua_acc_id acc_id = 你的acc_id;
pj_bool_t is_valid = pjsua_acc_is_valid(acc_id);
// 验证你的账号
if (!is_valid) {
    // 一些错误处理
    return;
}


NSString *host = [self currentHost];
NSInteger port = [self currentPort];
NSString *targetUri = nil;
// 线上环境走tls
if (self.ffstls) {
    targetUri = [NSString stringWithFormat:@"sip:%@@%@:%ld;transport=tls", target, host, (long)port];
} else {
    targetUri = [NSString stringWithFormat:@"sip:%@@%@:%ld;transport=tcp", target, host, (long)port];
}


// 要放在 To 标头中的 URI（通常与目标 URI 相同）
pj_status_t status;
pj_str_t dest_uri = pj_str((char *)targetUri.UTF8String);
// 配置
pjsua_call_setting opt;
pjsua_call_setting_default(&opt);
// 这个是0意思是只有语音,1是带视频的通话,这里我们只用语音
opt.vid_cnt = 0;

// 这里函数就是呼叫
status = pjsua_call_make_call(0, &dest_uri, &opt, NULL, NULL, NULL);

if (status != PJ_SUCCESS) {
    // 一些错误处理
    return;
}
```

应该已经可以外呼了.不过具体规则还是要看后端的配置.

> 这里有一个通话的状态回调函数,比如你如何知道你电话已拨打...或者拨通正在等待对方接听...又或者对方接听了,这些都涉及到你的UI状态的变化,因为拨打和接听类似,所以我准备在后面统一去介绍.

#### dial_dtmf

你平时打一些语音客服电话时,你会发现在通话中,它让你去按一些数字或者符合(*/#),这又是怎么搞的呢,其实PJSIP里是有这方面支持的.

上代码:

```objc
pjsua_state state = pjsua_get_state();
if (state != PJSUA_STATE_RUNNING) {
    // 一些错误处理
    return;
}

pj_bool_t is_active = pjsua_call_is_active(你的call_id);
if (!is_active) {
    // 一些错误处理
    return;
}

// number就是你外面输入的内容,你应该控制它,确保一次是一个字符
// 对一些全角的处理
if ([number isEqualToString:@"﹡"]) {
    number = @"*";
} else if([number isEqualToString:@"＃"]) {
    number = @"#";
}

pj_str_t digits;
pj_status_t status;

digits = pj_str((char *)number.UTF8String);
// 这个函数就是输入
status = pjsua_call_dial_dtmf(你的call_id, &digits);
if (status != PJ_SUCCESS) {
    // 一些错误处理
    return;
}
```

### 接听电话

接听电话,首先要确保你的PJSIP已经运行了,并且注册上了,正常情况下,如果你没有注册,你的incoming回调是无法收到消息的,好了

假定我们之前我们流程一切正常,就算没有启动程序(iOS设备),你也已经使用VOIP方式叫它后台启动了,然后初始化,启动,注册,一切都OK👌的话.

你应该还记的你之前注册过的函数吧

```objc
cfg.cb.on_incoming_call = &on_incoming_call;//来电通知
```

来看下如何处理on_incoming_call函数

```objc
static void on_incoming_call(pjsua_acc_id acc_id,
                             pjsua_call_id call_id,
                             pjsip_rx_data *rdata) {
    //来电通知
    pjsua_call_info call_info;
    pjsua_call_get_info(call_id, &call_info);
    
    NSString *voipno = [NSString stringWithFormat:@"%s",call_info.remote_info.ptr];
    // 这里是我们自己定义的方法,从<sip:xxx@xxxx>中获取到我们需要的账号部分
    voipno = [XYPPApi getVoipNoForURI:voipno];
    
    // 如果当前你正在同话中,那么选择挂断这个来电,理由是忙,对方会听到忙音
    if ([XYPPApi shared].isBusyHere) {
        // 你挂断的状态传入PJSIP_SC_BUSY_HERE(486)
        // 全部的状态码,你可以通过pjsip_status_code查看
        pjsua_call_hangup(call_id, PJSIP_SC_BUSY_HERE, NULL, NULL);
        return;
    }
    
    // 走到这里就是进入可以准备接听的状态了.
    pjsua_call_setting opt;
    pjsua_call_setting_default(&opt);
    // 这里之前介绍了0=纯语音,1带视频的
    opt.vid_cnt = 0;
    
    // 先answer让对方响铃,这个函数执行成功后,标识你已经通了
    // 对方会听到接通的"嘟嘟"声,打过电话的都知道吧.
    pj_status_t state = pjsua_call_answer2(call_id, &opt, PJSIP_SC_RINGING, NULL, NULL);
    
    // 此时你获取应该通知一下UI来做一些提示
    // 比如显示有人给你来电了,然后你可以提供一些决绝,接听,之类的功能按钮
}
```
> 你需要自己维护一下,本次通话的call_id,因为无论你是要接听,还是拒绝,都需要这个告知是操作哪个call_id

其实原先没有用CallKit的时候,我们这里本地也要加上铃音播放和震动提示,类似微信语音电话那种"噔,噔,噔,噔,噔".好在后面可以用CallKit,类似原生电话的体验,效果更牛B👍🏻

好了,如果假设你的UI上已经显示出来了.
那么我们通常有2个选择,接听或者拒绝


##### 同意接听
先说下接听吧,都很简单,只是把answer的状态传入PJSIP_SC_OK

```objc
pj_status_t status = pjsua_call_answer2(call_id, &opt, PJSIP_SC_OK, NULL, NULL);
```
##### 拒绝接听

决绝接听其实就是给他挂了,但是挂断原因的状态码是PJSIP_SC_DECLINE

```objc
pj_status_t status = pjsua_call_hangup(call_id, PJSIP_SC_DECLINE, NULL, NULL);
```

其实也可以告诉他你是在忙,这个完全看业务需求

```objc
pj_status_t status = pjsua_call_hangup(call_id, PJSIP_SC_BUSY_HERE, NULL, NULL);
```
眼熟吧,跟incoming里自动忙时一样的


### 关于IP切换问题

这个属于iOS手机中遇到的,因为有可能出现在这种场景,你正在打电话,并且从WiFi环境中离开,你的手机会切换到4G/5G状态,这时你的IP会改变,如果不做任何处理的话,显然你的通话会出现异常.

然后这个问题PJSIP给出了解决方案

```objc
pjsua_state state = pjsua_get_state();
if (state==PJSUA_STATE_RUNNING) {
    pjsua_ip_change_param param;
    pjsua_ip_change_param_default(&param);
    pjsua_handle_ip_change(&param);
}
```

我们一般会在网络状态变更时,来触发.你应该有办法来判断网络是否切换了


### 关于麦克风静音

这个是电话必备功能,但是可能不常用,也许在多人会议中经常使用,这个后面有时间我再总结一下,语音会议室,开发遇到的新鲜事.

```objc
- (void)mute:(BOOL)enable {
    if (enable) {
        // 静音
        _isMute = YES;
        pjsua_conf_adjust_rx_level(0, 0);
    } else {
        // 解除静音
        _isMute = NO;
        pjsua_conf_adjust_rx_level(0, 2);
    }
}
```


### 关于媒体状态回调

```objc
static void on_call_media_state(pjsua_call_id call_id) {
    //获取呼叫信息
    pjsua_call_info info;
    pjsua_call_get_info(call_id, &info);
    
    switch (info.media_status) {
        case PJSUA_CALL_MEDIA_NONE: {
            break;
        }
        case PJSUA_CALL_MEDIA_ACTIVE: {
            // 媒体激活时
            pjsua_set_snd_dev(PJSUA_SND_DEFAULT_CAPTURE_DEV,PJSUA_SND_DEFAULT_PLAYBACK_DEV);
            // 呼叫接通
            // 建立单向媒体流从源到汇
            pjsua_conf_connect(info.conf_slot, 0);
            pjsua_conf_connect(0, info.conf_slot);
                        
            // 这时,你可以设置成静音,不设置的话你就自己填一个
            // 1.0是正常, 2.0相当于增益了
            pjsua_conf_adjust_rx_level(0, 2.0);
            pjsua_conf_adjust_tx_level(0, 2.0);
            break;
        }
        case PJSUA_CALL_MEDIA_LOCAL_HOLD: {
            break;
        }
        case PJSUA_CALL_MEDIA_REMOTE_HOLD: {
            break;
        }
        case PJSUA_CALL_MEDIA_ERROR: {
            break;
        }
        default:
            break;
    }
}
```


### 关于通话状态回调

这些状态回调中,我们一般用来通知UI更新和做一些设置变更

```objc
static void on_call_state(pjsua_call_id call_id, pjsip_event *e) {
    // 通话状态:CALLING
    // 通话状态:EARLY
    // 通话状态:EARLY
    // 呼叫成功,等待对方接听
    // 通话状态:CONNECTING
    // 通话状态:CONFIRMED
    // DISCONNCTD  对方挂断
    
    //获取通话信息
    pjsua_call_info call_info;
    pjsua_call_get_info(call_id, &call_info);
    
    switch (call_info.state) {
        case PJSIP_INV_STATE_NULL: {
            break;
        }
        case PJSIP_INV_STATE_CALLING: {
            break;
        }
        case PJSIP_INV_STATE_INCOMING: {
            break;
        }
        case PJSIP_INV_STATE_EARLY: {
            break;
        }
        case PJSIP_INV_STATE_CONNECTING: {
            break;
        }
        case PJSIP_INV_STATE_CONFIRMED: {
            // 点击同意通话
            break;
        }
        case PJSIP_INV_STATE_DISCONNECTED: {
            // 挂断断开通知
            break;
        }
        default:
            break;
    }
}
```


### 未完可能待续