
## 虚拟机问题排查工具
### jps

jps这个命令十分有用，在通过ps命令定位到异常的进程之后，可以通过jps来定位到具体的java进程。其中有些有用的参数

1. -l 输出启动类的全限定名
2. -v 输出jvm启动参数
3. -m 输出传给main方法的参数
 
### jmap

这个主要用于排查内存问题，当出现内存溢出和gc异常的时候可以通过他来dump下堆和查看堆的信息。常用的如下：

1. -heap 查看堆的信息，比如使用了那种垃圾收集器，堆的大小等
2. -dump dump堆的内容，然后可以用其他工具分析。 jmap -dump:live,format=b,file=heap.bin <pid> live表示只打印存活对象
3. -permstat 查方法区的统计信息
4. -finalizerinfo 查看终结队列中的对象数量
5. -F 虚拟机未响应时强制响应

### jstack

这个主要用于排查cpu和死锁问题，在通过ps定位到异常进程，用H参数定位到线程，然后用jstack去打印线程堆栈，查看那个方法有问题。 他的话参数不多，主要就—F和-l，-F同上，-l用于输出锁相关的东西，可以帮助排查死锁等。


### jstat和jinfo

这两个命令都是用来查看虚拟机信息，jstat是查看gc，类加载和编译的信息，jinfo主要是查看虚拟机的配置信息，比如系统变量和虚拟机参数。jinfo还可以动态修改一些参数。
> java -XX:+PrintFlagsFinal -version|grep manageable这些参数可以被jinfo动态修改
