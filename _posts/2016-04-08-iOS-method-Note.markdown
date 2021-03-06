---
layout: post
title: "基础篇-Objc消息机制"
date: 2016-04-08 22:12:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
消息发送和转发流程可以概括为：消息发送（Messaging）是Runtime通过selector快速查找IMP的过程，有了函数指针就可以执行对应的方法实现，消息转发（Message Forwarding）是在查找IMP失败后执行一系列转发流程的慢速通道，如果不做转发处理则会打印日志或者抛出异常。
深入理解原理查看[八面玲珑的博客](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)


### 一、Runtime基本概念 <br>
1、IMP
>IMP是“implementation”的缩写，它是Objective-C指向方法（method）实现开始的指针（A pointer to the start of a method implementation.）。<br>
表示:<br>
```typedef id (*IMP)(id, SEL, ...);``` <br>
它是一个函数指针，这是由编译器生成的，当你发起一个Objc消息之后，最终就会执行那段代码，就是由这个函数指针指定的，而IMP这个函数指针指向了这个方法的实现，既然得到了执行某个实例的某个方法的入口，我们就可以绕开传递阶段，直接执行方法。<br>
IMP指向的方法与objc_msgSend函数类型相同，参数都包含id和SEL类型，每个方法名都对应一个SEL类型的方法选择器，而每个实例对象中的SEL对应的方法实现肯定是唯一的，通过一组id和SEL参数就能确定唯一的方法实现地址；反之亦然。

2、SEL/objc_selector
> SEL是一个指向C String的指针<br>
```typedef struct objc_selector *SEL;```<br>
可以使用Objc编译器@selector()或者runtime系统的sel_registerName函数获得一个SEL类型的方法选择器。selector是方法选择器。


3、id/objc_object
>id  指向一个类的实例对象<br>
```typedef struct objc_object *id;```<br>
objc_object函数表示：```struct objc_object {Class isa;}```<br>
可以看到，objc_object中，只保存一个Class类型的isa，对象中保存了一个指向类的指针。objc_object结构体包含一个isa指针，根据isa指针就可以顺藤摸瓜找到对象所属的类。<br>
PS: isa 指针不总是指向实例对象所属的类，不能依靠它来确定类型，而是应该使用Class方法确定对象的类，因为kvo的实现机制就是将被观察对象的isa指针指向一个中间类而不是真实的类，这是这一种isa-swizzling的技术。

4、Class/objc_class
> Class - 指向类对象（objc_class）的一个指针<br>
```typedef struct objc_class *Class; ``` <br>
在objc_class结构体中：ivars是objc_ivar_list指针，methodLists是指objc_method_list指针的指针，也就是说可以动态的修改*methodLists的值来添加成员方法，这也是category实现的原理，同时解释了category不能添加属性的原因。<br>
PS:但是现在可以利用在category中添加@dynamic的属性，并利用运行期间动态提取存储方法或干脆动态转发，或者干脆使用关联度对象（assciatedObject），就是利用runtime的特性。<br>

objc_ivar_list和objc_method_list 分别表示成员变量列表和方法列表：
> 理解：<br>
objc_ivar_list结构体存储这objc_ivar数组列表，而objc_ivar结构体存储了类的单个成员变量的信息，同理objc_method_list结构体存储着objc_method数据列表，而objc_method结构体存储了类的某个方法的信息。

为什么objc_class中也有一个isa对象？
> 因为objc类本身同时也是一个对象，为了处理类和对象的关系，runtime库创建了一种叫做元类（meta class）的东西，类对象所属类型就叫做元类，它用来表述类对象本身所具备的元数据，类方法就定义于此处，因为这些方法可以理解成类对象的实例方法，每个类仅有一个类对象，而每个类对象仅有一个与之相关的元类，当你发送一个类似[NSObject alloc]的消息时，你事实上是把这个消息发送给了一个类对象（class object），这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类（root meta class）的实例，所有的元类最终都指向根元类为其超类，所有的元类的方法都有能够响应消息的类方法，所以当[NSObject alloc] 这条消息发送给类对象的时候，objc_msgSend()会去它的元类里面查找能够响应消息的方法，如果找到然后对这个类对象执行方法调用。


