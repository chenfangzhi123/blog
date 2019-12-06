[TOC]
## 一、ssh命令

**登录类型**
1. 密码登录： 服务器发送公钥给客户端，客户端使用公钥加密后回传给服务器，服务器解密验证密码。
2. 公钥登录： 服务器发送一个随机字符串给客户端，客户端用私钥加密，服务器用公钥解密（rsa作为签名使用）

----


**ssh命令相关参数**
1. -A 密钥转发 这个参数在使用跳板机等场景非常有用，如果发现始终连不上需要检查下这个
2. -i 指定密钥文件
3. -p 端口号
4. -C：请求压缩所有数据；
5. -f 后台运行
1. -N 参数： 不要求分配shell，有些场景下ssh禁止账号请求shell终端，比如这个账号只是作为转发
1. -g  默认这个LocalPort端口只允许本机连接，可以通过这个参数允许别的机器连接这个端口
2. -T ：不要求分配终端
3. -o ServerAliveInterval=60 隔段时间发送保活消息
5. -q 抑制一些调试性的额外输出
7. -v 显示详细的调试信息，如果ssh连不上可以使用这个参数看看哪一步出问题了

----

**相关的命令**
1. ssh-keygen 用于生成密钥对
2. ssh-copy-id 用于复制公钥到服务器
> 复制公钥也可以使用：ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub

----
**相关的文件** 
1. ~/.ssh/authorized_keys  用于保存用户的公钥文件
2. ~/.ssh/known_hosts文件  保存的服务器用于辨别服务器的唯一散列码
3. ~/.ssh/id_dsa 用户的私钥文件
4. ~/.ssh/id_rsa.pub  默认生成的用户的公钥文件，用于将该公钥追加到需要登录的服务器authorized_keys文件
5. /etc/ssh/ssh_config  客户端ssh配置
6. /etc/ssh/sshd_config 服务端ssh配置


----
**使用模式**

这里推荐一种使用的模式，在利用脚本做自动化的时候，可以利用ssh操作远程主机，这种方式可以灵活的运用管道，<font id="ssh" color=red size=5>使用了重定向</font>，如上面修改authorized_keys。例：

将远程主机$HOME/src/目录下面的所有文件，复制到用户的当前目录:`ssh user@host 'tar cz src' | tar xzv`

将$HOME/src/目录下面的所有文件，复制到远程主机的$HOME/src/目录: `cd && tar czv src | ssh user@host 'tar xz'`

## 二、端口转发

**动态转发**：`ssh -D 1080 user@host -Nfg`   

最广泛的用途是作为sock5代理，另外还有加密连接的附加好处，广泛使用的ss软件就是用的这个。
另外还可以作为跳板机实现，公网的服务器有些没有外网ip，通过有外网的服务器作为代理去访问那些只有内网ip的服务器。

----
**本地转发**：`ssh -L LocalPort:remoteHost:remotePort sshHost`    
注意这里`remoteHost:remotePort`是相对于sshHost的地址，比如remoteHost设置为localhost，实际就是sshHost本地


一般用于无法直连的场景，比如防火墙，没有开发公网端口等， 本地不能直接连接remoteHost，需要用sshHost来做中转。
当时我们公司的一个场景，我们的服务器一些后台没有开通外网端口，在公司内部我们需要访问后台，利用内网的一台服务器ssh本地转发到公网服务器，我们在内网直接访问内网服务器。

----
**远程转发**：`ssh -R LocalPort:remoteHost:remotePort sshHost`

注意这里`remoteHost:remotePort`是相对于ssh命令执行的机器的和本地转发不同。    
另外注意这个命令执行和机器和本地转发不同。比如我们有这么个需求，将服务器serverA的21端口映射到client的2021。         
本地转发：这时我们在客户机上执行本地转发命令,`ssh -L 2021:localhost:21 serverA`
远程转发： 则是在服务器上运行，`ssh -R 2021:localhsot:21 client`,client指的是我们的客户机，也就是说client需要有sshServer    


> 上面的本地转发和远程转发很像，同样的功能命令差在一个参数，但两者有时候不可相互取代。本地和远程从数据的出口来记：    
> 本地：客户端连接sshServer将本地的数据转发到本地端口转发出去      
> 远程： 客户端连接sshServer，在sshServer建立端口，数据从sshServer到本地来

一般用于公网访问局域网的场景。在局域网的机器建立远程转发让公网的服务器可以访问局域网

> xsell中 菜单->查看—>隧道窗格中可以快速创建这三种类型。

