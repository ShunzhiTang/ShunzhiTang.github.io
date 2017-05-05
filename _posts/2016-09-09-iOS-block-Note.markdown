---
layout: post
title: "基础篇-Block"
date: 2016-09-09 23:12:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
一个众所周知的概念： Block就是Objective-C 对于闭包的实现。

### 一、iOS中内存相关<br>
**1、iOS内存分区**<br>
- 栈区（stack）
>
*1.由系统自动分配，一般存储函数参数值，局部变量等。<br>
*2.由编译器自动创建和释放，一旦出了作用域就会被销毁，不需要程序员管理栈区变量内存。<br>
*3.操作方式类似于数据结构中的栈，即后进先出，先进后出原则。<br>
*4.栈区地址从高到低分配。

- 堆区（heap）
>
*1.一般由程序员申请并指明大小，最终需要由程序员释放；如果程序员不回收，程序结束时可能由OS回收。<br>
*2.堆区（iOS）分配内存使用alloc ，C中是malloc函数。<br>
*3.堆区的管理采用链表式管理，操作系统有一个记录空闲内存地址的链表，当接受到程序分配的内存申请时，OS就会遍历该链表，遍历到一个记录的内存地址大于申请内存的链表节点，并将该节点从该链表中删除，然后该节点记录的内存地址分配成程序。<br>
*4.堆区的地址是从低到高分配 。

- 全局区/静态区（static）
>
*1.全局变量和静态变量存储在这个区域。<br>
*2.包括未初始化（eg:int a;）,初始化（eg: int b = 10;），就是说初始化的全局变量和静态常量存储在一块区域，未初始化全局变量和静态常量存储在一块。程序结束后由系统释放。<br>

- 常量区
>
*存储字符串常量。程序结束后由系统释放

- 程序代码区
>*主要存放函数体的二进制代码


### 2、闭包（Closure）<br>
闭包就是一个函数，或者一个指向函数的指针，加上这个函数执行的非全局变量；通俗点说，就是闭包允许一个函数访问声明该函数运行上下文中的变量，甚至可以访问不同运行上下文中的变量。<br>
闭包 = 一个函数[或指向函数的指针] + 该函数执行的外部的上下文变量[自由变量]

### 3、Block基础<br>
**1、Block可以认为是一个匿名函数，使用如下语法**<br>
声明Block类型：<br>
```return_type (^block_name)(parameters)```
<br>
定义Block：<br>
```^(parameters){return return_type};``` 这种写法省略了返回值类型，也可以显示地指出返回值类型。

**2、Block结构图如下：**
 ![Block](/image/blockStruct.png  "Block结构图")
<br>

**3、Block使用<br>**
> 声明并定义完一个Block之后，便可以向函数一样使用它，同时，Block是一种Objective-C对象，可以赋值，当做参数传递，也可以放在NSArray或NSDictionary中。<br>
注意：当用于函数参数时，Block应该放在参数列表的最后一个。

Block语法：
>
- 作为变量:<br>
```return_type (^blockName)(parameterTypes) = ^returnType(parameters){...}```
- 作为属性:<br>
```@property (nonatomic ,copy) returnType (^blockName)(parameterTypes)```
- 作为函数声明中的参数: <br>
```- (void)someMethodThatTakesABlock:(returnType)(^)(parameterTypes)blockName;```
- 作为函数调用中的参数:<br>
``` [someObject someMethodThatTakesABlock:^returnType (parameters) {...}];```
- 作为typedef:<br>
```typedef  returnType (^TypeName)(parameterTypes);```<br>
```TypeName blockName = ^returnType(parameters){...}```

**4、Block和外部变量<br>**
- 1.默认情况
> 对于Block外的变量引用，Block默认是将其复制到其他数据结构中来实现访问的。也就是说Block的自动获取变量只针对Block内部使用的自动变量，不使用则不获取，因为获取的自动变量会自动存在于Block的结构体内部，会导致Block体积变大。特别注意：默认情况下Block只能访问变量不能修改局部变量的值。

- 2.Block修改外部变量的值
> 对于用__block修饰的外部变量的引用，block是复制其引用地址来实现访问的。Block可以修改__block修饰的外部变量的值。<br>
?为什么使用__block修饰的外部变量可以被Block修改呢？<br>
*使用clang将OC代码转换成C++代码，会发现一个局部变量加上__block修饰符会变成了和Block一样的一个```__Block_byref_val_0```结构体类型的自动变量实例，此时我们在Block内部访问val变量则需要通过一个叫__forwarding的成员变量间接访问val变量。