5、method/objc_method
> method - 是一种代表类中的某个方法的类型<br>
```typedef struct objc_method *method;```<br>
而objc_method它存储了方法名，方法类型和方法实现。<br>
* 方法名类型SEL,注意的是相同名字的方法即使在不同类中定义，它们的方法选择器是相同的。<br>
* 方法类型method_types是一个char指针，其实存储着方法的参数类型和返回值类型。<br>
* method_imp 指向了方法的实现，本质上是一个函数指针


6、_cmd
> SEL类型的一个变量，Objective -C的函数的前两个隐藏参数为self和_cmd

7、ivar
>ivar - Objective-C类中的实例变量的类型<br>
```typedef struct objc_ivar *Ivar;```

8、Cache
> 定义 :```typedef struct objc_cache *Cache;``` <br>
Cache 为方法调用的性能进行优化，通俗的说，每当实例对象接收到一个消息时，他不会直接在isa指向的类的方法列表中遍历查找能够响应消息的方法，因为这样效率太低了，而是优先在Cache中查找，runtime系统会调用的方法存到Cache中（防止下次调用再重新去找，提高效率）。

9、property
> @property 标记了类中的属性，它是一个指向objc_property 结构体的指针<br>
 ```typedef struct objc_property *property;```<br>
 ```typedef struct objc_property *objc_property_t;```// 一般这个常用<br>
 注意：<br>
 1、通过class_copyPropertyList 获取类中的属性，不带下划线<br>
 2、通过protocol_copyPropertyList 获取协议中的属性<br>
 3、通过class_copyIvarList 可以获取类中的属性，包括成员变量，但是此时获取的属性名是带下划线的

### 二、消息
> Objc中发送消息就是用中括号([])把接受者括起来，而直到运行时才会把消息方法和方法实现绑定。

1、objc_msgSend函数<br>
objc_msgSend 消息发送步骤：
> 1.检测这个selector是不是要忽略<br>
2.检测这个target是不是nil对象，objc的特性允许对一个nil对象执行一个方法不会crash，因为会忽略<br>
3.如果上面两个都过了，那就开始查找这个类的IMP,先从Cache中查找，完了就去对应的函数去执行。<br>
4.如果Cache找不到就找下一个方法分发表（Class 中的方法列表：它将方法选择器和方法实现地址联系起来）。<br>
5.如果分发表中找不到就去超类的分发表去找，一直找，直到找到NSObject类为止。<br>
6.如果还找不到就要开始进入动态方法解析（后面讲解）。<br>
四个调用方法：objc_msgSend ,objc_msgSend_stret ,objc_msgSendSuper,objc_msgSendSuper_stret，根据情况选择一个阿里调用。

2、method中的隐藏参数<br>
当objc_msgSend 找到方法对应的实现时，它将直接调用该方法的实现，并将消息中的所有的参数传递给方法实现，同时还将传递两个隐藏的参数:
> 接收消息的对象（也就是self指向的内容）<br>
方法选择器（_cmd）指向的内容

3、动态方法解析<br>
使用@dynam关键字在类的实现方法中修饰一个属性:
> ```@dynamic propertyName;```<br>
这表明我们会为这个属性动态的提供存取方法，也就是说编译器不会再默认为我们生成setter和getter方法，而需要我们动态提供，我们可以通过分别重载resolveInstanceMethod:和resolveClassMethod:方法分别添加实例变量和类方法实现，因为当runtime系统在Cache和方法分发表找不到执行的方法时，就会调用resolveInstanceMethod:和resolveClassMethod:来给程序员一次动态添加方法实现的机会。我们需要用class_addMethod函数完成向特定类添加特定方法实现的操作。<br>
ps： 动态方法解析会在消息转发机制侵入前执行。

