linux的内存备忘


## 理解buffer和cache

Buffer：针对块设备的缓冲区
Cache： 内存页的缓存,加快文件读写等


早期：Buffer被用来当成对io设备写的缓存，Cache被用来当作对io设备的读缓存，这里的io设备，主要指的是块设备文件和文件系统上的普通文件


现在一般情况下，在linux系统不够的时候会回收部分Buffer和Cache的内存，但是想tmpfs，mmap和共享内存导致占用的cache并不能被回收。    


## 手动回收Cache


参数：**/proc/sys/vm/drop_caches**

1. echo 1 > /proc/sys/vm/drop_caches：表示清除 page cache。
2. echo 2 > /proc/sys/vm/drop_caches：表示清除回收 slab 分配器中的对象（包括目录项缓存和 inode 缓存）。slab 分配器是内核中管理内存的一种机制，其中很多缓存数据实现都是用的 page cache。
3. echo 3 > /proc/sys/vm/drop_caches：表示清除 page cache 和 slab 分配器中的缓存对象。



## 交换分区

swap分区主要是
Swap 的作用是把一个本地文件或者一块磁盘空间作为内存来使用，包括换出和换入两个过程。Swap 需要读写磁盘，所以性能不是很高，事实上，包括 ElasticSearch 、Hadoop 在内绝大部分 Java 应用都建议关掉 Swap，这是因为内存的成本一直在降低，同时这也和 JVM 的垃圾回收过程有关：JVM在 GC 的时候会遍历所有用到的堆的内存，如果这部分内存被 Swap 出去了，遍历的时候就会有磁盘 I/O 产生。Swap 分区的升高一般和磁盘的使用强相关，具体分析时，需要结合缓存使用情况、swappiness 阈值以及匿名页和文件页的活跃情况综合分析。



参数：**/proc/sys/vm/swappiness**

`cat /proc/sys/vm/swappiness`显示为60表示内存在使用到100-60=40%的时候，就开始出现有交换分区的使用

1. swappiness=0表示最大限度使用物理内存，然后才使用swap空间
2. swappiness＝100表示积极使用swap分区，并且把内存上的数据及时的搬运到swap空间里面







参考链接 

1. [cache都能被回收么？](https://linux.cn/article-7310-1.html)
2. [如何回答性能优化的问题，才能打动阿里面试官？](https://mp.weixin.qq.com/s?__biz=MzIzOTU0NTQ0MA==&mid=2247492338&idx=1&sn=1b261f2eda75163e0878d3b5e4373834&chksm=e92adffdde5d56eb4720f9ee0b3b81795ee31bedb2ebe6a3112e0e1b538242e69076ff5aa715&token=1454462344&lang=zh_CN&scene=21#wechat_redirect)