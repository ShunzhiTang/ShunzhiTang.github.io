---
layout: post
title: "基础篇-LLDB调试"
date: 2018-05-20 22:20:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
LLDB是一个有着REPL的特性和C++，Python插件的开源调试器。LLDB绑定在Xcode内部，存在于主窗口底部的控制台中。调试器允许你在程序运行的特定时机暂停它，你可以查看变量的值，执行自定的指令，并且按照你所以为合适的步骤来操作程序的进展。在iOS的开发中，调试是一个非常重要的功能，有时间不需要重复的运行程序就可以指定我们的调试过程，这时候就需要使用LLDB来实现。

### 一、LLDB的简单使用
**1、LLBD的命令使用**<br>
1.help 命令
>
最简单的命令就是help命令，它会列举出所有的命。如果你忘记一个命令是做什么的，或者想知道更多的话，可以通过```help <command>```了解更多的细节，例如help print 或者help thread。如果你甚至忘记了help命令是做什么的。可以使用help help。

2.print
>
LLDB实际上会作前缀匹配，所以你可以使用prin ，pri 或者p。但是你不能使用pr，因为LLDB不能消除和process的歧义。<br>
一般情况使用print这个命令，结果中有个$0 ,实际上你可以使用它来指向这个结果。任何以$美元符号开头的东西都是存在于LLDB的命名空间的，他们是为了帮助你进行调试而存在的。

3.expression
>
expression 不仅会改变调试器中的值，实际上它也会改变程序中的值，这时候继续执行程序，就会替换成改变后的值

一般请求下都是使用p来代替print ，e来代替expression

4.breakpoint 断点
>
所有的调试都是从断点开始的，一般情况我们队breakpoint命令用的不多，而是在Xcode的GUI界面中直接添加断点的。<br>
```
Add Exception Breakpoint
```
为异常断点，一般都会在debug的情况下添加一个，可以看到异常的位置。
```
breakpoint set
```
可以在运行的时候给你想要添加断点的地方增加断点。<br>
 查看程序中所有的断点，使用```breakpoint list```


5.watchpoint 观察设置点
>
作为断点的补充，LLDB支持观察点以在不中断程序运行的情况下监测一些变量。
```
watchpoint set self->testVar     //为该变量地址设置watchpoint
watchpoint set expression 0x00007fb27b4969e0 //为该内存地址设置watchpoint，内存地址可从前文提及的`p`命令获取
watchpoint command add -o 'frame info' 1  //为watchpoint 1号加上子命令 `frame info`
watchpoint list //列出所有watchpoint
watchpoint delete // 删除所有watchpoint
```

6.thread、bt、frame   线程和堆栈
>
bt 即是thread backtrace，作用是打印出当前线程的堆栈信息。当线程发生crash后，我们可以用该命令打印出发生crash的当前程序堆栈，查询出发生crash的调用路径。<br>
thread另一个常用的用法就是 thread return，调试的时候，我们希望在当前执行的程序堆栈直接返回一个自己想要的值，可以执行该命令直接返回。
frame 即是帧，其实就是当前的程序堆栈，我们输入bt命令，打印出来的就是当前线程的frame。在调试中，一般我们比较关心当前堆栈的变量值，我们可以使用frame variable来获取全部变量值。当然也可以输入特定变量名，来获取单独的变量值，如frame v self->testVar来获取testVar的值。如果想看另外一帧，可以使用frame select

7.image
>
image list 可以查看工程中使用的库<br>
image lookup --address 可以根据执行文件的地址找到程序具体的位置，对于奔溃调试很重要。<br>


