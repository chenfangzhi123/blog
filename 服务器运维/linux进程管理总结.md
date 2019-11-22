[TOC]

## 一、进程相关的概念

进程需要了解 进程，父进程，进程组,会话和控制终端的相关概念。

1. 进程和父进程：每个进程都有父进程，而所有的进程以init进程为根，形成一个树状结构   

2. 进程组：每个进程都会属于一个进程组(process group)，每个进程组中可以包含多个进程。进程组会有一个进程组领导进程 (process group leader)，领导进程的PID成为进程组的ID (process group ID, PGID)，以识别进程组。
    > kill给组发送信号进程组号前加负号如：kill -9 -2189

3. 会话：一个或是多个进程组集合。 进程可以通过调用 pid_t setsid(); 来建立一个新会话，如果调用此函数的进程不是进程组长，就会创建一个新的会话，那么此时会：
    1. 该进程称为会话首进程 (session leader)
    2. 该进程称为进程组组长
    3. 该进程没有控制终端，即使之前有控制终端这种联系也会断掉
    >可以使用第三个特性来创建 daemon 进程。 调用 getsid 可以获得会话首进程进程组 pid，也就是会话首进程进程 id。
    
 4. 控制终端：
     1. 一个会话持有一个控制终端 (controlling terminal)，可以是终端设备也可以是伪终端
     2. 建立与控制终端连接的会话首进程被称为控制进程 (controlling process)
     3. 一个会话有多个进程组，允许存在多个后台进程组 (backgroup process group) 和一个前台进程组 (foregroup process group)
     4. 键入终端的中断键 (Ctrl+C) 会发送中断信号给前台进程组所有进程
     5. 键入终端的退出键 (Ctrl+\) 会发送退出信号给前台进程组所有进程
     6. 终端或是网络断开会将挂断信号发送给会话首进程
     
可以看到执行ps -fj结果如下：

```bash

UID         PID   PPID   PGID    SID  C STIME TTY          TIME CMD
chen      36829  36825  36829  36829  0 10:56 pts/0    00:00:00 -bash
chen      37247  36829  37247  36829  0 10:57 pts/0    00:00:00 vim
chen      90490  36829  90490  36829  0 11:57 pts/0    00:00:00 ps -fj

```
其中PID就是进程id，PPID是父进程id，PGID为进程组id，SID为会话ID

## 二、关闭会话时子进程进程被杀死

终端在关闭时会发送SIGHUP信号给session leader，此处就是bash进程，bash收到后向session内的所有进程发送SIGHUP然后退出。
SIGHUP信号如果为注册处理函数默认行为就是退出。所以会话退出时子进程都被杀死。

解决方案：
1. 注册SIGHUP信号处理函数：可以在代码中处理或者使用nohup命令(`nohup daemon & &>daemon.log`)
2. 重新设置setsid：可以在代码中处理或者使用setsid命令(`setsid daemon`)
 
## 三、nohup的原理

其实很简单就是注册了SIGHUP的一个处理函数，忽略这个信号，然后去执行实际的命令。
源码地址：https://github.com/MaiZure/coreutils-8.3/blob/master/src/nohup.c

关键代码：

```c
   // 注册处理函数
  signal (SIGHUP, SIG_IGN);

  char **cmd = argv + optind;
  //执行实际的代码
  execvp (*cmd, cmd);
```
## 四、setsid原理

fork进程之后的子进程共享父进程的很多东西，并且会话组长就是父进程的会长组长，所以会收到来自父进程会话组长的信号。
setsid用余新建一个会话，调用这个函数之后会当当前进程成为进程组组长和会话组组长，那么原来的会话产生的信号便不会发送到这个进程，从而不会受影响。

## 五、daemon &和守护进程的区别

因为守护进程的实现是用的setsid，所以其实就是setsid和nohup的区别，两者都可以用来防止进程在终端断开的时候被杀死，nohup还需要配合&放入后台运行。区别的的话守护进程已经脱离了终端，不受终端控制，也就没有 了stdin，stdout和stderr，而使用nohup之后的进程还是有一个终端，只是忽略了其中的SIGHUP信号，存在正常的stdin，stdout和stderr，nohup默认将stdout和stderr重定向到了nohup.out，nohup的使用也推荐`< /dev/null`来重定向stdin

最佳实践：
1. 如果是一次性的后台任务，可以使用nohup十分方便
2. 如果是长期运行的服务，则推荐使用系统的systemd来管理服务
3. 如果是定时运行的任务则推荐使用cron来运行

## 六、服务进程为什么要fork两次

首先说明两次不是必须的，有很多程序都采用了一次fork。

