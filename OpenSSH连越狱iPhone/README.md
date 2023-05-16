# SSH连你的越狱iPhone

### 使用流程

* 在Cydia中安装OpenSSH包

* 在系统设置/无线局域网(WiFi)中找到你当前的WiFi,查看当前ip

> 比如:192.168.0.100

* 打开电脑控制台, 输入`ssh root@192.168.0.100` (假设你刚才查到的ip是这个)

*  成功连接


### 其他

在首次连接越狱的 iPhone 或 iPad 时，SSH 会提示您验证远程主机的指纹。指纹是一个用于验证远程主机身份的唯一标识符，可以防止中间人攻击和其他安全威胁。

当您首次连接越狱的 iPhone 或 iPad 时，SSH 会显示远程主机的指纹，如下所示：
```shell
The authenticity of host '192.168.0.100 (192.168.0.100)' can't be established.
ECDSA key fingerprint is SHA256:ABCDEFGHIJKLM0123456789NOPQRSTUVWXYZ.
Are you sure you want to continue connecting (yes/no)?
```

其中，"ECDSA key fingerprint" 是远程主机的指纹，"SHA256:ABCDEFGHIJKLM0123456789NOPQRSTUVWXYZ" 是指纹的具体值。

您需要验证指纹是否正确，以确保您连接的是正确的远程主机。

如果指纹正确，输入 "yes" 并按下回车键，SSH 将会将指纹保存到您的 ~/.ssh/known_hosts 文件中，以便以后的连接时自动验证远程主机的身份。