**5、__block变量与__forwarding<br>**
关系如图：
![Block](/image/blockForwarding.png  "__block和__forwarding结构图")
通过_forwarding ，无论是在Block中还是Block外访问__block变量，也不管该变量是在堆上还是栈上，都能够顺利访问同一个__block变量。


**6、Block的存储<br>**
>
- 1.Block不访问外界变量（包括栈中和堆中的变量）
  *Block不管在ARC还是MRC下都存储在全局区
- 2.Block访问外部变量
  *MRC环境下：访问外部变量的Block默认存储在栈区，如果有copy操作，则Block存储在堆区。<br>
  *ARC环境下：访问外界变量的Block都（不管默认还是copy）存储在堆区（实际是存放在栈区，但是ARC下默认copy到堆区），自动释放。


### 四、Block知识点<br>
**1、在Block内部如何修改Block外部的变量？**<br>
即：写操作不对原变量生效， 加上__block，原因上面解释了。<br>
！！！注意：Block不允许修改外部变量的值，这里所说的外部变量的值，指的是栈中指针的内存地址。栈区是红灯区，堆区才是绿灯区。ps：Block内部的变量是可以修改的。

**2、使用Block id类型变量的问题**<br>
对于id类型的变量，在MRC情况下，使用__block id x不会retain变量，而在ARC情况下则会对变量进行retain（相当于默认的copy操作）。如果不想在Block中进行retain可以使用```__unsafe_unretain __block id x```，不过这样可能会造成野指针出现，更好的方法是使用__weak 的临时变量，或者把使用 __block 修饰的变量设为nil，打破引用循环。

**3、Block使用copy和strong问题** <br>
在非ARC的情况下，对于Block类型的属性应该使用copy，因为操作Block需要维持其作用域中捕获的变量。在ARC下编译器会自动对Block进行copy操作，因此使用strong 或者copy都可以，没有什么区别，但是Apple建议使用copy来指明编译器的行为。

**4、Block循环引用问题**<br>
☀︎ ARC<br>
*Block在捕获外部变量的时候，会保持 一个强引用，当在Block中捕获self时，由于对象对Block进行了copy，于是就形成了强引用循环，为避免强引用循环，最好捕获一个self的弱引用（__weak typeof(self) weakSelf = self;）,但是弱引用还会带来另一个问题，weakSelf有可能为nil，如果多次调用weakSelf的方法，有可能在Block执行过程中weakSelf变成nil，出现crash，因此需要在Block中将weakSelf”强化“（__strong typeof(self) strongSelf = weakSelf），__strong 这一句执行的时候，如果weakSelf还没有变成nil，那么就会retain self，让self在执行期间不会变成nil。如果weakSelf变成nil，就直接return 返回。  这样在Block内部的东西要么全部执行要么不执行。<br>
☼ MRC<br>
*使用__block解决循环引用，但是需要把这个__block 修饰的变量设为nil。

**5、使用系统的某些Block Api（如UIView的block版本写动画时），是否也考虑引用循环问题？**<br>
系统的某些Block Api中，UIView的Block版本写动画时不需要考虑，但是有些APi需要考虑一下。<br>
如果使用了一些参数中含有ivar的系统API ，如GCD ，NSNotificationcenter要小心一点，比如：如果GCD内部如果引用了self，而且GCD的其他参数是ivar，则要考虑循环引用。<br>
解决办法： 使用检测工具或者在写代码的时间注意一下就可以了。


---
参考资料：<br>
[@iOS程序犭袁的Git分享](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#14-synthesize%E5%90%88%E6%88%90%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E7%9A%84%E8%A7%84%E5%88%99%E6%98%AF%E4%BB%80%E4%B9%88%E5%81%87%E5%A6%82property%E5%90%8D%E4%B8%BAfoo%E5%AD%98%E5%9C%A8%E4%B8%80%E4%B8%AA%E5%90%8D%E4%B8%BA_foo%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E9%82%A3%E4%B9%88%E8%BF%98%E4%BC%9A%E8%87%AA%E5%8A%A8%E5%90%88%E6%88%90%E6%96%B0%E5%8F%98%E9%87%8F%E4%B9%88) <br>
[iOS Block详解](http://www.imlifengfeng.com/blog/?p=457)
