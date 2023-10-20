# MacOS本地搭建RTMP服务

## 安装

`brew tap denji/homebrew-nginx`
`brew install nginx-full --with-rtmp-module`


## 查看安装位置

`brew info nginx-full`

找到`nginx.conf`位置

```
The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.
```

## 修改nginx.conf

打开Finder `cmd + shift + g` 前往 `/usr/local/etc/nginx/nginx.conf` 路径


**# 在http{}块的外面配置rtmp{}块**

```
http {
......
}


rtmp {
    server {
        listen 1935;
        ping 30s;
        notify_method get;
        
        # RTMP支持
        application live {
            live on;
            record off;
            max_connections 1024;
        }
        
        # HLS支持
        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
            hls_fragment 10s;
        }
    }
}
```

**# 在http{}块中的server{}块中配置HLS**

```
#HLS配置
location /hls {
    # Serve HLS fragments
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        root /tmp;
        add_header Cache-Control no-cache;
}
```



## 启动nginx

* `nginx -t -c /usr/local/etc/nginx/nginx.conf` 让配置生效
* `nginx -s reload` 重启
* `nginx` 开启
* `nginx -s stop` 强制退出
* `nginx -s quit` 执行完任务退出

在浏览器中查看`http://localhost:8080`, 如果显示Welcome to nginx! 则说明配置成功.

## ffmpeg推流(本地文件版)

* `brew install ffmpeg` 安装

* `ffmpeg -re -i /Users/Qixin/Downloads/111.mp4 -c:v libx264 -c:a aac -strict -2 -f flv rtmp://localhost:1935/live/room`

> * -re：以实时模式运行 FFmpeg，按照视频的原始帧速率进行处理，并且不使用缓存。
* -i /Users/Qixin/Downloads/111.mp4：指定输入文件为 /Users/Qixin/Downloads/111.mp4，即将要转码的视频文件。
* -c:v libx264：指定视频编码器为 H.264 编码。
* -c:a aac：指定音频编码器为 AAC 编码。
* -strict -2：在使用 AAC 编码器时，指定使用非标准的 AAC 编码参数。
* -f flv：指定输出文件格式为 FLV 文件。
* rtmp://localhost:1935/live/room：指定输出 URL，即将转码后的视频流推送到 RTMP 服务器，其中 rtmp://localhost:1935/live 是 RTMP 服务器的地址和端口，room 是直播间的名称。

> > 该命令中使用了 -strict -2 参数，这是因为从 FFmpeg 3.0 版本开始，AAC 编码器默认使用了更严格的标准，不再支持一些非标准的编码参数，因此需要通过 -strict -2 参数指定非标准的编码参数。如果您使用的是较新版本的 FFmpeg，可以尝试不使用 -strict -2 参数，看看是否会有警告或错误提示。


## ffmpeg获取本机可用的设备信息

* `ffmpeg -f avfoundation -list_devices true -i ""`

> 能看到类似如下反馈信息,如有多个屏幕设备,则会显示多个

```js
[AVFoundation indev @ 0x7f8dedc11080] AVFoundation video devices:
[AVFoundation indev @ 0x7f8dedc11080] [0] FaceTime HD Camera (Built-in)
[AVFoundation indev @ 0x7f8dedc11080] [1] USB Camera
[AVFoundation indev @ 0x7f8dedc11080] [2] Capture screen 0
[AVFoundation indev @ 0x7f8dedc11080] AVFoundation audio devices:
[AVFoundation indev @ 0x7f8dedc11080] [0] Unknown USB Audio Device
[AVFoundation indev @ 0x7f8dedc11080] [1] Built-in Microphone
```

## ffmpeg推流(摄像头)

* `ffmpeg -f avfoundation -video_device_index 0 -i "0" -f avfoundation -audio_device_index 1 -i "1" -c:v h264_videotoolbox -pix_fmt yuv420p -preset ultrafast -tune zerolatency -f flv rtmp://localhost:1935/live/room`

> * -c:v 这里可以选择`h264_videotoolbox(硬)`或者`libx264(软)`


## ffmpeg推流(电脑屏幕)

 * `ffmpeg -f avfoundation -i "2:none" -s 1920x1080 -c:v h264_videotoolbox -b:v 6000k -pix_fmt yuv420p -preset ultrafast -tune zerolatency -f flv rtmp://localhost:1935/live/room`
 