---
## 三、跳板机登录

很多时候线上服务器的权限管理是通过跳板机来控制的，比如服务器a,b,c你不能直接连接，而是通过先登录跳板机再去连接。如果你现在想在本地连接服务器，有如下方案：
1. 在本地使用动态转发，如`ssh -D 1080 user@host `主机和用户使用的是跳板机，在xshell中新建连接时使用1080作为代理，此时你可以将这个连接认为是跳板机在连，比如你在连接中填写localhost，这个localhost到时就是跳板机
2. 在本地使用本地转发，如`ssh -L 2222:hosta:22 tiaobanHost`，这时候我们就可以使用localhost和2222端口在连接服务器a，不需要配置代理
3. 远程转发一般不用，因为服务器不能访问公司的局域网

上面的方法虽然可以实现登录后端服务器，但是两部操作还是有些不便，可以使用更方便的ProxyCommand。

该方法也有两种形式：  
1. `ssh -o ProxyCommand="ssh user@jumpHost -W %h:%p" serverHost`
2. `ssh -o ProxyCommand="nc -x jumpHost:jumpPort %h:%p" serverHost`

这个命令如果经常使用可以将ProxyCommand写入到ssh的配置文件中

现在有三个机器
1. 客户机:192.168.199.3
2. 跳板机：192.168.199.6
3. 目标机：192.168.199.5

-----

**第一种执行：`ssh -o ProxyCommand="ssh 192.168.199.6 -W %h:%p" 192.168.199.5`**   
> 注意这个-W是在新版中才加入，openssh 5.4之后才支持，相当于简化版的nc

客户机进程：
```shell
chen      50607  50529  0 17:52 pts/0    00:00:00 ssh -o ProxyCommand=ssh 192.168.199.6 -W %h:%p 192.168.199.5
chen      50608  50607  0 17:52 pts/0    00:00:00 ssh 192.168.199.6 -W 192.168.199.5:22
```

客户机显示的连接：
```
tcp        0      0 192.168.199.3:34306     192.168.199.6:22        ESTABLISHED 50608/ssh
```

跳板机显示的连接：
```
tcp        0      0 192.168.199.6:36932     192.168.199.5:22        ESTABLISHED -                   
tcp        0      0 192.168.199.6:22        192.168.199.3:34306     ESTABLISHED - 
```

目标机显示的连接： 
```
tcp        0      0 192.168.199.5:22        192.168.199.6:36932     ESTABLISHED - 
```
从上面的结果可以看到，跳板机和两头各建立了一个连接，另外客户机是50608进程占用了这个连接

---


**第二种执行：`ssh -o ProxyCommand="ssh 192.168.199.6 nc %h %p" 192.168.199.5`**

这种方式和方面的一样，显示的连接也都一致。 


----
**最后说说一种nc**

注意这种方式需要有个sock5代理，所以跳板机先开启代理：`ssh -D 4000 192.168.199.5 -Nfg`
nc支持多种代理，包活scok4，sock5和http，这种方式和上面的两种完全不同。有一点很奇怪如果跳板机没开sock5代理也没有任何报错信息，ssh并没有
并没有使用代理而是直接连接目标服务器
**`ssh -o ProxyCommand="nc -x 192.168.199.6:4000 %h %p" 192.168.199.5`**


## 四、scp 命令

例子：`scp test.txt chen@centos:/home/chen/data/`

1. -P  指明端口
2. -r  递归复制
3. -i  指明密钥文件


## 五、rsync命令

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


指定端口，如1280端口： `rsync -ravuz -e 'ssh -p 1280' 192.168.10.10:/home/chenfangzhi/ .` 
另外rsync还有一种服务器模式，采用rsync服务端和客户端的模型，需要长期同步文件，推荐使用这种模式，这种模式的账号和linux系统账号是分开的，更加安全。

## 六、 ssh-agent

最后说的这个东西是非常有用的，如果经常使用的ssh的肯定会遇到需要用多个私钥的场景和私钥被加密的场景。如果私钥被加密，每次连接还是需要输入密码，
当在各个服务器之间穿可能需要多次id_rsa密码，很是繁琐，另外多个多个主机采用不同的私钥的时候需要指定私钥，ssh-agent就是用来解决这个问题的。

这个功能需要在sshd配置文件中配置AllowAgentForwarding，ssh配置文件中配置ForwardAgent

