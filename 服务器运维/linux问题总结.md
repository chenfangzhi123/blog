## 环境变量

只会影响当前进程和子进程不会影响别的进程



## rsyslog和logrotate会不会丢记录的问题

不会



## systemd启动时执行脚本的问题

如果是快速简单的命令可以放在原来的init目录



## 日志输出和标准输出

## 计划任务随机执行而不是同一时刻触发

## 为什么有些文件夹大小不是4096的整数倍

https://superuser.com/questions/585844/why-directories-size-are-different-in-ls-l-output-on-xfs-file-system

查看大小
``` c
du -sh *
```
ls命令显示文件大小问题
ls -s -S

-s 输出大小
-S 按大小排序
-r 逆序排序
## reboot和shutdown等软连接实现原理


## ssh远程执行


## linux权限问题

vmware的网关问题


readlink命令

lsof命令详解

tmpfs和proc文件系统