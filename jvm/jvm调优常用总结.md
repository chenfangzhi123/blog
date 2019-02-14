本文主要是工作过程中总结的一些jvm调优的参数和注意的地方，作为一个备忘录，先占个坑，有时间在来细化具体的实例。

1. gc日志是覆盖的方式如果文件名字固定会导致上一次被覆盖可以采用这个-Xloggc:backv2_gc_%t.log
2. **jinfo**可以动态修改java -XX:+PrintFlagsFinal -version|grep manageable这些参数
3. **打印java可配置的非稳定参数**：java -XX:+PrintFlagsFinal ，输出的信息中 “:=” 表明了参数被用户或者 JVM 赋值了
3. jstat可以查看类加载和gc的耗时信息 -t参数表示每行前面输出时间
4. java堆溢出时获取heap dump -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./m.hprof
5. 当系统发生OOM错误时，虚拟机在错误发生时运行一段第三方脚本， 比如， 当OOM发生时，重置系统 -XX:OnOutOfMemoryError=c:\reset.bat
6. 取消outofmemory警告：-XX:-UseGCOverheadLimit
6. **获取GC信息**
    1.  -verbose:gc(-verbose:class可以输出类加载的信息) 或者 -XX:+PrintGC 打印gc日志
    2.  如果要获得更加详细的信息， 可以使用 -XX:+PrintGCDetails.
    3.  如果需要查看新生对象晋升老年代的实际阈值， 可以使用参数 -XX:+PrintTenuringDistribution
        > java8中使用这个参数没有详细输出各个年龄的分布，因为java8默认的收集器是ParallelGC和ParallelGC Old，
        > 这个收集器注重吞吐量没有用年龄，所以没必要打印详细的年龄分布。只会显示晋升到老年代的阈值还有期望的Survivor区的预期大小。
    4.  输出jvm启动时的参数  -XX:+PrintFlagsInitial
7. **禁用代码中显示的触发FULL GC** [System.gc()]：-XX:+DisableExplicitGC 
8. **64位机器上压缩指针** -XX:+UseCompressedOops 
9. **禁用类验证**   -Xverfy:none
10.  禁用类元数据回收 -Xnoclassgc     
11.  确定堆内存大小 -Xmx 堆最大内存, -Xms对最小内存
12.  合理分配新生代和老生代-Xmn 新生代大小, -XX:SurvivorRatio Eden和Survivor空间的比例 默认是8  设置年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代) -XX:NewRatio=4 默认是2    
> **老年代和新生代大小比例调节**：如果应用存在大量的临时对象，应该选择更大的年轻代；如果存在相对较多的持久对象，年老代应该适当增大。但很多应用都没有这样明显的特性，在抉择时应该根据以下两点：   
>a.本着Full GC尽量少的原则，让年老代尽量缓存常用对象，JVM的默认比例1：2也是这个道理   
>b.通过观察应用一段时间，看其他在峰值时年老代会占多少内存，在不影响Full GC的前提下，根据实际情况加大年轻代，比如可以把比例控制在1：1。但应该给年老代至少预留1/3的增长空间
14.  设置每个线程的堆栈大小，如：-Xss128k
14. 导致jvm停顿的原因
    1.  垃圾收集
    2.  代码反优化
    3.  Flushing code 
    4.  类重定义如热加载 
    5.  取消偏向锁
    6.  调试动作（死锁检测，输出线程堆栈）
15. **STW**的四个阶段
    1. Spin阶段:因为jvm在决定进入全局safepoint的时候，有的线程在安全点上，而有的线程不在安全点上，这个阶段是等待未在安全点上的用户线程进入安全点。
    2. [Block阶段:即使进入safepoint，用户线程这时候仍然是running状态，保证用户不在继续执行，需要将用户线程阻塞](http://blog.csdn.net/iter_zc/article/details/41892567)
    3. Cleanup:这个阶段是JVM做的一些内部的清理工作
    4. VM Operation. JVM执行的一些全局性工作，例如GC,代码反优化,偏向锁  
16. **jvm停顿时间的输出**：-XX:+PrintGCApplicationStoppedTime  上一次gc停顿程序运行时间 -XX:+PrintGCApplicationConcurrentTime
17. **jvm停顿原因分析**： -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1
    1. vmop：引发STW的原因，以及触发时间该项常见的输出有：RevokeBias、BulkRevokeBias、Deoptimize、G1IncCollectionPause。GC log可以根据该项内容定位Total time for which application threads…引发的详细信息。
    2. total ：STW发生时，JVM存在的线程数目。
    3. initially_running ：STW发生时，仍在运行的线程数，这项是Spin阶段的 时间来源
    4. wait_to_block ： STW需要阻塞的线程数目，这项是block阶段的时间来源
    5.  输出如下
        ```
            发生时间    操作                        线程   总数   正在运行        等待阻塞               
                       vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
            0.462: ForceSafepoint              [           8          0              1        ]    [       0     0     0     0     0  ]      0  
        ```
18. -XX:+UnlockDiagnosticVMOptions -XX:-DisplayVMOutput -XX:+LogVMOutput -XX:LogFile=vm.log 可以将详细的停顿信息输出到日志文件中
19. 来解锁任何额外的隐藏参数-XX:+UnlockDiagnosticVMOptions和-XX:+UnlockExperimentalVMOptions
20. 输出启动时的参数信息和vm根据环境设置的参数信息：  -XX:+PrintCommandLineFlags    
18. CMS垃圾收集器的理解
    1. [参数优化](https://www.cnblogs.com/onmyway20xx/p/6605324.html)
    2. cms清理老年代并不是[FULL GC ](https://www.zhihu.com/question/41922036/answer/93079526)
19. java启动参数-agentlib,最常用的两种常见一个是jdwp远程调试，还有个hprof内存和CPU分析
20. string的intern方法，内部实现是hash数据结构，参数StringTableSize可以用来控制散列桶的数量，java8默认是60013，自定义的话最好设置一个素数
21. JIT相关参数：-Djava.compiler=NONE禁用JIT，-XX:+PrintCompilation 打印JIT编译情况
22. 定制JIT编译的参数 -XX:CompileCommand



待解决：
GC overhead limit exceeded问题


