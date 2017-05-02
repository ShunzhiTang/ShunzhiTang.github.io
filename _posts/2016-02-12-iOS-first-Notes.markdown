---
layout: post
title: "基础篇"
date: 2016-02-12 19:32:24.000000000 +09:00
tags: Objective-C回顾温习集
---
##### 一、property相关
@property有哪些修饰符<br>
1、线程安全（原子性）<br>
&emsp;&emsp;atomic 和 nonatomic
<p>
详解：<br>
atomic（原子性---默认）:这个属性是为了保证程序在多线程下，编译器会自动生成自旋锁代码，避免该变量读写不同步问题，提供多线程安全，即多线程中只能一个线程对它进行访问。<br>
注意：<br>
1.atomic原子性指的是一个操作不可以被CPU中途暂停，然后再调度。即不能被中断，要么就执行完，要么就不执行。<br>
2.atomic是自旋锁，当上一线程没有执行完毕的时候（被锁住），下一个线程会一直等待（不会进入睡眠状态），当上衣线程任务执行完毕，下一线程立即执行。它区别于互斥锁，互斥锁在等待的时候，会进入睡眠状态，当被上一线程执行完毕后，会被唤醒，然后再执行。<br>
3.atomic只给setter方法上锁，getter不会加锁。<br>
4.atomic需要消耗大量的资源，执行效率低<br>
<br>
nonatomic (非原子性):非线程安全，多个线程可以同时对其进行访问，使用该属性会少生成加锁的代码，提高性能和效率，使用频率高，一般都是放弃安全，提高性能。
</p>
2、访问权限<br>
&emsp;&emsp;readonly、readwrite<br>
<p>
  详解：<br>
  readonly 只读属性、只会生成getter方法，不会生成setter方法<br>
  readwrite 默认，拥有getter/setter方法，可读可写
 </p>
3、内存管理<br>
&emsp;&emsp;ARC: assign、strong、weak、copy（mutableCopy）、unsafe_unretained<br>
&emsp;&emsp;MRC: assign、retain（release释放）<br>
  详解：<br>
1.assign 默认 : 适用于基本数据类型:NSInteger、CGFloat和C数据类型int、float 以及enum类型等<br>
2.strong对应MRC中的retain : 强引用，只有OC对象才能够使用该属性，它使对象的引用计数器加1<br>
3.weak : 弱引用，只是单纯引用某个对象，但是并未拥有该对象，即一个对象被持有无数个弱引用，只要没有strong引用指向它，那么它就会自动释放。<br>
4.copy(mutableCopy) : 如果想要创建一个对象，该对象与源对象的内容一致，那么就可以使用copy。<br>
copy和mutableCopy的区别：<br>
>
源对象是不可变类型或者可变类型<br>
1.copy出来的对象类型总是不可变类型（例如：NSString、NSArray，NSDictionary）<br>
2.mutableCopy出来的对象类型总是可变类型（例如：NSMutableString,NSMutableDictionary,NSMutaleArray）<br>

深拷贝和浅拷贝：<br>
>
1.深拷贝：拷贝出来的对象与源对象地址不一致，这意味着我们改变拷贝对象的值对原来的值没有任何影响。<br>
2.浅拷贝：拷贝出来的对象与源地址一致，这意味着我修改拷贝对象的值会影响到源对象。<br>
注意：<br>
”copy都是浅拷贝，mutableCopy都是深拷贝“ 这句话是错误的，解释：当我们用copy从一个可变对象拷贝出一个不可变对象时，这种情况属于深拷贝而不是浅拷贝。<br>

copy拓展： 深拷贝和浅拷贝有相对之分：
>
A、对于NSString 对象，深拷贝就是深拷贝，浅拷贝就是浅拷贝<br>
B、对于NSArray ，NSDictionary ，NSSet这些容器类的对象呢？当然浅拷贝依然是指针的拷贝，但是深拷贝要分为不完全深拷贝和完全深拷贝<br>
&emsp;不完全深拷贝：不完全深拷贝就是只拷贝对象，而对于容器内的对象则只保存一份引用。<br>
&emsp;完全深拷贝：就是连同容器的对象在内，完完全全拷贝一份<br>
ps： 默认状态下深拷贝指的是不完全深拷贝，如果要实现完全深拷贝，则要重写copyWithZone:方法<br>
如果要实现完全深拷贝可以利用容器（NSArray）的分类重写copyWithZone:方法，但是Apple官方不推荐这样做（copy内部默认调用copyWithZone:方法，但是NSArray不会调用这个方法，所以默认是不完全深拷贝）

block 为什么使用copy？
> 首先，block是一个对象，所以block理论上是可以使用retain/release，但是block在创建时它的内存是默认分配在栈（stack）上的，而不是堆（heap），当在作用域外调用该block时，MRC下就会奔溃，使用copy就能把他放在堆区，这样在作用域外调用这个block就不会crash。<br>
Apple官方文档：”Typically, you shouldn’t need to copy (or retain) a block. You only need to make a copy when you expect the block to be used after destruction of the scope within which it was declared. Copying moves a block to the heap.”一般情况下不需要自行调用copy或者retain修饰一个block，只有当需要在作用域外的地方使用时才需要copy，copy将block存内存区移到堆区。<br>
在ARC下修饰属性block<br>
1、使用weak或者assign修饰block，block访问外部变量，此block就是栈block，保存在栈中的block，当block所在的函数执行后，该bloc就会被销毁，在其他方法中访问该block就会产生野指针错误。<br>
2、copy、strong都可以修饰block，但是建议使用copy。block访问外部变量时此时block就是堆block。<br>

