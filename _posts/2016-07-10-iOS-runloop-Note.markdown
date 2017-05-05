---
layout: post
title: "基础篇-RunLoop"
date: 2016-07-10 21:12:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
RunLoop 正如其名，loop表示某种循环，和run放在一起就表示一直在运行着的循环，在iOS应用中，随时处于待命状态的就是这个RunLoop，下面详细介绍RunLoop相关。

### 一、RunLoop <br>
**1、线程和RunLoop的关系<br>**
- 1.正如前面所说，RunLoop就是一个运行着的循环，实际上RunLoop和线程是紧密相连的，可以说RunLoop是为了线程而生，没有线程，它就没有了存在的必要。RunLoop是线程的基础架构部分，Cocoa和CoreFundation都提供了RunLoop对象方便配置和管理线程的RunLoop。每个线程，包括程序的主线程（main thread）都有与之对应的RunLoop对象。

- 2.在程序启动后会有一个main()函数，在·```UIApplicationMain()```这个方法会为main thread设置一个NSRunLoop对象，这就解释了为什么我们的应用在无人操作的时间休息，需要它干活的时候又能立马相应。

- 3.对其他线程来说，RunLoop默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，如果线程去执行一个长时间已确定的任务则不需要。

- 4.在任何一个cocoa程序的线程中，都可以通过``` NSRunLoop *RunLoop  = [NSRunLoop currentRunLoop];``` 得到当前线程的RunLoop。

**2、关于线程的几点说明**
- 1.Cocoa中的NSRunLoop类并不是线程安全的
> 我们不能再一个线程中去操作另一个线程的RunLoop，那样会造成意想不到的后果，但是CoreFundation中的不透明类CFRunLoopRef是线程安全的，而且两种类型的RunLoop是可以混用的，所以使用```-(CFRunLoopRef)getCFRunLoop;```获取CFRunLoopRef类来达到线程安全的目的。

- 2.RunLoop的管理并不是完全自动的
> 当我们的额程序中需要用到RunLoop，就可以设计线程代码在适当的时候启动RunLoop并正确响应事件。

- 3.RunLoop同时也负责autorelease pool 的创建和释放
> 每当一次运行循环结束的时候，它都会释放一次autorelease pool ，同时pool中的所有自动释放类型变量都会被释放掉。

- 4.RunLoop的优点
 一个RunLoop就是一个事件处理循环，用来不停的监听和处理输入事件并将其分配到对应的目标上进行处理。<br>
优点：
>
*首先，NSRunLoop是一种更加高明的消息处理模式，他的高明在对消息处理过程进行更好的抽象和封装，这样才能使我们不用处理一些很繁琐很底层的具体消息的处理，在NSRunLoop中的每一个消息被打包在input source或者timer source中。<br>
*其次，使用RunLoop可以使你的线程在工作的时候工作，没有工作的时候休眠，这样可以节省系统资源。


**3、RunLoop输入源**<br>
1.输入事件来源<br>
RunLoop 接收输入事件来自两种不同的来源： 输入源（input source）和定时源（timer source），两种源都是程序的某一特定的处理例程来处理到达的事件。
> 需要说明的是，当你创建输入源将其分配给RunLoop中的一个或多个模式，模式只会在特定事件影响监听的源。大多数情况下，RunLoop运行在默认模式下，但是你也可以运行在自定义模式，若某一源在当前模式下不被监听，那么任何其生成的消息只在RunLoop运行其相关联的模式下才会被传递

2.输入源（input source）<br>
传递异步事件，通常消息来自于其他线程或程序，输入源传递异步消息给相应的处理例程，并调用```runUntilDate:```方法退出（在线程里面相关的NSRunLoop对象调用）。<br>
输入源分类：
- 1.基于端口的输入源
> 基于端口的输入源由内核自动发送。Cocoa和Core Foundation内置支持使用端口相关的对象和函数来创建基于端口的源.<br>
1.例如，在Cocoa里面你从来不需要直接创建输入源。你只要简单的创建端口对象，并使用NSPort的方法把该端口添加到run loop。端口对象会自己处理创建和配置输入源。<br>
2.在Core Foundation ，你必须人工创建端口和它的RunLoop源，我们可以使用端口相关的函数（CFMachPortRef，CFMessagePortRef，CFSocketRef）来创建合适的对象。

