# 关于线上静态页面资源更新的一些经验分享

[TOC]

最近在负责公司的后台项目，包括了后端和前端。后端直接编译完打成jar包直接上线运行没什么问题。但是前端的页面文件更新每次都要把页面给运维，然后告诉运维路径让运维挨个替换，当然也可以整包替换，
但是如果文件比较多的情况下，整包替换就不合适了。因为现在开发的项目版本控制基本必不可少了，这时候可以利用版本控制软件来生成Patch文件，然后直接交给运维，让运维在项目根目录打补丁就行。

## 关于Linux的Patch
如果熟悉linux的话对patch命令肯定不陌生，linux很多软件代码的更新都用的是patch，相较于重新下载整份源码，patch文件体积小只需要更新改动的地方。patch本身和diff经常一起用的，具体的用法我就不详细介绍了，网上已经说明很多了。这里用来更新页面的思路其实和linux中的思想是一样的。
常用的的格式：

```bash
    patch -d /root/amqp -p0 -E  </root/amqp/0001-.patch
```

> -d 指明了patch的工作目录，此处指源码的根目录。     
> -pN中的N表示忽略路径中的第几层。如patch文件中路径为a/src/main/java/spittr/chen/AlertServiceImpl.java,如果需要忽略前面的a目录就需要指明-p1   
> -E表示如果文件更新之后为空则删除这个文件。   
> -R表示反向，回滚更新
> -b用于备份改动的文件，用于重要的操作

## 关于git

git作为最流行的版本控制软件也实现了diff和patch的功能，git导出patch有两种方式，git diff和git format-patch 。git diff用于生成通用的patch格式，而format-patch用于生成git专用的patch文件。不过我经过测试，两种生成的文件都是和linux的patch命令兼容的，也就是说两种生成的文件都可以直接使用patch命令来打补丁。不过这里有一个需要的需要注意的地方就是，git生成的patch文件中，在指明影响的文件路径时，默认原来的文件前面会加上路径a/,而改动后的文件前面会加上路径b/(我查看了git官网文档也有提及，但是不清楚为什么要默认这么写,如果有知道的希望能告知下),如下所示：    

```
diff --git a/src/main/java/spittr/chen/AlertServiceImpl.java b/src/main/java/spittr/chen/AlertServiceImpl.java
new file mode 100644
index 0000000..7c8ebd1
--- /dev/null
+++ b/src/main/java/spittr/chen/AlertServiceImpl.java
```

所以在linux中使用的时候需要指明参数-p1来忽略第一层路径，或者在使用git diff命令时加上参数<font color=red size=5>--no-prefix</font>。

其实git本身也有对应于linux patch的命令apply，还有更为强大的am。但是要求是必须是git项目。如果使用的git项目，推荐使用git的命令尤其是am命令，他必须使用format-patch生成的文件，format-patch文件携带了提交的记录包括作者，提交注释，日期等等，信息和pull差不多。关于这两个命令网上有很多的教程这里不再说明。对于我这个需求。已经因为线上的页面文件一般是文件夹的形式存在，所以使用linux的patch即可满足需求。

## 关于Idea

如果你用的开发工具是Idea的话，idea可以很方便的生成patch文件。如下图：

![](http://cdn.jiangmiantex.com/201809172245_845.png)

在版本的控制的log标签页，选中自己需要生成补丁的提交记录右键即可生成patch文件。