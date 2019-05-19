1. 服务网络排查
服务报错文件描述符不够用，


利用lsof查看服务的进程发现，有大量的can't identify protocol文件。通过cat /proc/服务进程/limits发现文件描述符的上限是4096，而不是系统设置值100000。两个问题
通过lsof -p 服务id|wc -l计算了进程的文件描述符，发现最高在三千左右，


### limits未生效：

原因：通过ssh user@ip "command" 执行命令的时候，没有调用pam_limits模块，导致/etc/secturity/limits.conf设置未生效。查看线上sshd的版本是5.3，我在自己虚拟机centos7.3中测试sshd为7.4版本，发现没有这个问题。

解决方案：修改sshd中的将sshd的UseLogin no 设置为yes，重启sshd服务。因为程序修改limits值需要root权限，所以在ssh的cammand中设置limits值行不通

### 大量的time_wait

通过cat /proc/net/sockstat发现time_wait值在14万左右。首先想到两个网络连接，一个是haproxy，一个是dsp
利用ss -apt|grep 服务进程id 发现time_wait都是8086端口，说明是haproxy
统计8086端口的所有time_wait数量：ss -apt|grep TIME|grep 8086|wc -l 


这里是否可以通过重用连接在优化

### lsof文件中显示的 can't identify protocol


首先明确time_wait状态的连接已经不占用文件描述符，所有利用lsof计算的时候最高只有三千多的文件描述符而time_wait始终在100000以上。


利用lsof -i:8086发现连接最高不超过60，这个连接数变化比较大，有时候查询一个都没有，有时候有几十个，和ss -np|grep 服务进程|grep 8086结果一致。猜测因为8086上的高并发有关。

另外一组测试数据：
`ss -p|grep 服务进程|wc -l `为1430个
`lsof -p 服务进程|wc -l `为1709个（其中can't identify protocol有380个,8086端口的连接一个都没有，8086名字为d-s-n）

can't identify protocol这种状态是对应的是FIN_WAIT2，见：https://idea.popcount.org/2012-12-09-lsof-cant-identify-protocol/，可以利用其中的代码复现，在新版本的netstat中，`can't identify protocol`显示改为`protocol: TCP`

我也写了一个简单的脚本对比，启动server.py，然后启动client.py，过一会杀死client.py。利用lsof -p 进程号出现can't identify protocol，利用` ss -ap|grep 21168`命令则未观察到这些连接。也说明了lsof连接数显示比ss多的原因。


现在的状况是haproxy和服务之间有大量的短时间的请求，每次生成一个连接，请求完就销毁，导致大量的time_wait，其中time_wait之前的FIN_WAIT2这种状态也是需要占用占用文件描述符，估算应该在几百个连接左右。
dsp和服务之间有大量的连接，同时存在几千个。

结论：lsof的数据在高并发的场景下一致性很差，应该是ss的结果为准。虽然haproxy和服务之间有很高的并发，但是没有占用太多文件描述符，连接被快速销毁，但是FIN_WAIT2的连接还是需要占用描述符。有必要做haproxy的连接复用

