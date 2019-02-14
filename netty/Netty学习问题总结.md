[TOC]

本篇记录了Netty学习过程中想到的问题和自己的一些思考，对于应用层的协议也有了更好的理解，所以在此做一个记录。

### 一、HTTP协议分包

TCP是作为面来流的协议，所以需要应用层协议自己去分包。常见的分包格式如下：   
1. 定长： 比如100字节每个报文，不足的前面补0，这时候每次取消息就取到100字节算整包)
2. 分隔符： 换行符其实是一种特别的分隔符，每次读取到分隔符就知道一个包读取完毕)
3. 指明长度： 比如前两个字节为长度字段，先读取两个字节，知道了需要多少字节，读取到对应的字节就是一个整包了

然而HTTP协议格式并不是上面的简单的一种，它结合了2和3两种来进行分包，因为请求和响应报文格式一样，所以这里针对请求报文进行说明。我们知道HTTP分为请求行，请求头，请求体。
下面是报文说明摘自[HTTP RFC文档](https://tools.ietf.org/html/rfc2616)中4.1 Message Types：


**HTTP报文格式**：
```
    generic-message = start-line
                      *(message-header CRLF)
                      CRLF
                      [ message-body ]
```
![HTTP报文格式](http://cdn.jiangmiantex.com/http%E6%8A%A5%E6%96%87.png)

**报文读取过程**

* 读取请求行：每个HTTP请求的第一行作为请求行，所以知道读取到CRLF就说明结束了
* 读取请求头：请求头有多行，行数不是固定的，他的结尾是根据连续的两个CRLF来判断的，在message-body之前会有一个CRLF
* 读取请求体：对于POST等带有请求体的方法来说，请求体的长度是不固定的，这时候请求头中会有个Content-Length字段说明了请求体的长度，所以只要读取完Content-Length个字节，整个HTTP请求报文也就得到了


**关于报文分割一点备注**：早期的HTTP 1.0时代，因为它每次请求都会经历tcp的三次握手连接过程，所以它是通过连接的关闭来判断报文已经读取完毕，但是这里还有一个问题，如果这个连接关闭时因为服务端的错误引起的那客户端就无法区分了。到了HTTP 1.1，因为很多请求会重用一个连接，所以需要用到Content-Length这个字段来做分包。另外还有一种不需要Content-Length的方法就是请求头中Transfer-Encoding为chunked，这是一种分块传输，在压缩传输，动态内容生成等响应在一开始长度未知的场景下很有用。他的报文分割也很简单，详情参见：[分块传输编码](https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)



## 二、WebSocket协议分包

理解了HTTP协议的分包，WebSocket的协议也容易理解，道理都是想通的。一开始在谷歌的时候一直搜索不到相关的报文，最后搜索WebSocket数据帧才搜到了结果(搜索是门技巧啊)。我最关注的是opcode字段，因为在用WebSocket的时候就用到了这个字段来判断是什么帧类型。第二是Payload len字段，这是一个变长字段，为了节省字节数，含义如下：

1. 如果数据长度小于等于125的话，那么该7位用来表示实际数据长度。
2. 如果数据长度为126到65535(2的16次方)之间，该7位值固定为126，也就是 1111110，往后扩展2个字节(16为，第三个区块表示)，用于存储数据的实际长度。
3. 如果数据长度大于65535， 该7位的值固定为127，也就是 1111111 ，往后扩展8个字节(64位)，用于存储数据实际长度。

摘录自：https://www.cnblogs.com/tugenhua0707/p/8542890.html，具体关于WebSocket可以查阅资料，有非常多的讲解。

[WebSocket RFC](https://tools.ietf.org/html/rfc6455#section-5.2)中websocket报文格式
```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

## 三、HTTP和WebSocket协议共用一个端口的问题

之前对这个现象十分不理解，很多Web服务器例如Tomcat都支持HTTP和WebSocket共用一个端口，他是怎么做到的？

其实理解了报文的解析就很容易理解了，HTTP和WebSocket协议的下层都是TCP连接，他们是应用层连接，所以在处理TCP的字节流时，可以先获取几个字节，如果前几个字节解析出来是GET，POST等HTTP协议的用到的，那么就根据HTTP报文的分包规则获取一个HTTP报文然后流转到后端处理。如果是WebSocket协议就根据WebSocket报文来解析然后做响应的处理。

补充：很多的协议栈都是以魔数打头，这样就更容易实现同端口多协议的支持，如Dubbo协议栈前两个字节是魔数，只需要判断报文的前两个字节就知道是不是Dubbo协议了，Dubbo协议栈头报文如下图(摘抄自http://ifeve.com/dubbo-protocol/)：

![Dubbo协议栈](http://cdn.jiangmiantex.com/Dubbo%E5%8D%8F%E8%AE%AE%E6%A0%88.png)




## 四、TIME WAIT状态占用了什么资源

我们知道TCP四次挥手时主动发起方会经历一个TIME WAIT状态，也正是因为这个原因我们尽量让客户端主动关闭连接。对于这个状态有人说是占用了文件描述符，有人说是端口，那么究竟是占用了什么资源？

根据我自己的实验，Windows系统下，TIME WAIT状态占用了端口，该端口不能作为客户端端口使用，但仍然可以作为服务端端口使用。实验端口：服务端8080，客户端6060 。情况如下：

1.  客户端主动关闭，客户端重启时报BindException，服务端用6060端口仍可正常启动
1.  服务端主动关闭，服务端重启正常，客户端重启也正常，但是如果停掉服务端8080，客户端用6060报BindException

实验代码如下，可根据需要自己修改试验：

**服务端主动关闭**：
```java 
public class ServerSocketCloseTest {
    public static void main(String[] args) throws IOException, InterruptedException {
        Runnable runnable = () -> {
            try {
                ServerSocket serverSocket = null;
                serverSocket = new ServerSocket(8080, 10);
                while (true) {
                    Socket accept = serverSocket.accept();
                    accept.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        };
        new Thread(runnable).start();
        Socket socket = new Socket(InetAddress.getLocalHost(), 8080, InetAddress.getLocalHost(), 6060);
        Thread.sleep(1000);
        socket.close();
        System.in.read();
    }
}
```

**客户端主动关闭场景**
```java
public class ClientSocketCloseTest {

    public static void main(String[] args) throws IOException, InterruptedException {
        Runnable runnable = () -> {
            try {
                ServerSocket serverSocket = new ServerSocket(8080, 10);
                while (true) {
                    Socket accept = serverSocket.accept();
                    Thread.sleep(1000);
                    accept.close();
                }
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        };
        new Thread(runnable).start();

        Socket socket = new Socket(InetAddress.getLocalHost(), 8080, InetAddress.getLocalHost(), 6060);
        socket.close();
        Thread.sleep(1000);
        System.in.read();
    }

}
```

结论：处于TIME WAIT状态下的端口不能作为客户端端口使用。对于服务端端口没有影响，服务端是仍然是可以正常绑定，接收到客户端连接后本地端口和监听端口是同一个所以不存在端口占用。另外通过查阅资料，TIME WAIT是释放了文件描述符，但是TCP连接的五元组并未释放，还占用一定的内存。参考地址如下：https://stackoverflow.com/questions/1803566/what-is-the-cost-of-many-time-wait-on-the-server-side/1806033#1806033


## 五、关于


待补充

