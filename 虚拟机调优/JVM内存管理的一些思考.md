这个文章主要是自己关于jvm内存的一点思考，范围比较杂，设计类加载器，方法区和内存泄露等

## 一、 内存是怎么分配的 

主要是指针碰撞和空闲列表两类。新生代一般是复制算法，老年代一般是标记整理（cms用了标记清除导致内存碎片较多）。复制和标记整理采用指针碰撞，标记清除采用标记清除。如果是指针碰撞需要考虑指针的冲突，有cas和本地线程分配缓冲两种策略。

## 二、 方法区

方法区包括 常量池（jdk7被移入堆中），代码，类信息，类静态变量。jdk8后方法区实现为本地内存，受本机物理内存影响。    


## 三、 java对象的生命周期

创建对象（如果对象的类型没有加载则加载），使用对象，不可达，被标记，如果实现了finalize进入终结队列，回收

## 四、 Class对象是在方法区还是堆中

深入理解java虚拟机p215,Class是在方法区中，作为程序访问方法区的入口。这一点其实没有太大的意义，但是知道类的静态字段存在在方法区中，在类加载的准备阶段初始化内存是很重要的。


## 五、java对象的大小

对象由对象头，数据和对其对齐填充字节。主要考虑的是对象头，对象头包含一个指向对应Class的指针和对象信息。32位下，两者都是4字节，共8字节。64位下两者都是8字节共16字节，如果开启了指针压缩那么Class指针为4字节共12字节。如果是数组则多4个字节表明数组的长度。

## 六、 类加载的初始化阶段

在java并发编程思想中有个代码例子如下：

```java
public abstract class testAbstract {
    public testAbstract() {
        System.out.println("absStart");
        testChildImpl();
        System.out.println("absEnd");
    }

    public abstract void testChildImpl();
}
public class testAbstractImpl extends testAbstract {
    private String str = "123";

    public testAbstractImpl() {
        System.out.println(str);
        System.out.println("real");
    }

    public static void main(String[] args) {
        new testAbstractImpl();
    }

    @Override
    public void testChildImpl() {
        System.out.println(str);

    }
}

```
问输出的是什么？  答案：   
```
absStart
null
absEnd
123
real
```
以前在看这个问题的时候根本不知道答案，看了也觉得输出中的null非常难以理解，在重新看书的时候想到了其实就是关于类的初始化。上面的子类作为启动类需要初始化，但是根据加载的顺序，父类要先初始化，父类初始化的时候调用了子类的函数，这时子类输出静态字段，因为子类的静态字段在类加载的准备阶段已经分配了内存并且有默认值null，“123”要到子类初始化的时候才去赋值，所以输出为null。一切都解释通了，这也是我喜欢计算机的原因，一切都有原因。 （可以参考深入理解jvm的p219）


## 七、Class.forName和Classloader
区别在于Class.forName加载类的时候会经过加载阶段的初始化阶段。Classloader的loadClass根据传入resolve是否为true执行连接阶段（验证，准备，解析），始终不会执行初始化阶段。测试代码如下：

```java
  public class ClassLoadTest {
      public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, IOException {
          ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
          Class<? extends ClassLoader> aClass = ClassLoader.getSystemClassLoader().getClass();
          Method loadClass = aClass.getDeclaredMethod("loadClass", String.class, boolean.class);
          loadClass.setAccessible(true);
          loadClass.invoke(systemClassLoader, "jvm.classLoad.test.ClassOut", true);
          System.in.read();
          // Class.forName("jvm.classLoad.test.ClassOut");
      }
  }

```
另外补充下类加载器的一些重要方法：
1. loadClass 定义了双亲委派模型的框架
2. findClass 自己基本ClassLoader一般重写这个方法
3. findLoadedClass 找到当前类加载器加载的类 
4. defineClass  类加载的加载阶段（注意这个时候Class对象已经在内存中生成）
5. resolveClass 类加载的验证、准备、解析阶段

## 八、 类加载器的回收

很多场景都是讲的是类的回收，我在网上查的时候很少有谈到类加载的回收。类加载的回收三个条件都有讲到，其中需要类加载器被回收，那么类加载器什么时候能被回收呢？在这里的我的理解是当类加载器不可达时即可被回收（代码中就是将所有类加载器的引用消除，一般我们都在本地存有它的引用然后置为null，但是有很多意想不到的类加载器引用泄露的问题，例如序列化，log4j，ThreadLocal问题），注意类的回收是方法区，类加载器则是堆中。由此可以解释ThreadLocal导致tomcat内存泄露的问题（tomcat新版已经解决了这个问题https://wiki.apache.org/tomcat/MemoryLeakProtection）。对于这个问题有时间在Tomcat好好看下源码。网上看了很多关于类加载器泄露的坑，自己编写要十分谨慎。


## 九、 内存泄露

看了一篇[文章](https://juejin.im/post/5c6128c96fb9a049f36290ed)讲了内存泄露很不错，自己在公司遇到过一次连接泄露的问题，所以正好整理了下相关的问题来分享下。

场景描述：我们公司的用户服务对接了第三方腾讯云通信服务，在用户注册的时候我们需要走http接口调腾讯云，问题就出在http连接那块，同事当时采用了,线上出现了cpu100%的问题，日志出现`java.lang.OutOfMemoryError: GC overhead limit exceeded`。

排查思路：这个其实很好定位，本来还想打印线程栈看下到底是哪个导致的cpu100%，一看日志直接定位到gc出问题。GC overhead limit exceeded是指gc占用了大量的cpu时间又回收不了内存引起的，从内存泄露去考虑，重启服务 ，启动参数加上-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./user.hprof -verbose:gc -Xloggc:user%t.log。问题复现的时候获得了堆的dump文件，然后通过Jprofile分析，发现有大量的http.HttpKeepAliveCache实例，占用了80%的内存，大致定位到是由于http连接泄露。同事封装的HttpUtil中使用了HttpsURLConnection，在读取数据的时候没有关闭InputStream导致连接没有关闭。

## 十、String拼接导致内存溢出

公司的后台有段时间会间歇性的卡顿，严重的情况下会导致cpu100%。在cpu100%的时候，[通过top定位到进程号，然后输入H切换到线程，记住具体的进程号，使用jstack打印java进程的线程栈，jstack输出为十六进制，需要将top的转换成十六进制的然后入找线程经常卡在哪个方法](https://www.cnblogs.com/chenfangzhi/p/9981614.html)。定位到方法发现是查询用户关联设备号的方法出问题，方法的逻辑是从数据库查询设备号，在内存中以以逗号分隔拼接返回，如1,2,3。这个bug的原因是有如下：
> 1. sql出错，导致查询返回数据量很多，正常情况最多几百个，但是异常情况有七万个设备号
> 2.  字符串拼接采用str+="1234"的形式，导致大量的内存分配和回收。
> 运营在点击后台查询的时候发现没返回，点掉就重新点，导致服务器多个线程卡在这个方法造成cpu100%。解决完sql，改用StringBuilder问题解决。



   