1. eval \`ssh-agent -s\` ：开启agent，这里必须使用eval
2. ssh-add  id_rsa_file:用来添加密钥，这里如果不指定文件则添加的是~/.ssh/id_rsa文件


## 七、ssh执行命令不退出问题


我们在批量执行服务器命令经常会用到`ssh host command`命令，但是在有时候发现这个命令不能正常退出，说下这种模式下命令正常退出的条件：

1. 远程的执行的进程执行完成或者放入后台运行
2. 如果是放入后台运行要保证该进程或子进程的标准输入输出和当前ssh进程没有联系

这种问题经常出现的场景是，我去批量启动服务器上的服务，调用start.sh脚本之后（`ssh host "bash start.sh &"`），发现无法退出。这时候我采用了
`ssh host "nohup bash start.sh &"`，发现还是不能正常退出。最后查了下就是上面的两个原因，因为start.sh脚本中启动了新的进程，子进程继承了bash进程文件描述符，
也就是输入输出都和bash相同，和ssh进程还有联系。所以改写为：`ssh host "bash start.sh &>out.log &"`成功运行。
另外说说nohup这个命令，在这里其实nohup是完全没必要使用的，一方面是因为nohup是为了忽略SIGHUP信号，但是如果是使用`ssh host command`这种模式的话，进程是不会和终端绑定也就是进程不会
收到SIGHUP信号。另一方面是nohup是在执行的进程的标准输入和输出绑定了终端时会重定向，但是这个场景下，标准输入输出已经被重定向到了管道。

ssh有个-t参数，如果加入这个参数，则会默认分配一个终端，上面的逻辑就变了，需要使用nohup来进行重定向。但是我发现如果是使用`ssh -t host "nohup bash start.sh &>out.log &"`启动进程还会导致进程无法启动的问题，网上说是ssh进程退出的太快了，导致nohup被杀死。但是&后面不能sleep函数，语法报错。一种方法是start.sh中启动服务地方加入&，然后上面改写成``ssh -t host "nohup bash start.sh &>out.log;sleep 1"``


>重定向的问题参看<a href="#ssh">上面红字</a>

我在虚拟机的实验结果如下：
```
[chen@cc1 ~]$ ssh root@192.168.199.5  "TMPSPID=\$(ps -ef | grep -v grep |grep -e 'sshd.*notty' | awk '{print \$2}');echo \$TMPSPID;ls -l /proc/\$TMPSPID/fd;echo \$\$;ls -l /proc/\$\$/fd"
root@192.168.199.5's password: 
14719
total 0
lrwx------. 1 root root 64 May 28 21:37 0 -> /dev/null
lrwx------. 1 root root 64 May 28 21:37 1 -> /dev/null
l-wx------. 1 root root 64 May 28 21:37 11 -> pipe:[122988]
lr-x------. 1 root root 64 May 28 21:37 12 -> pipe:[122989]
lr-x------. 1 root root 64 May 28 21:37 14 -> pipe:[122990]
lrwx------. 1 root root 64 May 28 21:37 2 -> /dev/null
lrwx------. 1 root root 64 May 28 21:37 3 -> socket:[122854]
lrwx------. 1 root root 64 May 28 21:37 4 -> socket:[122951]
lr-x------. 1 root root 64 May 28 21:37 5 -> pipe:[122954]
l-wx------. 1 root root 64 May 28 21:37 6 -> /run/systemd/sessions/94.ref
l-wx------. 1 root root 64 May 28 21:37 7 -> pipe:[122954]
14725
total 0
lr-x------. 1 root root 64 May 28 21:37 0 -> pipe:[122988]
l-wx------. 1 root root 64 May 28 21:37 1 -> pipe:[122989]
l-wx------. 1 root root 64 May 28 21:37 2 -> pipe:[122990]
```

## 八、sz和rz命令

这两个命令十分方便，在Windows下使用xshell客户端，如果遇到跳板机这种场景，需要频繁的穿来穿去，这两个命令可以自动穿隧道，十分方便，命令本省十分简单。
1. -e 采用二进制传输，这个非常重要，有时候在传输可执行文件时
2. -y 如果存在则覆盖原文件，默认是生成一个


## 参考文章

1. [SSH原理与运用（一）：远程登录](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)
2. [SSH原理与运用（二）：远程操作与端口转发](http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)
3. [Linux ssh命令详解](https://www.cnblogs.com/ftl1012/p/ssh.html)
4. [ssh -W and ssh nc](https://stackoverflow.com/questions/22635613/what-is-the-difference-between-ssh-proxycommand-w-nc-exec-nc)
5. [ssh远程执行nohup命令不退出](https://blog.csdn.net/oneinmore/article/details/50073443)