扩展--Block存储类型
>
1、不管是ARC还是MRC环境下，block内部如果没有访问外部变量，这个block就是全局block __NSGlobalBlock__,存储在全局区。<br>
2、在MRC下，block内部如果不访问外部变量，这个block是栈block __NSStackBlock__ ，存储在内存中的栈上。<br>
3、在MRC下，block内部访问外部变量，同时对该block做一次copy操作，这个block就是堆block __NSMallocBlock__，存储在内存的堆上。
4、在ARC下，block内部如果访问外部变量，这个block就是堆block，__NSMallocBlock__ ，存储在内存中的堆上，因为在ARC下，默认对block做一次copy操作。

![图片1](/image/blockCopy.png "block内存一览图")

---
关于copy 和strong<br>
> NSString 、NSArray、NSDictionary常用copy，为什么不用strong？<br>
strong是一个强引用，指向的是一个内存地址，copy是内容拷贝，会另外开辟内存空间，指针指向一个不同的内存地址，copy返回的是一个不可变对象，如果使用strong修饰可变对象，那么对象就可能在不经意间修改，有时不是我们想要的，而copy就不会发生这种情况。

5.unsafe_unretained : 和assign一样的作用 ， 同时也是一种弱引用的表示。<br>
weak 和unsafe_unretained的区别？
> 对于weak，指针的对象在它指向的对象释放的时候会转换为nil，并且会把所有引用过它的对象置为nil，消耗性能，这是一种安全的行为；但是unsafe_unretained 会继续指向对象存在的内存，即使在它已经销毁之后，这会导致因为访问那个已经释放对象而引起crash。<br>
为什么还要使用__unsafe_unretained？<br>
因为iOS4.0以前没有weak
<br>

4、指定方法名称
&emsp;&emsp;getter、setter<br>
方法指定： getter = XXX ,setter = XXX<br>
使用@property ,编译器会自动为我们添加getter和setter方法。

***
##### 二、基础知识点
**1、ARC下，不显示指定属性关键字时，默认关键字有哪些？<br>**
1.基本数据类型： atomic , readwrite, assign<br>
2.普通OC对象 : atomic , readwrite , strong<br>
**2、@property (copy) NSMutableArray *array; 这样写有什么问题？<br>**
&emsp;1.因为NSMutableArray使用的是copy修饰关键字(copy出来的对象总是不可变类型)，外面不管传值为NSMutableArray或者NSArray对象，array都是NSArray类型，编译器还是会认为是NSMutableArray，但是调用addObject就会crash。<br>
&emsp;2.使用了atomic属性严重影响性能。<br>
**3、如何让自己的类用copy修饰符？如何重写带copy关键字的setter？<br>**
>若想令自己所写的类具有拷贝功能，需要实现NSCopying协议；如果自定义的对象分为可变和不可变，那么同时实现NSCopying与NSMutableCopying协议。

具体步骤：<br>
1.需声明该类遵从NSCopying协议<br>
2.实现NSCopying协议的协议方法<br>
```- (id)copyWithZone:(NSZone *)zone;```<br>
重写copy关键字的setter<br>
```- (void)setName:(NSString *)name {```<br>
    ```_name = [name copy];```<br>
```}```<br>

**4、@property的本质是什么？ivar、getter、setter是如何生成并添加这个类中的？<br>**

&emsp;property的本质是：```@property = ivar + getter + setter;```<br>
解释如下：
>属性（property）有两大概念 ： ivar (实例变量)、存取方法（access method）= getter + setter

ivar、getter、setter如何生成并添加到类中的？
> “自动合成（autosynthesis）”<br>
完成属性定义后，编译器会自定编写访问这些属性所需的方法，此过程叫做“自定合成autosynthesis”，需要注意的是，这个过程由编译器在编译期执行，所以编辑器里看不到“合成方法（synthesis method）”的源码，除了生成getter、setter之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加上下划线，一次作为实例变量名的名字。比如：_name , 也可以在类的实现代码里通过@synthesize 语法来指定实例变量的名字（@synthesize name = _myName）

反编译类实现如下：
> 1.OBJC_IVAR_$类名$属性名称 : 该属性的偏移量（offset），这个偏移量是硬编码，表示该变量距离存放对象的内存区域的起始地址有多远。<br>
2.setter 与 getter方法对应的实现函数<br>
3.ivar_list : 成员变量列表<br>
4.method_list: 方法列表<br>
5.prop_list : 属性列表<br>
总结：每次增加一个属性，系统都会在ivar_list中添加一个成员变量的描述，在method_list中增加getter和setter方法的描述，在属性列表中增加一个属性的描述，然后计算该属性在对象中的偏移量，然后给出getter和setter方法对应的实现，在setter方法中从偏移量的位置开始赋值，在getter方法中从偏移量的位置开始取值，为了能够读取正确的字节数，系统对偏移量的位置进行了类型强转。

**5、@protocol和category 中如何使用@property**<br>
1.在protocol中使用property只会生成getter和setter方法的声明，我们使用属性的目的是希望遵守我协议的对象能实现该属性.<br>
2.category使用@property也是只会生成setter和getter方法的声明，如果我们需要给category增加属性的实现，需要借助于运行下面两个函数
> ```objc_setAssociatedObject``` <br> ```objc_getAssociatedObject```