- 2.自定义输入源
> 自定义的输入源需要人工从其他线程发送。<br>
为了创建自定义输入源，必须使用Core Foundation里面的CFRunLoopSourceRef类型相关的函数进行创建，你可以使用回调函数来配置自定义输入源，Core Foundation 会在配置源的不同地方调用回调函数，处理输入事件，在源从run loop移除的时候清理它。

- 3.Cocoa上的selector源
> 除了基于端口的源，Cocoa定义了自定义输入源，允许你在任何线程执行selector方法，和基于端口的源一样，selector请求会在目标线程上序列化，减缓许多在线程上允许多个方法容易引起的同步问题，不像基于端口的源，每个selector执行完成后自动从RunLoop里面移除。

3、定时源(timer source)<br>
定时源在预设的时间点以同步方式传递消息，这些消息都会发生在特定或者重复的时间间隔，定时源则直接传递消息给处理例程，不会立即退出RunLoop。<br>
注意:
> 尽管定时器可以产生基于时间的通知，但它并不是实时机制，和输入源一样，定时器也和RunLoop的特定模式有关。如果定时器所在的模式当前未被RunLoop监视，那么定时器将不会开始直到RunLoop运行在相应的模式下。类似的，如果定时器在RunLoop处理某一事件期间开始，定时器会一直等待直到下次RunLoop开始相应的处理程序。如果RunLoop不再运行，那定时器也将永远不启动。

**4、RunLoop观察者**<br>
&emsp;&emsp;源是在合适的同步或异步时间发生时触发，而RunLoop观察者则是在RunLoop本身运行的特定时候触发，你可以使用RunLoop观察者来处理某一特定事件或者是进入休眠的线程做准备。<br>
RunLoop观察者和以下事件关联：
>
- 1.RunLoop入口
- 2.RunLoop 何时处理一个定时器
- 3.RunLoop何时处理一个输入源
- 4.RunLoop何时进入休眠状态
- 5.RunLoop何时被唤醒，但在唤醒之前要处理的事件
- 6.RunLoop终止

**5、RunLoop的事件队列**<br>
每当运行RunLoop，你线程的RunLoop会自动处理之前未处理的消息，并通知观察者。具体顺序如下：
>
- 1.通知观察者RunLoop已经启动
- 2.通知观察者任何即将要开始的定时器
- 3.通知观察者任何即将启动的非基于端口的源
- 4.启动任何准备好的非基于端口的源
- 5.如果基于端口的源准备好并处于等待状态，立即启动，并进入步骤9
- 6.通知观察者线程进入休眠
- 7.将线程置于休眠直到任意下面的事件发生：
   * 某一事件到达基于端口的源
   * 定时器启动
   * RunLoop设置的时间已经超时
   * RunLoop被显示唤醒
- 8.通知观察者线程将被唤醒
- 9.处理未处理的事件
   * 如果用户定义的定时器启动，处理定时器事件并重启RunLoop，进入步骤2
   * 如果输入源启动，传递相应消息
   * 如果RunLoop被显示唤醒而且时间还没超时，重启RunLoop，进入步骤2
- 10.通知观察者RunLoop结束

从这个事件队列可以看出：
>
① 如果是事件到达，消息被传递给相应的处理程序来处理，RunLoop处理完当次事件后RunLoop会退出，而不管之前预定的时间到了没有，你可以重启RunLoop来等待下一事件。<br>
② 如果线程中有需要处理的源，但是响应的事件没有到来的时候，线程就会休眠等待相应事件的发生。这就是为什么run loop可以做到让线程有工作的时候忙于工作，而没工作的时候处于休眠状态。