理解[self class]与object_getClass(self)以及object_getClass(self class)的关系?
>1.当self 为实例对象时，[self class] 与object_getClass(self)等价，因为前者调用后者，object_getClass([self class])得到元类。<br>
2.当self为类对象时，[self class]返回值为自身，还是self,object_getClass(self)与object_getClass(self class)等价

4、重定向<br>
在消息转发机制执行前，runtime系统会给我们一次偷梁换柱的机会，即通过重写- ```(id)forwardingTargetForSelector:(SEL)aSelector```方法替换消息接受者为其他对象。

5、转发<br>
当动态方法解析不做处理返回No时，消息转发机制会被触发，在这时```forwardInvocation:```被执行，我们可以重写这个方法定义我们自己的转发逻辑。

6、转发与继承<br>
消息转发弥补了objc不支持多继承的性质。<br>
尽管转发很像继承，但是NSObject类不会将两者混淆，想```respondsToSelector: 和 isKindOfClass: ``` 这类方法只会考虑继承体系，不会考虑转发链。

7、Objective-C Associated Objects<br>
Runtime系统让Objc支持向对象动态添加变量，设计下列三个函数:
>
1.```void objc_setAssociatedObject ( id object, const void *key, id value, objc_AssociationPolicy policy );```<br>
2.```id objc_getAssociatedObject ( id object, const void *key );
```<br>
3.```void objc_removeAssociatedObjects ( id object );```<br>
这些 方法以键值对的形式动态的地向对象添加、获取、或删除关联值。

### 三、基础知识点 <br>
**1、objc中向一个nil对象发送消息将会发生什么？<br>**
> 在Objective-C中向nil发送消息是完全有效的----只是在运行时不会有任何作用。<br>
1.如果一个方法的返回值是一个对象，那么发送给nil的消息将返回0（nil）<br>
2.如果方法返回值为指针类型，其指针大小为小于或等于sizeof(void *)，float,double，long double，或者long long的整型数标。发送给nil消息将返回0<br>
3.如果方法返回值为结构体，发送给nil的消息将返回0，结构体中各个字段的值将都是0。<br>
4.如果方法的返回值不是上述提到的这些情况，那么发送给nil的消息的返回值将是未定义的。<br>

具体原因：
> objc是动态语言，每个方法在运行时会被动态转发为消息发送，即:```objc_msgSend(receiver,selector)```<br>
objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，objc_msgSend方法不会返回值，所谓的返回内容都是具体调用时执行的。那么如果向一个nil对象发送消息时，首先在寻找isa指针就返回0，所以不会出现任何错误。

**2、objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间什么关系？**<br>
> [obj foo] 该方法编译之后就是objc_msgSend()函数的调用<br>
[obj foo] 在objc动态遍以时，会被转意成：objc_msgSend(obj ,@selector(foo))

**3、什么时候会报unrecognized selector的异常？**<br>
一般简单的说:
> 当调用该对象上某个方法，而该对象上没有实现这个方法的时候，可以通过“消息转发”来解决。

消息发送流程：
> objc 在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类的中的方法列表以及其父类的方法列表中寻找方法运行，如果在最顶层的父类中依然找不到相应的方法时，程序在运行时就会crash并且跑出异常，unrecognized selector send to XXX

在程序crash之前，objc的运行时会给出三次拯救奔溃的机会：<br>
1.Method resolution<br>
>objc运行时会调用```resolveInstanceMethod:```或者```+resolveClassMethod:```,让我们有一个机会提供函数的实现，如果添加了函数，那运行时系统就会重新启动一次消息发送的过程，否则，运行时就会移到下一步“消息转发(Message Forwarding )”。<br>