第一次：为了调用setsid，这也解释了为什么调用setsid之前需要先fork的原因：
linux规定调用这个函数之前,当前进程不允许是session leader。进程组leader是该进程组的第一个进程，fork出来的进程必定不是第一个，所以可以调用setsid。另外父进程一般直接退出，可以让shell收到进程结束的通知继续执行，而不是等待他结束。

第二次：为了限制进程打开控制终端，只有会话组长能打开控制终端（非必须，相当于加了个限制条件Daemon不需要打开终端）


## 七、systemd管理daemon

现在很多的linux发行版都采用systemd来代替原来的init程序，systemd提供了很优秀的进程管理功能，我们需要注册服务时可以利用systemd功能，可以参看鸟哥的systemd介绍。
 
另外补充点内核进程和Systemd进程：  
0号进程为内核进程，1号为Systemd进程，其他还有些内核进程在ps命令查看时以[]包裹。具体关系见：[LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)


## 八、僵尸进程

这个定义摘抄自维基百科：在类UNIX系统中，僵尸进程是指完成执行（通过exit系统调用，或运行时发生致命错误或收到终止信号所致）但在操作系统的进程表中仍然有一个表项（进程控制块PCB），处于"终止状态"的进程。这发生于子进程需要保留表项以允许其父进程读取子进程的exit status：一旦退出态通过wait系统调用读取，僵尸进程条目就从进程表中删除，称之为"回收（reaped）"。

## 九、进程名字和启动时指定进程名字

kill，ps，top，pstree这些命令都比较熟悉就不再提了。

至于还有一组命令则不是通过进程号而是通过进程名字来操作进程，pkill和killall一样都是通过名字来杀死进程，而pgrep是通过名字来寻找进程。
他们的原理都是通过查找/proc这个内存文件系统。

在启动的时候可以通过exec命令重命名：
`bash -c "exec -a myname sleep 500 &"`

 你可以通过`ps -ef|grep myname`来查看进程的详细信息


## 十、source command和./command 和exec命令的区别

通常执行脚本有三种方式
1. ./command(同sh command) 
2. source command(同. command)
3. exec command

简单说明下上面三种方式：

第一种其实就是对应了linux的fork系统调用，在执行command时候，command是在子进程中执行的，当前shell等待直到子进程的command运行完毕在返回到当前shell。第二种则是直接在当前的进程中直接执行，执行完继续接受用户输入。第三种则对应了linux的exec系统调用，当前进程的执行流程会转向command，command是在当前进程直接执行，但是执行完之后便会直接退出。

所以我们一般用的是第一和第二两种，这种的主要区别就是开不开新的进程（开进程是要一定开销的），另外因为第二种是在当前进程执行的，所以如果在command中设置了变量，那么相当于在当前进程中设置了变量，所以我们一般是用第一种去执行避免当前进程的变量被污染。 


思考：

现在加入你在终端已经运行了一个非常耗时的任务，你按ctrl+z放入了后台，然后利用bg开始任务，因为终端断开就会收到SIGHUP信号，有没有办法忽略这个信号或者终端断开不收到这个信号？


遗留：

进程调试工具：ltrace strace ftrace

参考链接：

## 十一、disown命令

有这么一种场景，如果你已经在运行一个软件了，但是中途想要退出，但是如果此时关闭中断会导致进程收到SIGHUP信号，对于已经启动的进程，disown命令就是用来解决这个问题的。对于运行中的命令，我们可以通过ctrl+z命令放入后台，注意此时进程转为stop状态(crtl+z会给这个进程发送STOP信号)，我们可以利用bg命令恢复进程的运行。然后执行disown命令，这样进程就不会收到SIGHUP信号了。如下：
```shell
[chen@chen ~]$ disown
-bash: warning: deleting stopped job 1 with process group 12882
```
注意上面的警告，我使用ctrl+z命令后没有使用bg恢复进程的执行，然后直接使用了disown命令，他会警告移除了一个停止状态的任务。此时可以使用`kill -CONT pid`（bg命令实际也是使用了这个信号）恢复进程的运行

命令解释：
```
disown [-ar] [-h] [jobspec ...]

-a：移除所有的后台任务
-r：移除正在运行中的任务
-h：不移除任务，但是会阻止控制进程给任务进程发SIGHUP信号
jobspec :可以是进程号，也可以是任务列表

默认直接执行disown就是移除最近的任务
```

1. [Linux进程组和会话](https://my.oschina.net/hosee/blog/507098)
2. [在线APUE译文](http://zdyi.com/books/apue/)
3. [linux终端关闭时为什么会导致在其上启动的进程退出？](https://blog.csdn.net/ybxuwei/article/details/77149575)
4. [What's the difference between nohup and a daemon?](https://stackoverflow.com/questions/958249/whats-the-difference-between-nohup-and-a-daemon?newreg=60c6365482544823b940ebd4e33089f4)