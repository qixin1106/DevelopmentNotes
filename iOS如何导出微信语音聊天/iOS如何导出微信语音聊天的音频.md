# iOS如何导出微信语音聊天的音频

## 1.迁移手机聊天记录到桌面端

> 如果桌面端微信已缓存语音消息，则可以直接跳过该步骤，进入步骤2

因为iOS客户端不方便去操作沙盒文件，我们需要先将iOS端的聊天数据，迁移到桌面端。

##### 迁移方式：

1. 查看语音消息所在的群聊，确定消息存在，记下聊天名字。

2. `我`/`设置`/`通用`/`聊天记录迁移与备份`/`迁移`/`迁移到电脑`/`迁移指定聊天记录`，选择你刚才找到的聊天。

3. 迁移成功后手机端，桌面端会出现提示。

## 2.打开桌面端沙盒文件

1. 找你需要导出语音消息的聊天，在其中发一个文件或者图片，或者找之前发过的文件图片

2. 右键选择`在Finder中打开`。

3. 此时会跳转到该聊天的微信沙盒目录。

4. 找到`Audio`文件夹

5. 此时可以查看到许多xxxx.aud.silk格式的文件，这个就是该聊天的语音文件了

## 3.将aud.silk转为mp3

1. 我们需要先github下载一个[工具脚本](https://github.com/kn007/silk-v3-decoder)

2. 确保本地安装过ffmpeg，可使用`brew install ffmpeg`进行安装（安装过程稍慢，看网速）如果是M系列芯片使用`arch -arm64 brew install ffmpeg`。

## 开始使用脚本进行转格式

#### 1.单独一个文件转

`sh converter.sh 33921FF3774A773BB193B6FD4AD7C33E.slk mp3`

> 其中33921FF3774A773BB193B6FD4AD7C33E.slk是要转换的文件，而mp3是最终转换后输出的格式。文件存在相同目录中

##### 贴一个我用的示例

```bash
sh /Users/qixin/Downloads/silk-v3-decoder-master/converter.sh /Users/qixin/Desktop/input/9864f70252742fe39a10.aud.silk mp3
```

#### 2.批量转

`sh converter.sh input ouput mp3`

> 注意：其中input是要转换的目录，而output是最终转换后音频输出的目录，最后的mp3参数是最终转换后输出的格式。你需要先把微信的.silk文件放到你指定的input文件夹中。

##### 贴一个我用的示例

```bash
sh /Users/qixin/Downloads/silk-v3-decoder-master/converter.sh /Users/qixin/Desktop/input /Users/qixin/Desktop/output mp3
```