2.Fast  Forwarding<br>
 > 如果目标对象实现了```-forwardingTargetForSelector:```，runtime这时就会调用这个方法，给你把这个消息转发给其他对象，只要这个方法返回的不是nil和self，整个消息发送的过程就会被重启，当然发送的对象会变成返回的那个对象，否则就会继续Normal Forwarding。这里叫fast，只是为了区别下一步转发机制，因为这一步不会创建新的对象，但下一步转发会创建一个NSInvocation对象，所以相对fast。<br>

 3.Normal forwarding
 >这一步是runtime最后一次挽救的机会，首先它发送```-methodSignatureForSelector:```消息获得函数的参数和返回值类型，如果```-methodSignatureForSelector:```返回nil，runtime则会发出```-doesNotRecognizeSelector:```消息,程序在这个时间已经挂掉了。如果返回一个函数签名，runtime就会创建一个NSInvocation对象并发送```-forwardInvocation:```消息给目标对象。

**4、一个objc对象如何进行内存布局？(考虑有父类的情况)**<br>
.所有父类的成员变量和自己的成员变量都会存放在该对象所对应的存储空间中。<br>
.每一个对象内部都会有一个isa指针，指向他的类对象，类对象中存放着本对象的如下信息：
> 1.对象方法列表（对象能够接受的消息列表，保存在它所对应的类对象中）<br>
2.成员变量的列表<br>
3.属性列表<br>
类对象的内部也有一个isa指针指向元对象（meta class），元对象内部存放的是类方法列表，类对象内部还有一个superclass的指针，指向他的父类对象。<br>

注意：
>.根对象就是NSObject，它的superclass指针指向的是nil<br>
.类对象也是对象，是一个实例，类对象也有一个isa指针指向他的元类，即类对象的元类实例，元类内部存放的是类方法列表，根元类isa指针指向自己，superclass指向NSObject类。

**5、一个objc对象的isa指针指向的是什么？有什么作用？**<br>
指向他的类对象，从而找到对象上的方法（属性，成员变量）

**6、runtime是如何通过selector找到IMP地址？(分别类方法和实例方法)**<br>
每一个类对象中都有一个方法列表，方法列表中记录着方法的名称，方法实现，以及参数类型，其实selector本质就是方法名称，通过这个方法名称就可以在方法列表中找到对应的实现。

**7、使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放吗？**<br>
无论MRC还是ARC下均不需要

对象的内存销毁时间表，分四个步骤：
> 1.调用-release ： 引用计数器为0<br>
&emsp;* 对象正在被销毁，生命周期即将结束<br>
&emsp;* 不能在有新的__Weak 弱引用，否则将指向nil<br>
&emsp;* 调用[self dealloc]<br>
2.子类调用-dealloc<br>
&emsp;* 继承关系中最底层的子类在调用-dealloc<br>
&emsp;* 如果是MRC代码 则会手动释放实例变量们（iVars）<br>
&emsp;* 继承关系中每一层的父类都在调用-dealloc<br>
3.NSObject 调 -dealloc<br>
&emsp;* 只做一件事：调用Objective-C runtime 中的object_dispose()方法<br>
4.调用object_dispose()<br>
&emsp;* 为C++的实例变量们（iVars）调用destructors<br>
&emsp;* 为ARC状态下的实例变量们（iVars）调用-release<br>
&emsp;* 解除所有使用runtime Associate方法关联对象<br>
&emsp;* 解除所有__Weak 引用<br>
&emsp;* 调用free（）

**8、objc中的类方法和实例方法有什么本质区别和联系？**<br>
类方法：
>- 1.类方法是属于类对象的<br>
- 2.类方法只能通过类对象调用<br>
- 3.类方法中的self是类对象
- 4.类方法可以调用其他的类方法
- 5.类方法中不能访问成员变量
- 6.类方法中不能直接调用对象方法

实例方法：
>
- 1.实例方法是属于实例对象的
- 2.实例方法只能通过实例对象调用
- 3.实例方法中的self是实例对象
- 4.实例方法中可以访问成员变量
- 5.实例方法中可以直接调用实例方法
- 6.实例方法可以调用类方法（通过类名）

