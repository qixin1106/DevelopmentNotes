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

## ffplay拉流

* `ffplay "rtmp://localhost:1935/live/room live=1"`

> live=1 这意味着 ffplay 将尝试从 RTMP 服务器实时接收流数据。

> 除了 live 参数外，RTMP 流地址还可以包含其他参数，例如：
> * timeout：指定 RTMP 流连接的超时时间，单位为秒。
> * buffer：指定 RTMP 流的缓冲区大小，单位为字节。
> * start：指定 RTMP 流的开始时间，格式为 hh:mm:ss。
> * stop：指定 RTMP 流的结束时间，格式为 hh:mm:ss。

> 例如:```rtmp://localhost:1935/live/room live=1 timeout=10 buffer=1024 start=00:01:30 stop=00:02:00``` 

> 局域网内其他设备拉流时,请将localhost改为服务设备的ip 



## ffmpeg推流(HLS版)

还记得之前配置hls吗? 这里可以测试一下.

* `ffmpeg -re -i /Users/Qixin/Downloads/111.mp4 -c:v libx264 -c:a aac -strict -2 -f flv rtmp://localhost:1935/hls/room`

> 将最后的地址应用名改为`hls`, 这对应着你conf文件中的配置.


## 浏览器拉流

打开浏览器输入

`http://localhost:8080/hls/room.m3u8`

> 局域网内其他设备拉流时,请将localhost改为服务设备的ip 


## ffmpeg推流(摄像头)

使用以下命令查看摄像头设备的名称和参数 

`ffmpeg -f avfoundation -list_devices true -i ""`

执行上述命令后，会输出系统中可用的音视频设备列表，包括摄像头、麦克风、屏幕等。摄像头设备的名称通常为 default 或 0。

```
[AVFoundation indev @ 0x7f8a7d517640] AVFoundation video devices:
[AVFoundation indev @ 0x7f8a7d517640] [0] FaceTime HD Camera (Built-in)
[AVFoundation indev @ 0x7f8a7d517640] [1] USB Camera
[AVFoundation indev @ 0x7f8a7d517640] [2] Capture screen 0
[AVFoundation indev @ 0x7f8a7d517640] AVFoundation audio devices:
[AVFoundation indev @ 0x7f8a7d517640] [0] Unknown USB Audio Device
[AVFoundation indev @ 0x7f8a7d517640] [1] Built-in Microphone
```

`ffmpeg -f avfoundation -framerate 30 -video_size 640x480 -i "0" -c:v libx264 -preset ultrafast -tune zerolatency -f flv rtmp://localhost:1935/live/room`


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
