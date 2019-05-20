## ssh命令

**登录类型**
1. 密码登录： 服务器发送公钥给客户端，客户端使用公钥加密后回传给服务器，服务器解密验证密码。
2. 公钥登录： 服务器发送一个随机字符串给客户端，客户端用私钥加密，服务器用公钥解密（rsa作为签名使用）

**ssh命令相关参数**
1. -A 密钥转发
2. -i 指定密钥文件
3. -p 端口号
4. -C：请求压缩所有数据；
5. -f 后台运行
1. -N 参数： 不要求分配shell，有些场景下ssh禁止账号请求shell终端，比如这个账号只是作为转发
2. -T ：不要求分配终端
3. -o ServerAliveInterval=60 隔段时间发送保活消息

**相关的命令**
1. ssh-keygen 用于生成密钥对
2. ssh-copy-id 用于复制公钥到服务器
> 复制公钥也可以使用：ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub


**相关的文件** 
1. ~/.ssh/authorized_keys  用于保存用户的公钥文件
2. ~/.ssh/known_hosts文件  保存的服务器用于辨别服务器的唯一散列码
3. ~/.ssh/id_dsa 用户的私钥文件
4. ~/.ssh/id_rsa.pub  默认生成的用户的公钥文件，用于将该公钥追加到需要登录的服务器authorized_keys文件
5. /etc/ssh/ssh_config
6. /etc/ssh/sshd_config

**使用模式**

这里推荐一种使用的模式，在利用脚本做自动化的时候，可以利用ssh操作远程主机，这种方式可以灵活的运用管道，如上面修改authorized_keys。例：

将远程主机$HOME/src/目录下面的所有文件，复制到用户的当前目录:`ssh user@host 'tar cz src' | tar xzv`

将$HOME/src/目录下面的所有文件，复制到远程主机的$HOME/src/目录: `cd && tar czv src | ssh user@host 'tar xz'`

## 转发

**动态转发**：`ssh -D 8080 user@host `   

加密连接，sock5代理，广泛使用的ss软件就是用的这个。
另外还可以作为跳板机实现，公网的服务器有些没有外网ip，通过有外网的服务器作为代理去访问那些只有内网ip的服务器。


**本地转发**：`ssh -L 2121:host2:21 host3`

一般用于无法直连的场景，比如防火墙，没有开发公网端口等， 本地不能直接连接host2，需要用host3来做中转。
当时我们公司的一个场景，我们的服务器一些后台没有开通外网端口，在公司内部我们需要访问后台，利用内网的一台服务器ssh本地转发到公网服务器，我们在内网直接访问内网服务器。


**远程端口转发**：`ssh -R 2121:host2:21 host1`

一般用于公网访问局域网的场景。

> xsell中 菜单->查看—>隧道窗格中可以快速创建这三种类型。


## 跳板机


存在跳板机的场景示例如下：

```
# 首先登陆跳板机
ssh -A 用户名@跳板机ip
ssh -A 用户名@目标机器ip
```



## scp 命令

例子：`scp test.txt chen@centos:/home/chen/data/`

1. -P  指明端口
2. -r  递归复制
3. -i  指明密钥文件


## rsync命令

例子： `rsync -avuz ~test/ chen@centos:/home/chen/data/`

rsync命令和scp类似，主要是采用'rsync'算法只同步不同的文件，支持压缩传输和断点续传，一般情况下速度更快。参数如下：
1. -t  不更新modify time
2. -z  压缩
3. -P  断点续传功能，大文件用到
4. -r  递归传递
5. -I  强制同步
6. -a 归档模式并保持所有文件属性， 等价于-rlptgoD (no -H,-A,-X)
7. -v 输出传输详情
8. -u 如果接受者上的文件比传输者的新旧不同步

另外rsync还有一种服务器模式，采用rsync服务端和客户端的模型，需要长期同步文件，推荐使用这种模式，这种模式的账号和linux系统账号是分开的，更加安全。

## sz和rz命令

这两个命令十分方便，在Windows下使用xshell客户端，如果遇到跳板机这种场景，需要频繁的穿来穿去，这两个命令可以自动穿隧道，十分方便，命令本省十分简单。
1. -e 采用二进制传输，这个非常重要，有时候在传输可执行文件时
2. -y 如果存在则覆盖原文件，默认是生成一个


-o "StrictHostKeyChecking no"

## 参考：   
1. [SSH原理与运用（一）：远程登录](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)
2. [SSH原理与运用（二）：远程操作与端口转发](http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)
3. [Linux ssh命令详解](https://www.cnblogs.com/ftl1012/p/ssh.html)
4. 