LLDB是一个很强大的调试工具，在开发的过程中需要经常使用，提高我们的开发效率，以上只是简单的命令介绍，因为网上有很全的参考资料，就不在赘述；LLDB解析[《与调试器共舞 – LLDB 的华尔兹》](https://objccn.io/issue-19-2/),和南峰子的技术博客[《LLDB调试器使用简介》](http://southpeak.github.io/2015/01/25/tool-lldb/)这两篇文章很详细的介绍了LLDB的使用和原理，值得一看。


### 二、Chisel-LLDB命令插件
学习了上面的LLDB的基本调试技巧，我们下面来看一个强大的第三方插件Chisel，Chisel是facebook开源的LLDB插件，可以让调试更加的容易。

**1、安装** <br>
源代码地址：[Chisel](https://github.com/facebook/chisel),Chisel 使用homebrew来安装。<br>

如果没有安装homebrew，请看下面步骤 ，安装过的，请忽略
>
打开MAC终端，输入下列命令:
```
brew update
brew install chisel
```
安装完成按照安装日志上的提示，在~/.lldbinit文件中添加一行，没有则新建。 提示类似如下：
```
==> Caveats
Add the following line to ~/.lldbinit to load chisel when Xcode launches:
  command script import /usr/local/opt/chisel/libexec/fblldb.py
```
注意这里的地址（/usr/local/opt/chisel），需要你自己看清楚<br>
做好上面的步骤就可以重启Xcode来使用了。


**2、内置命令** <br>
***2.1. pviews***
>
这个命令可以递归打印所有的view，并能标示层级，相对于UIView的私有辅助方法```view recursiveDescription```  。使用pviews可以在调试定位时省去很多麻烦。

***2.2. pvc***
>
这个命令也是递归打印层级，但不是view，而是viewController。利用它我们对viewController的结构也是一目了然。

***2.3. visualize***
>
这个命令可以让你使用Mac的预览打开一个UIImage、CGImageRef，UIView、或CALayer。这个功能或许可以帮助我们来截图、用来定位一个view具体内容，但是这个功能现在只是在模拟器好使，在真机不好使。

***2.4. fv & fvc***
>
fv 和 fvc 这两个命令是用来通过类名搜索当前内存中存在的view和viewController实例的命令，支持正则搜索。

***2.5. show & hide***
>
这两个命令用来显示和隐藏一个指定的UIView ,你甚至不需要Continue Progress就可以看到效果。

***2.6. mask/umask  border/unborder***
>
这两组命令用来标识一个view或layer的位置时用，mask用来在view上覆盖一个半透明的矩形，border可以给view添加边框。但是实际使用的过程中mask一般会有问题。mask/unmask 一般不使用。用border命令是一样的效果。反正二者的用途都是找到一个对应的view。

***2.7. caflush***
>
这个命令会重新渲染，即可以重新绘制界面，相当于执行了```[CATransaction flush]``` 方法，要注意如果在动画过程中执行这个命令，就直接渲染动画结束的效果。<br>
当你想在调试界面颜色、坐标之类的时候，可以直接在控制台修改属性，然后caflush 就可以看到效果啦，很方便省事。<br>
例子，给其中的$122即目标UIView使用caflush
```
(lldb) p view
(long) $122 = 140718754142192
(lldb) e (void)[$122 setBackgroundColor:[UIColor greenColor]]
(lldb) caflush
```

***2.8. bmessage***
>
这个命令是用来打断定用的，一般情况我们都是在GUI界面中直接打断定的，但是如果想在没有实现的方法中打断点剧需要重新实现这个方法，然后打断点，然后rebuild。这样很麻烦。那么bmessage就是解决这个问题的。<br>
比如我们没有实现```viewWillAppear:```方法，但是要在它里面打断点，使用方法 ```bmessage -[MyViewController viewWillAppear:]``` 上面的命令会在其父类的```viewWillAppear:```方法中打断点，并添加上条件```[self isKindOfClass:[MyViewController class]]```





---
参考资料：<br>

[LLDB调试器使用简介](http://southpeak.github.io/2015/01/25/tool-lldb/)<br>
[与调试器共舞 – LLDB 的华尔兹](https://objccn.io/issue-19-2/)<br>
[Chisel-LLDB命令插件，让调试更Easy](https://blog.cnbluebox.com/blog/2015/03/05/chisel/)