**6、什么时候使用RunLoop**<br>
仅当在为你的程序创建辅助线程的时候，你才显式运行一个RunLoop。RunLoop是程序主线程基础建设的关键部分。所以Cocoa提供了代码运行主程序的循环并自动启动RunLoop。<br>
RunLoop在你要和程序有更多的交互时才需要，比如下列情况：
>
- 1.使用端口或自定义输入源和其他线程通信
- 2.使用线程的定时器
- 3.Cocoa中使用任何的performSelector方法
- 4.使线程周期性工作

**7、RunLoop Mode**<br>

![RunLoopMode](/image/RunLoopMode.png "RunLoopMode一览图")

如图所示，RunLoop Mode实际上是Source ，Timer和Observer的集合,不同的Mode把不同组的Source ，Timer和Observer隔绝开来。RunLoop在某个时刻只能跑一个Mode，处理一个Mode当中的Source ，Timer和Observer。<br>
苹果文档中提到的Mode有五个，分别是：
>
- 1.NSDefaultRunLoopMode(kCFRunLoopDefaultMode)：RunLoop 的默认 Mode，通常主线程在这个 Mode 下运行。
- 2.NSConnectionReplyMode
- 3.NSModalPanelRunLoopMode
- 4.NSEventTrackingRunLoopMode
- 5.NSRunLoopCommonMode(kCFRunLoopCommonModes)：这是一个占位 Mode，不是一个真正的 Mode。一个模式可以被标记为 NSRunLoopCommonMode。默认情况下，NSDefaultRunLoopMode 和 UITrackingRunLoopMode 被标记为 NSRunLoopCommonMode，RunLoop 在这个模式下运行，则表示 RunLoop 可以同时执行在 NSDefaultRunLoopMode 和 UITrackingRunLoopMode 两个模式下。

PS: iOS 中公开暴露出来的只有 NSDefaultRunLoopMode 和 NSRunLoopCommonModes。 <br>
注意点：
>
* 1.一个RunLoop对象可以包含多个模式，每个模式可以包含多了Source、Observer、Timer，可以监听多个对象
* 2.RunLoop 只能选择一种模式运行，这个Mode就是currentMode
* 3.如果需要切换Mode，只能先退出RunLoop，再重新指定一个Mode进入，这样为了分割不同Mode的Source，Timer，Observer，使它们互不影响。
* 4.一个RunLoop当店Mode没有任何的 Source，Timer，Observer，则RunLoop直接退出。

扩展Mode：
>
- UITrackingRunLoopMode：界面追踪 Mode，用于 UIScrollView 追踪，触摸滑动，保证界面动画不受其他Mode影响
- UIInitializationRunLoopMode：在刚启动APP时进入的第一个 Mode，启动完成后就不再使用。（这个模式主要是苹果在用，开发者用不到）
- GSEventReceiveRunLoopMode：接受系统事件的内部 Mode（绘图事件），通常开发者用不到。


**8、与RunLoop相关的坑**<br>
日常开发中，与 RunLoop 接触得最近可能就是通过 NSTimer 了。一个 Timer 一次只能加入到一个 RunLoop 中。我们日常使用的时候，通常就是加入到当前的 RunLoop 的 default mode 中，而 ScrollView 在用户滑动时，主线程 RunLoop 会转到 UITrackingRunLoopMode（UITrackingRunLoopMode：界面追踪 Mode，用于 UIScrollView 追踪，触摸滑动，保证界面动画不受其他Mode影响） 。而这个时候， Timer 就不会运行。<br>
解决办法：
>
- 第一种： 设置RunLoop Mode，例如NSTimer,我们指定它运行于 NSRunLoopCommonModes ，这是一个Mode的集合。注册到这个 Mode 下后，无论当前 RunLoop 运行哪个 mode ，事件都能得到执行。
- 第二种：另一种解决Timer的方法是，我们在另外一个线程执行和处理 Timer 事件，然后在主线程更新UI。

### 二、RunLoop相关知识点<br>
**1、RunLoop的Mode作用是什么？<br>**
Mode主要是用来指定事件在运行循环中的优先级 ，详细了解同上。

**2、猜想runloop内部是如何实现的？**<br>
> 一般来说，一个线程一次只能执行一个任务，执行完任务后线程就会退出。如果我们需要一个机制，让线程随时处理事件但并不退出。

