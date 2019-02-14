# Idea Live Template总结

[TOC]

live template是idea中提高效率的利器之一，以前看过一些教程，平时经常在使用，减少了我很多繁复的工作，但是没有系统的去整理过，最近准备系统的整理下，主要是自己平时用到的和官方文档的说明，如果有不正确的地方。   

**定义**：  Live template可以让你快速、高效、正确的插入平时经常使用的或者自定义的代码片段。

## 一、演示 
在开始之前，为了引起大家的兴趣，我们先来看下使用live template的演示效果：

![](http://pewaccq76.bkt.clouddn.com/201809132225_152.gif)

上面使用的就是live template，其中有预定义的sout,psvm和fori，还有自定义的todo和log等。可以看到使用live template可以用缩略词产出设置好的代码片段。

## 二、详细介绍

### 2.1 类型
> live template一共有三种类型分别是简单、参数化和环绕类型。   

1. **简单类型**   
简单类型就是固定的代码片段，当通过缩略词展开的时候，会在源代码中展开。如最简单和常用的pdvm展开就是main函数的定义。   
2. **参数类型**
参数类型就是代码片段中带有参数的模板，参数用$界定，如参数MY，这位$MY$,参数类型非常有用，我们自定义的模板很多都会用到参数，等会再设置中在进行讲解。      
2. **环绕类型**
环绕模板指的是那种包裹代码块的模板，比如try catch，还有下面演示的callable语句。
三种类型的演示如下：   

![](http://pewaccq76.bkt.clouddn.com/201809132302_521.gif)

### 2.2设置（win默认快捷键win+alt+s）
路径如下图箭头1处：  

![](http://pewaccq76.bkt.clouddn.com/201809142040_551.png)

如图中所示，iterations是idea自带的group，fori是缩略词，顾名思义这个组是针对迭代等操作的。
在使用时我们可以输入10.fori，list.fori或者直接输入fori然后按tab键(箭头7处)插入代码。idea会根据上下文生成不同的代码片段，如10.fori直接生成了“for (int i = 0; i < 10; i++) {”，而直接输入fori则是“for (int i = 0; i < ; i++) {”，注意此时10没有自动生成需要你手动输入。    
我们可以点击2处新建自己的template，template的缩略词在同一group内不能重复，所以为了不和自带的键重复我们最好新建自己的一个group比如MY，不同的group中的缩略词可以重复。箭头5是描述用来助记的。   
我们来自定义一个如下图：   

![](http://pewaccq76.bkt.clouddn.com/201809142039_514.png)

图中是一个非常常用的输入，根据类名来生成log静态变量。你可以看到用$包裹的字符，这个就是上面介绍的参数类型，在生成的代码片段中，如果要输出$，需要用$转义，即输出$则在代码片段中输入$$。系统自带两个预定义的只读变量，$END$和$SELECTION$，$END$代表代码片段展开后光标最终停留的位置(如果有用户自定义变量且需要用户输入的话则会一次停留在用户变量处)，默认如果不写$END$光标会停留在最后，如果加和不加效果是一样的。$SELECTION$代表的是你用光标选中的所有字符，属于环绕类型，等会用例子会很明白。用户变量需要我们赋值，点击edit variables，在箭头2处进行编辑。可以输入两种，一种是直接输入字符串(需要用双引号包裹)用的比较少因为是写死的，另一种是idea的预定义函数(即通过下拉菜单选择)，比如这里就是取类名。idea有很多预定义的函数，比如日期，行数，方法名，作者等等。一般用到这些预定义的函数就已经足够了，但是有时复杂的输出，就需要使用groovy脚本(下拉菜单groovyScript，这里需要用到的语法很简单)来进行。比如输入方法的所有参数，如下图：

![](http://pewaccq76.bkt.clouddn.com/201809172315_237.gif)

我自定义了一个info(代码片段："$CLASS$.$METHOD$  linenum:$LINE$, param:{$PARAM$} info:$MY$"$END$)，输出了类名、方法名、行数和参数，这些信息在记录日志的时候非常有必要。其中$PARAM$变量就用到了脚本。我们来看下

```groovy
// methodParameters是预定义函数，其中双引号里的就是脚本，_1占位符只带methodParameters参数
groovyScript("_1.collect { it + ' = [\" + ' + it + ' + \"]'}.join(', ') ", methodParameters())
```

关于备份和分享：live template文件保存在“{user}\\{version}\config\templates”,user是指用户目录，version是idea目录，如我的目录就是C:\Users\chen\.IntelliJIdea2017.3\config\templates，其中的文件名以group为名字。也可以在在File->Export Settings对话框中选中live template可以保存配置。
**说明**：在设置变量的值时有一列是Skip if define，这一列的意思是，如果有值了是否跳过(即光标是否停留)，光标停留的位置是变量对话框中的顺序来定的，可以用右边的箭头排序。如果所有的变量填充完了便会跳到$END$变量的位置，如果没有定义$END$则跳到代码片段结尾。

### 2.3 快捷键

win平台默认的快捷键主要是三个ctrl+j(insert live template)、ctrl+alt+j(sround with live template)和ctrl+alt+t(sround with)。     

![](http://pewaccq76.bkt.clouddn.com/201809142044_775.png)

快捷键是live template中经常需要用到的，所以需要记住。由于每个平台不一样，也有可能有人修改了快捷键，所以我用括号注明了快捷键对应的名字，如果你的idea该快捷键不生效可以直接按图中搜索名字。

 * ctrl+j：插入普通的live template
 * ctrl+alt+j:插入包裹的live template
 * ctrl+alt+t:插入包裹的代码片段，这个包含了ctrl+alt+j但是又包含一些系统自带的语句块，比如if，while和for等等。

**这里就需要重点介绍下包裹的代码片段**，其实就是指的你用光标选中的代码。使用这种代码片段需要我们用光标去选择然后输入快捷键ctrl+alt+t或者ctrl+alt+j选中需要的使用的缩略词。在自定义的代码片段中有个自带的$SELECTION$指的就是你用光标选中的代码，在插入代码片段时，就会将你选中的代码插入到$SELECTION$。让我们在实现一个带包裹代码片段的sloge，设置如下:

![](http://pewaccq76.bkt.clouddn.com/201809142100_646.png)

**注意设置中箭头的位置，选择java**，表示快捷键应用的上下文。   
使用方法： 用鼠标选中代码，输入ctrl+alt+j或者ctrl+alt+t选择sloge。如下图：

![](http://pewaccq76.bkt.clouddn.com/201809142107_36.gif)

### 2.4 实战
我自定义了几个非常常用的代码片段，分别是
* "info":输出调试信息   
```
    // 代码片段      
    "$CLASS$.$METHOD$  linenum:$LINE$, param:{$PARAM$} info:$MY$"$END$      
    // 变量定义     
    $CLASS$：className()      
    $METHOD$：methodName()   
    $LINE$：lineNumber()   
    $PARAM$：groovyScript("_1.collect { it + ' = [\" + ' + it + ' + \"]'}.join(', ') ",      methodParameters())   
```
* "fen"：分割线的注释   
```
    // 代码片段 
    /* ---------------- $E$ -------------- */$END$
```
* "log":定义日志常量
```
    // 代码片段 
    private static final Logger logger= LoggerFactory.getLogger($CLASS$.class);
    // 变量定义     
    $CLASS$：className()  
```
* "zhushi"：带名字和日期的注释
```
    // 代码片段 
    // comment --$USER$-- $D$ ------>$ANNOTATION$
    // 变量定义     
    $USER$："chenfangzhi"    
    $D$ ：date("YYYY-MM-DD hh:mm:ss")
```
* "todo"：todo注释
```
    // 代码片段 
    // todoBy$USER$ ---- $D$ ------>$TODO$
    // 变量定义     
    $USER$："chenfangzhi"    
    $D$ ：date("YYYY-MM-DD hh:mm:ss")
```

**说明**:todo的作用我就不讲解了，这里的第4和第5项可能很像，有很多地方需要标注是谁操作的，现在的项目很多都是多人开发，如果都是使用默认的todo，就会很混乱，这时候我们就需要自己来定义属于自己的todo注释，这时候就需要带上名字。代码片段可以自己定义，可以同时带上todo和名字，这样在查看todo列表的时候就可以进行筛选。如下图：

![](http://pewaccq76.bkt.clouddn.com/201809142158_610.png)

图上有两个todo，在todo列表中可以点击箭头2处的过滤器筛选自己想要的看到的类型。我就是直接看chen这个类型。2处有个Edit Filter可以编辑过滤类型，很简单的正则匹配。