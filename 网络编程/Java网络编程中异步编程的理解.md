##前言##
这篇文章主要是总结自己对于网络编程中异步，同步，阻塞和非阻塞的理解，这个问题自从学习nio以来一直困扰着我，，其实想来很久就想写了，只不过当时理解不够，无从下手。最近在学习vertx框架，又去熟悉了下netty的代码，因为了对于多线程也有了更深的理解，所以才开始对于这些概念有了理解，用于理清思路。


## 一、异步，同步，阻塞和非阻塞的理解


## 二、为什么使用异步


## 三、异步编程从用户层面和框架层面不同角度的理解


## 四、ComplablFuture，Netty和Vertx中异步的应用

## 五、理解这些能在实际中用到吗


Future 模式

Netty is an asynchronous event-driven network application framework 
for rapid development of maintainable high performance protocol servers & clients.



以前在学习c++中muduo只是记得陈硕说的epoll是一个同步非阻塞的模型


参考文章：

1. [回调地狱的今生前世](https://juejin.im/entry/57fa6a4e67f3560058752542)
2. [怎样理解阻塞非阻塞与同步异步的区别？ - 严肃的回答 - 知乎](https://www.zhihu.com/question/19732473/answer/20851256)
3. [怎样理解阻塞非阻塞与同步异步的区别？ - 陈硕的回答 - 知乎](https://www.zhihu.com/question/19732473/answer/26091478)
4. [netty官网](https://netty.io/)
5. [IO - 同步，异步，阻塞，非阻塞 ](https://blog.csdn.net/historyasamirror/article/details/5778378)
6. [nodejs真的是单线程吗?](https://segmentfault.com/a/1190000014926921)
7. [作为一个服务器，node.js 是性能最高的吗？ - 圆胖肿的回答 - 知乎](https://www.zhihu.com/question/35280583/answer/487808916)
6. Java8实战第11章和java并发编程实战


[1]:https://www.zhihu.com/question/35280583/answer/487808916