伪代码显示如下:

---
int main(int argc, char * argv[]) {<br>
     //程序一直运行状态<br>
     while (AppIsRunning) {<br>
          //睡眠状态，等待唤醒事件<br>
          id whoWakesMe = SleepForWakingUp();<br>
          //得到唤醒事件<br>
          id event = GetEvent(whoWakesMe);<br>
          //开始处理事件<br>
          HandleEvent(event);<br>
     }<br>
     return 0;<br>
}

***

**3、objc使用什么机制管理对象内存？** <br>
&emsp;&emsp; 通过retainCount（引用计数器）机制来决定对象是否需要释放，每次runloop的时候，都会检查对象的retainCount，如果retainCount为0，那么久说明该对象没有地方需要使用了，可以释放掉了。

**4、ARC通过什么方式帮助开发者管理内存？**<br>
简答的理解就是： 编译时根据代码上下文，插入retain/release<br>
解释：
>  ARC相对于MRC，不是在编译时添加retain/release/autorelease这么简单，而是在编译器和运行期两部分共同帮助开发者管理内存。<br>
在编译器，ARC用的是底层的C接口实现retain/release/autorelease，这样做性能更好，也是为什么在ARC不能手动retain/release/autorelease，同时对同一上下文的同一对象的成对retain/release操作进行优化；ARC也可以包含运行期组件。

**5、不手动指定autoreleasepool的前提下，一个autorelease对象什么时候释放？**<br>
分两种情况：手动干预释放时机、系统自动去释放
>
- 1.手动干预释放时机--指定autoreleasepool就是所谓的（当前作用域大括号结束时释放）。
- 2.系统自动去释放--不手动指定autoreleasepool

autorelease对象出了作用域之后，会被添加到最近一次创建自动释放池中，并会在当前runloop迭代结束时释放。<br>
释放时机如下下图所示：
![图片](/image/autoreleasePool.png "Autorelease释放时机图")

释放时机解释：
>从程序启动到加载完成是一个完整的运行循环，然后会停下来，等待用户交互，用户的每一次交互都会启动一次运行循环，来处理用户所有的点击事件、触摸事件。<br>
- 什么时候执行释放动作？ <br>
  在一次完整的运行循环结束之前，会被销毁
- 什么时候会创建自动释放池？
  运行循环检测到事件并启动后，就会创建自动释放池
- 子线程的runloop默认是不工作的，无法主动创建，必须手动创建
- autoreleasepool当自动释放池被销毁或者耗尽时，会向释放池中的所有对象发送release消息，释放释放池中的所有对象。
- 如果一个vc的viewDidLoad中创建一个Autorelease对象，那么该对象会在viewDidAppear方法执行之前被销毁。

**6、BAD_ACCES在什么情况下出现？** <br>
访问了野指针，比如对一个已经释放的对象执行了release，访问已经释放对象的成员变量或消息。死循环。

**7、Apple是如何实现autoreleasepool的？** <br>

autoreleasepool以一个队列数组的形式实现，主要通过三个函数完成：
- 1.```objc_autoreleasepoolPush```
- 2.```objc_autoreleasepoolPop```
- 3.```objc_autorelease```
看函数名就知道，对autorelease分别执行push 和pop操作。销毁对象时执行release操作。


---
参考资料：<br>
[@iOS程序犭袁的Git分享](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#14-synthesize%E5%90%88%E6%88%90%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E7%9A%84%E8%A7%84%E5%88%99%E6%98%AF%E4%BB%80%E4%B9%88%E5%81%87%E5%A6%82property%E5%90%8D%E4%B8%BAfoo%E5%AD%98%E5%9C%A8%E4%B8%80%E4%B8%AA%E5%90%8D%E4%B8%BA_foo%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E9%82%A3%E4%B9%88%E8%BF%98%E4%BC%9A%E8%87%AA%E5%8A%A8%E5%90%88%E6%88%90%E6%96%B0%E5%8F%98%E9%87%8F%E4%B9%88) <br>
[Objective-C之run loop详解](http://blog.csdn.net/wzzvictory/article/details/9237973)
