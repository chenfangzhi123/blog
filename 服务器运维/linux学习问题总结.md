[TOC]

本文主要是自己学习linux中的一些思考和总结的记录

## 一、环境变量和普通变量的区别

区别就是普通变量只会影响当前进程，子进程可以继承父进程的环境变量

## 二、rsyslog和logrotate会不会丢记录的问题

先说结论：不会

logrotate有create和copytruncate方案，这里是考虑create这种默认方案。说说logrotate的操作步骤：假设日志文件为daemon.log，当前已经到了轮换的时间，logrotate会先重命名daemon.log为daemon.log.1，然后新建一个daemon.log，给对应的进程发送HUP信号通知他日志已经更新

当时一直在思考这个问题，一个进程在写入一个日志，然后logrotate将该日志文件重命名，创建个新的同名文件的过程中，日志数据会不会丢失，这个问题其实还是对linux文件系统的理解不够深刻。

1. linux的文件分为inode和datanode，而文件系统是以inode来唯一标志这个文件的，进程在打开文件的时候对应这个inode的一个文件标志符。
2. 你要明白linux的文件名存在哪里，linux文件名其实是存在目录中，目录也是文件，该文件存放了文件名和inode的对应关系，这里也就理解了linux的硬链接，同一个文件可以有多个名字

理解了上面的也就理解了为什么不会，logrotate在重命名的时候只是修改了目录项，并没有影响实际的文件，所以<font color=red>该进程还是在往重命名后的文件写入</font>，logrotate给进程发信号通知进程，进程在响应信号的函数中去根据文件名查找新的inode，替换文件描述符，往新文件写入

这里要特别注意你的程序要能响应这个HUP信号，不处理这个信号默认行为就是停止进程。

## 三、为什么有些文件夹大小不是4096的整数倍

我们知道文件系统中文件存放需要inode(索引区块)和dnode(数据区块)，一个文件只有需要一个inode，dnode则是是一个或多个。目录也是文件，但是在查看目录的时候发现有时候目录竟然不占用区块，很多目录的大小都是4096的整数倍这个很好理解，因为文件系统每个dnode大小都是固定的，一般都是4k，所以占用一个或者多个就是4096的整数倍。那为什么有些目录大小不占用区块呢？

这其实是xfs的一个优化，如果目录项比较少，那么他就将数据存放到了目录的inode当中，所以不占用区块。

我在自己的电脑上里用`ll -s`命令查看跟目录，可以看到dev、bin和home不占用数据区块（看第一列）。

```shell
   0 lrwxrwxrwx.   1 root root    7 3月  23 08:30 bin -> usr/bin
   4 dr-xr-xr-x.   5 root root 4096 4月   3 22:01 boot
   0 drwxr-xr-x.  19 root root 3300 4月   3 23:17 dev
  12 drwxr-xr-x. 144 root root 8192 4月   6 11:18 etc
   0 drwxr-xr-x.   5 root root   45 4月   3 23:06 home
```

既然谈到了文件大小的问题就补充下ls命令和du命令

du查看文件和目录的大小
``` c
du -sh *
```

ls命令显示文件大小问题
```jshelllanguage
ls -s -S
-s 输出大小
-S 按大小排序
-r 逆序排序
```


## 四、reboot和shutdown等软链接实现原理

我们先来查看下reboot和shutdown的文件：
```shell
[chen@chen ~]$ ll `which reboot` `which shutdown`
lrwxrwxrwx. 1 root root 16 3月  30 22:12 /usr/sbin/reboot -> ../bin/systemctl
lrwxrwxrwx. 1 root root 16 3月  30 22:12 /usr/sbin/shutdown -> ../bin/systemctl
```

可以看到这两文件都是链接到systemctl符号链接，这里我觉得很奇怪，都是链接到systemctl为什么行为能不一样？难道这两个符号链接有什么不一样吗？

但是怎么去查看符号链接的内容，通过`cat /usr/sbin/reboot`这种命令去查看时是直接到了源文件，这时候可以使用`readlink`这个命令去读取符号链接的内容，内容如下：

```shell
[chen@chen ~]$ readlink `which reboot` `which shutdown`
../bin/systemctl
../bin/systemctl
```

可以看到内容是一样的，那么到底是怎么实现的。其实这里是通过获取启动时的名称来做判断的。看下下面的程序你就明白了：

```c
#include<stdio.h>

int main(int argc,char * argv[]){
        for(int i=0;i<argc;i++)
                printf("%s\n",argv[i]);
}
```

上面的程序打印了程序的输入参数，利用`gcc -std=c99 name.c`编译后产生a.out文件，建立一个a.out的符号链接`ln -s a.out b`。

```c
[chen@chen ~]$ `pwd`/a.out
/home/chen/a.out
[chen@chen ~]$ ./a.out
./a.out
[chen@chen ~]$ ./b
./b
```

可以看到输出的程序名称的不同，你在建立符号链接的时候用到了不同的名字，systemctl可以根据名字来走不同的逻辑。所以如果你在自己的目录建立一个reboot链接:`ln -s /bin/systemctl reboot`实现的效果是一样的。



## 五、systemd启动时执行脚本的问题

早先SystemV的init中我们有启动时需要执行的脚本时都会加入rc.local中，在systemd已经不推荐使用这个方法，我想很大的一个原因就是因为并行执行的问题。如果这个脚本有大量耗时的任务，那么这个脚本只能按顺序一个一个的执行才能启动。所以systemd其中很大的一个改进就是并行执行，它建议我们自己编写一个service的配置文件，利用systemd来管理。

我的理解是如果是快速简单的命令还是可以放在原来的init目录，毕竟比较方便

## 六、crontab计划任务随机执行

有计划任务想到的一个问题，如果我有很多任务在某个时刻需要执行，为了避免同一个时间执行导致负载过高所以需要一些随机化的处理而不是同一时刻触发

思想就是加入随机函数,例如要在1小时内随机化：利用RANDOM环境变量`$[(RANDOM%60]`

别忘了在cron配置文件中%需要转义：`$[(RANDOM\%60]`

最终结果如下 : `0 1 * * * sleep $[(RANDOM\%60]m ; /home/data/shell/script.sh`

## 七、日志输出和标准输出

这里谈谈自己java开发的业务中的日志，我们java的日志现在一般采用的是log4fj和logback这种，在日志的配置中我们一般会有多个appender，例如输出到文件的和输出到标准输出的。我们在ide中运行的程序的时候，标准输出的所有输出都会输出在console中，如果去线上部署，例如`nohup daemon & &>/dev/null`这种方式，标准输出被重定向到/dev/null中，因为日志也配置了文件的appender所以没什么问题，但是如果你对代码中有`System.out.println`这种输出，那么这些信息就都丢了。所以尽量使用log来统一输出日志，还可以配置下将程序的标准输出也写入到日志中。


参考链接

1. [why-directories-size-are-different-in-ls-l-output-on-xfs-file-system](https://superuser.com/questions/585844/why-directories-size-are-different-in-ls-l-output-on-xfs-file-system)
2. [Cron jobs and random times, within given hours](https://stackoverflow.com/questions/9049460/cron-jobs-and-random-times-within-given-hours)