**9、_objc_msgForward 函数是做什么的？直接调用它就会发生什么？**<br>
```_objc_msgForward``` 是IMP类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，```_objc_msgForward```会尝试做消息转发。<br>

回顾消息传递：
> 在”消息传递“的过程中objc_msgSend的动作就是：首先在Class中的缓存查找IMP（没缓存则初始化缓存），如果没有找到，则向父类的Class查找，如果一直查找到根类依旧没有实现，则用```_objc_msgForward```函数指针代替IMP，最后执行这个IMP。

总结```_objc_msgForward```消息转发做的几件事：
>- 1.调用```resolveInstanceMethod:```方法（或```resolveClassMethod:```）,允许用户在此时为该class动态添加实现，如果有实现了则调用并返回YES，那么重新开始```objc_msgSend```流程。，这一次对象会响应这个选择器，一般是因为它已经调过了```class_addMethod```，如果仍没实现，继续下面的动作
- 2.调用```forwardingTargetForSelector:```方法，尝试找到一个能响应消息的对象，如果获取到，则直接把消息转发给它，返回非nil对象，否则返回nil，继续下面动作，这里需要注意不能返回self，会造成死循环.
- 3.调用```methodSignatureForSelector:```方法，尝试获得一个方法签名，如果获取不到，则直接调用 ```doesNotRecognizeSelector:```抛出异常，如果能获取，则返回非nil，创建一个NSInvocation并传给```forwardInvocation:```
- 4.调用```forwardInvocation:```方法，将第三步获取到的方法签名包装成invocation传入，如何处理就在这里面，并返回非nil
- 5.调用```doesNotRecognizeSelector:```，默认的实现是抛出异常，如果第三步没能获取到一个方法签名，就执行这个方法。

以上的方法均属于模板方法，开发者尅override ，有runtime来调动，最常见实现消息转发：就是重写3和4方法，吞掉一个消息或者代理给其他对象都是没有问题的。<br>

直接调用```_objc_msgForward```会发生什么？<br>
+ 一旦调用了```_objc_msgForward```，将跳过超找IMP的过程，直接触发“消息转发”
+ 如果调用了```_objc_msgForward```，即使这个对象确实已经实现了这个方法，你也会告诉```objc_msgSend```我没有在这个对象里找到方法的实现。<br>

直接消息转发，是一个非常危险的操作，但是如果用的好就是大牛了。<br>
常见的使用```_objc_msgForward```场景：想获取某方法对应的NSInvocation对象
。详细实例：JSPatch，在：[《JSPatch实现原理详解》](http://blog.cnbang.net/tech/2808/)就直接调用了```_objc_msgForward```来实现的。

**10、能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量，为什么？**
- 不能向编译后得到的类中增加实例变量；
- 能向运行时创建的类中添加实例变量；
<br>

解释：
>
+ 因为编译后的类已经注册到runtime中，类结构体中的 ```objc_ivar_list```实例变量的链表和```instance_size```实例变量的内存已经确定，同时runtime会调用```class_setIvarLayout```或```class_setWeakIvarLayout```来处理strong weak的引用，所以不能向存在类中添加实例变量
+ 运行时创建的类是可以添加实例变量的，调用```class_addIvar```函数，但是要在调用```objc_allocateClassPair```之后，在```objc_registerClassPair```之前，原因同上。

---
参考资料：<br>
[@iOS程序犭袁的Git分享](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#14-synthesize%E5%90%88%E6%88%90%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E7%9A%84%E8%A7%84%E5%88%99%E6%98%AF%E4%BB%80%E4%B9%88%E5%81%87%E5%A6%82property%E5%90%8D%E4%B8%BAfoo%E5%AD%98%E5%9C%A8%E4%B8%80%E4%B8%AA%E5%90%8D%E4%B8%BA_foo%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E9%82%A3%E4%B9%88%E8%BF%98%E4%BC%9A%E8%87%AA%E5%8A%A8%E5%90%88%E6%88%90%E6%96%B0%E5%8F%98%E9%87%8F%E4%B9%88)
