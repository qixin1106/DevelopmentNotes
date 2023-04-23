# 免密SSH iPhone

* iphone越狱
* Cydia安装openSSH

### 正常ssh登录iphone

* 在同一个局域网下
* 查看iphone的ip地址

```shell
ssh root@192.168.x.xxx
```

首次登录可能会提示你
```shell
RSA key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no)?
```

输入yes后,提示你输入密码

默认情况下密码是`alpine`

### 修改登录密码

```shell
# 修改root密码
passwd root
```
然后输入新密码,再次验证密码.

### 忘记密码

如果忘记了密码,你可以手动恢复一下.

打开手机端的/etc/master.passwd

可以使用iFile或者iTools等工具

```text
root:/smx7MYTQIi2M:0:0::0:0:System Administrator:/var/root:/bin/sh
```
> 在第7行找到root:开头的位置,到:0:0::0:0:前面夹着的部分,是你加密的密码

> 替换为`/smx7MYTQIi2M`,即改为原始默认密码`alpine`


### SSH免密登录

在macOS电脑中打开.ssh文件夹

`cd ~/.ssh`

查看里面是否存在以下3个文件
- id_rsa		

> 在macOS系统中，id_rsa是SSH密钥对中的私钥文件，用于身份验证和加密通信。它通常存储在用户主目录下的.ssh文件夹中。如果您需要使用SSH连接到远程服务器，则需要将此文件复制到目标服务器上的.ssh文件夹中。

- id_rsa.pub	

> id_rsa.pub是SSH公钥文件，是SSH密钥对中的公钥部分，用于在SSH协议中进行身份验证。

- known_hosts

> 在macOS系统中，~/.ssh文件夹下的known_hosts文件是一个存储用户访问的主机的公钥的文件。这个文件通过将用户的身份保存到本地系统来确保用户连接到合法的服务器，并有助于避免中间人攻击。当你通过SSH连接到一个新的远程服务器时，系统会提示你是否要将远程主机添加到known_hosts文件。

如果没有,则需要创建

`ssh-keygen -t rsa -b 1024`

在iPhone中的/var/root目录下检查是否存在.ssh文件夹,如果没有则需要创建

```shell
cd /var/root
mkdir .ssh
```

然后拷贝电脑中的公钥到手机

```shell
scp id_rsa.pub root@192.168.x.xxx:/var/root/.ssh
```

然后登录到iPhone中
```shell
cd /var/root/.ssh

touch authorized_keys

cat id_rsa.pub >> authorized_keys
```

好了,下面重新ssh登录手机试试,可以免密登录了.