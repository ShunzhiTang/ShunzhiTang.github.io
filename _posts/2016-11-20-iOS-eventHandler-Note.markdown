---
layout: post
title: "基础篇-事件处理"
date: 2016-11-20 22:20:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
对于iOS设备用户来说，他们操作设备的方式主要有三种：触摸屏幕，晃动设备，通过遥控设施控制设备。对用的事件类型有以下三种：
- 1.触屏事件（Touch Event）
- 2.运动事件（Motion Event）
- 3.远端控制事件（Remote-Control Event）

### 一、响应者链(Responder Chain) <br>
当发生事件响应时，必须知道谁响应事件，在iOS中，由响应链来对事件进行响应。

所有事件响应的类都是USResponder的子类，响应者链是一个由不同对象组成的层次结构，其中的每个对象将依次获得响应事件消息的机会。当发生事件时，事件首先被发送给第一响应者，第一响应者往往是事件发出的视图，也就是用户触摸的地方。事件将沿着响应链一直向下传递，知道被接受并作出处理，一般来说，第一响应者是个视图对象或者其子类对象，当其被触摸后事件被交由它处理，如果它不处理，事件就会被传递给它的视图控制器对象ViewController（如果存在），然后是它的父视图（superView）对象（如果存在），一次类推，知道顶层视图。接下来会沿着顶层视图（top view）到窗口（UIWindow对象）再到（UIApplication对象）。如果整个过程中都么有响应这个事件，该事件就被丢弃。一般情况下，在响应者链中只要有对象处理事件，事件就会停止传递。

一个典型的事件响应路线如下：
>
First Responser --> The Window --> The Application --> nil（丢弃）

我们通过[reponder nextResponder]找到当前responser的下一个responder，持续这个过程最后找到UIApplication对象。通常情况下，我们在First Responder（一般就是当前用户触控的view）这里就会响应请求，进入事件分发机制。


### 二、事件分发
第一响应者 （First responder）指的是当前接受触摸的响应者对象（通常一个UIView对象），即表示当前该对象正在与用户交互，它是响应者链的开端。响应者链和事件分发的使命都是找出第一响应者。

iOS系统检测到手指触摸（Touch）操作时会将其打包成一个UIEvent对象，并放入当前Application 的事件队列，单例的UIApplication会从事件队列中取出触摸事件并传递给单例UIWindow来处理，UIWindow对象首先会使用 ```hitTest:withEvent:```方法寻找此次Touch操作初始化所在的视图（UIWindow），即需要将触摸事件传递给其处理的视图，这个过程称之为hit -test view。

// 寻找并返回最合适的view(能够响应事件的那个最合适的view)

```hitTest:withEvent:``` 方法的处理流程入下：
>
- 1.首先调用当前视图的```pointInside:withEvent:```方法判断触摸点是否在当前视图内。
- 2.若返回NO，则```hitTest:withEvent:```返回nil，若返回YES，则向当前视图的所有子视图（subViews）发送```hitTest:withEvent:```消息，所有的子视图的遍历顺序是从最顶层视图一直到最底层视图，即subviews数组的末尾向前遍历，直到有子视图返回非空对象或者全部子视图遍历完毕。
- 3.若第一次有子视图返回非空对象，则```hitTest:withEvent:```方法返回此对象，处理结束。
- 4.如所有子视图都返回空，则```hitTest:withEvent:```方法显示自身self。



### 三、事件处理注意和说明 <br>

1.如果最终hit-test 没有找到第一响应者，或者第一响应者没有处理该事件，则该事件会沿着响应者链向上回朔，如果UIWindow实例和UIApplication实例不能处理该事件，则该事件会被丢弃。

2.```hitTest:withEvent:```方法在以下情况会被忽略（不起作用）：
>
- 2.1 视图被隐藏，一般表现为hidden = YES
- 2.2 禁止用户操作(userInteractionEnabled=NO) 的视图
- 2.3 alpha级别小于0.01（一般0.0-<0.01表示透明）
- 2.4 如果一个子视图的区域超过父视图的bound区域（父视图的clipsToBounds属性为NO，这样超过父视图的bound区域子视图也会显示），那么正常情况下对于子视图在父视图之外的区域触摸是不会被识别的，因为父视图的```pointInside:withEvent:```方法会返回NO，这样就不会继续向下遍历子视图了。当然我们可以重写```pointInside:withEvent:```方法来处理这种情况。

3.可以通过重写```hitTest:withEvent``` 来达到有些特定的需求。可以参考[CYLTabBarController](https://github.com/ChenYilong/CYLTabBarController)是一个支持自定义 Tab 控件的开源项目。实现了超出TabBar frame也可以响应。


**总结：**<br>
当手指触摸（touch）操作时，iOS会将其打包成一个UIEvent对象，并加入Application的时间队列，touch事件就会沿着(UIApplication -> UIWindow ->view)去寻找最合适响应的view，如果当前的view不响应这个事件 ，则需要按照响应链（responser ->UIWindow ->UIApplication ->nil）去找到响应对象，实现这个事件的响应。如果到最后没有响应就丢弃。



---
参考资料：<br>

[@iOSGit资料](https://hit-alibaba.github.io/interview/iOS/Cocoa-Touch/Event-Handling.html)<br>