> * -i <选择设备编号:none>, 
> * -s 1920x1080 表示分辨率
> * -b:v 6000k 设置比特率,相当于0.75 MB/s,属于高清范畴

> 这里的 "2" 是输入流的索引号，用于指定要播放的特定流。索引号从0开始，表示不同的流。例如，0表示第一个流，1表示第二个流，以此类推。

> 而 ":none" 是输入流的参数，用于指定不使用特定的流。在这种情况下，":none" 表示不使用任何特定的音频或视频流，即不播放任何音频或视频。

> 因此，"-i "2:none"" 的含义是从输入源中选择索引号为2的流，但不使用该流中的音频或视频内容进行播放。请注意，具体的索引号和可用的流取决于输入源的内容和格式。


### 以上参数较多可自行搭配使用

## ffplay拉流(不建议,延迟太高)

* `ffplay "rtmp://localhost:1935/live/room live=1"`

> live=1 这意味着 ffplay 将尝试从 RTMP 服务器实时接收流数据。

> 除了 live 参数外，RTMP 流地址还可以包含其他参数，例如：
> * timeout：指定 RTMP 流连接的超时时间，单位为秒。
> * buffer：指定 RTMP 流的缓冲区大小，单位为字节。
> * start：指定 RTMP 流的开始时间，格式为 hh:mm:ss。
> * stop：指定 RTMP 流的结束时间，格式为 hh:mm:ss。

> 例如:```rtmp://localhost:1935/live/room live=1 timeout=10 buffer=1024 start=00:01:30 stop=00:02:00``` 

> 局域网内其他设备拉流时,请将localhost改为服务设备的ip 

## ffplay拉流(低延迟,建议)

* `ffplay -fflags nobuffer -flags low_delay -max_delay 100 -sync ext -bufsize 1000 -i rtmp://localhost:1935/live/room`

> * -fflags nobuffer：禁用内部缓冲。默认情况下，FFplay 会使用内部缓冲来提供更平滑的播放体验。使用 -fflags nobuffer 参数可以禁用内部缓冲，减少播放延迟，但可能会导致播放不稳定或出现卡顿现象。
> * -flags low_delay：启用低延迟模式。该参数启用低延迟模式，减少解码器的缓冲，以便更快地将数据传递到显示器进行播放。这有助于减少播放延迟。
> * -max_delay <delay>：设置最大的延迟时间。该参数限制播放的总延迟时间。延迟超过指定的时间将被丢弃，以避免播放过多积压的数据。例如，-max_delay 100 将最大延迟设置为 100 毫秒。
> * -sync ext：使用外部时钟同步策略。默认情况下，FFplay 使用内部时钟来同步音视频数据，但这可能会导致一定的延迟。通过使用 -sync ext 参数，FFplay 将使用外部时钟同步策略，以降低音视频同步的延迟。

## ffmpeg推流(HLS版)

还记得之前配置hls吗? 这里可以测试一下.

* `ffmpeg -re -i /Users/Qixin/Downloads/111.mp4 -c:v libx264 -c:a aac -strict -2 -f flv rtmp://localhost:1935/hls/room`

> 将最后的地址应用名改为`hls`, 这对应着你conf文件中的配置.


## 浏览器拉流

打开浏览器输入

`http://localhost:8080/hls/room.m3u8`

> 局域网内其他设备拉流时,请将localhost改为服务设备的ip 

## 其他

### 安装nginx-full可能的错误

#### 本地冲突

```shell
Error: Cannot install denji/nginx/nginx-full because conflicting formulae are installed.
  nginx: because nginx-full symlink with the name for compatibility with nginx

Please `brew unlink nginx` before continuing.

Unlinking removes a formula's symlinks from /usr/local. You can
link the formula again after the install finishes. You can --force this
install, but the build may fail or cause obscure side effects in the
resulting software.
```

**提示说无法安装 denji/nginx/nginx-full 因为已经安装了另外一个 nginx ，而这两个 nginx 之间存在冲突。nginx-full 中的某些文件与 nginx 中的文件重名，因此安装会导致冲突。
为了解决这个问题，Homebrew 建议你先运行 `brew unlink nginx` 命令，这样就会解除 nginx 符号链接，从而避免与 nginx-full 产生冲突。**
