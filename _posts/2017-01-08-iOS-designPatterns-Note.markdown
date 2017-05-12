---
layout: post
title: "基础篇-设计模式"
date: 2017-01-08 23:20:24.000000000 +09:00
tags: Objective-C回顾温习集
---
>
设计模式的功能在软件设计当中是为了解决一些重复的公共问题，它们是一些模板来帮助你更容易书写代码和复用你的代码。它们还可以帮助你创建低耦合的代码，你可以轻松的修改和替换其中的组件。<br>
在iOS中，常见的就是Cocoa中的设计模式：<br>
- 创建（Creational）：单例（Singleton）和抽象工厂（Abstract Factory）
- 结构（Structural）：MVC，装饰器（Decorator），适配器（Adapter），外观（Facade）和复合器（Composite）
- 行为（Behavioral）：观察者（Observer），备忘录（Memento），责任链（Chain of Responsibolity）和命令（Command）

### 一、Singleton - 单例模式 <br>
单例设计模式：确切的说就是一个类只有一个实例，有一个全局的接口来访问这个实例。当第一次载入的时候，它通常使用延时加载的方法创建一个单一实例。

简单的OC代码实例代码：<br>
```
/* Singleton.h */
  #import "Foundation/Foundation.h"
  @interface Singleton : NSObject
  + (Singleton *)sharedInstance;
  @end

  /* Singleton.m */
  #import "Singleton.h"
  static Singleton *_instance = nil;

  @implementation Singleton
  //第一种实现方式
  /*
  + (Singleton *)sharedInstance {
    if (!_instance) {
        _instance = [[Singleton alloc] init];
    }
    return _instance;
  }
  */
  //第二种实现方式
  + (Singleton *)sharedInstance {

    static dispatch_once_t onceToken;
    dispatch_once(&onceToken ,^{
      if (!_instance) {
          _instance = [[Singleton alloc] init];
      }

      });

    return _instance;
  }

```

- Cocoa库本身有很多地方也使用了单例模式，例如```[NSNotificationCenter defaultCenter]```,```[UIColor redColor]```,```[NSUserDefaults standardUserDefaults]```等等，一般Cocoa中shared或者default开头的类方法都是单例模式的方法。

- 这种写法的优点就是可以延迟加载，按需分配内存以节省开销

- 但是上述的第一种写法并不是一个线程安全的写法，比如两个或者多个线程并发的调用```sharedInstance```这个方法，有可能就会有多个实例。

- 怎么解决线程安全的问题呢？我们可以使用 @synchronized 进行加锁，代码如下:<br>

```
/* Singleton.h */
  #import "Foundation/Foundation.h"
  @interface Singleton : NSObject
  + (Singleton *)sharedInstance;
  @end

  /* Singleton.m */
  #import "Singleton.h"
  static Singleton *_instance = nil;

  @implementation Singleton
  //第一种实现方式
  /*
  + (Singleton *)sharedInstance {
  // 加锁
  @synchronized(self){
    if (!_instance) {
        _instance = [[Singleton alloc] init];
    }
  }
    return _instance;
  }
  */
  //第二种实现方式
  + (Singleton *)sharedInstance {

    static dispatch_once_t onceToken;
    //这种方式已经保证了线程的安全，所以不需要加锁。
    dispatch_once(&onceToken ,^{

      if (!_instance) {
          _instance = [[Singleton alloc] init];
      }

  });

    return _instance;
  }
```

分析：
- 第一种这种写法也是懒加载不过虽然保证了线程安全，但是由于锁的存在，当在访问多线程是性能会降低。

- 第二种方法使用GCD的dispatch_once方法，首先保证了线程的安全性，其次很好的满足了静态分析器的要求，GCD可以确保以更快的方式完成这些检测，它可以保证block中的代码在任何线程通过dispatch_once调用之前被执行，但它不会强制每次调用这个函数让代码进行同步控制。




### 二、工厂模式（Factory）
工厂模式： 本质上是使用方法简化类的选择和初始化过程。

下面是一个简单的工厂模式的例子：
```
//
//  OperationFactory.m
//  FactoryPattern

#import "OperationFactory.h"
#import "Operation.h"
#import "OperationAdd.h"
#import "OperationSub.h"
#import "OperationMul.h"
#import "OperationDiv.h"

@implementation OperationFactory

+ (Operation *) createOperat:(char)operate{
    Operation *oper = nil;
    switch (operate) {
        case '+':
        {
            oper = [[OperationAdd alloc] init];
            break;
        }
        case '-':
        {
            oper = [[OperationSub alloc] init];
            break;
        }
        case '*':
        {
            oper = [[OperationMul alloc] init];
            break;
        }
        case '/':
        {
            oper = [[OperationDiv alloc] init];
            break;
        }
        default:
            break;
    }
    return oper;
}
@end
```

由于Objective-C本身的动态特性，还可以反射来改写：
```
@implementation OperationFactory
+ (Operation *) createOperat:(NSString *)operate{
    Operation *oper = nil;
    Class class = NSClassFromString(operate);
    oper = [(Operation *)[class alloc] init];
    if ([oper respondsToSelector:@selector(getResult)]) {
        [oper getResult];
    }
    return oper;
}
@end
```

使用时，可以传入类名，来获取对应类的对象：<br>
```
Operation *oper = [OperationFactory createOperat: @"OperationAdd"];
oper.numberA = 10;
oper.numberB = 20;
NSLog(@"%f", oper.getResult);
```

总结：iOS中的工程模式就是利用OC语言的特性“动态”来创建不同的对象。

### 三、委托模式（Delegate）<br>
委托模式即是代理模式
>
在委托模式中，有两个对象参与处理同一请求，接受请求的对象将委托给另一个对象处理。

代码示例：
```
@protocol AClssDelegate <NSObject>
- (void)print;
@end


@interface AClass : NSObject<AClssDelegate>
@property （nonatomic ,weak） id<AClssDelegate> delegate;
@end

@implementation AClass

-(void)sayHello {
  // 这里一般需要判断
    [self.delegate print];
}

-(void)print {
    NSLog(@"Do Print");
}
@end

// 使用 AClass
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        AClass * a = [AClass new];
        a.delegate = a;
        [a sayHello];
    }
    return 0;
}

```

delegate的使用：
>
- 1.定义protocol和protocol中需要委托出去的方法。
- 2.定义变量id<xxxProtocol> delegate，用来设置委托对象。
- 3.在委托的对象中，配置协议，遵守协议，设置delegate = self(遵守协议的对象)，并实现协议方法。
- 4.在合适的地方调用 [delegate protocolMethod] 让委托对象工作。



### 四、观察者模式 (Observer)
在观察者模式中，当状态发生改变的时候，一个对象会通知另一个对象。这个对象不需要知道另一个对象发生了什么改变。它通常需要一个观察者（observer）注册跟踪另外一个对象的状态。当状态发生改变的时候，所有的观察者对象都会被通知改变。

Cocoa有两个常用的方法来执行观察者模式：NSNotification和Key-Value Observing（KVO）。

**4.1 NSNotification**<br>
NSNotification 基于Cocoa自己的消息中心组件NSNotificationCenter 实现。观察者需要统一在消息中心注册，说明自己要观察那些值得变化。注册代码如下：
```
[[NSNotificationCenter defaultCenter] addObserver:self
                         selector:@selector(methodName:)
                             name: @"notificationName"
                           object:nil];
```
上面的函数表明把自身注册成“notificationName”消息的观察者，当有消息时，会调用自己的"methodName:"方法。

消息发送者使用类似下面的函数发送消息：
```
[[NSNotificationCenter defaultCenter] postNotificationName:@"notificationName"
                                    object:nil
                                  userInfo:nil];
```

注意点： 在使用NSNotification的时候一定要记得在适当的时间移除通知，``` [[NSNotificationCenter defaultCenter] removerObserver:self];```

**4.2 键 – 值 观察 (Key-Value Observing KVO)**<br>
一个对象的任何一个特别的属性改变后都可以请求一个通知。在KVO中是允许一个对象观察一个属性的变化。
KVO的实现依赖Objective-C本身强大的KVC（Key Value Coding）特性，可以实现对于某个属性变化的动态监测。

实例代码如下：
```
// Book类
@interface Book : NSObject

@property NSString *name;
@property CGFloat price;

@end

// AClass类
@class Book;
@interface AClass : NSObject

@property (strong) Book *book;

@end

@implementation AClass

- (id)init:(Book *)theBook {
    if(self = [super init]){
        self.book = theBook;
        [self.book addObserver:self forKeyPath:@"price" options:NSKeyValueObservingOptionOld|NSKeyValueObservingOptionNew context:nil];
    }
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context{
    if([keyPath isEqual:@"price"]){
        NSLog(@"------price is changed------");
        NSLog(@"old price is %@",[change objectForKey:@"old"]);
        NSLog(@"new price is %@",[change objectForKey:@"new"]);
    }
}

- (void)dealloc{
    [self.book removeObserver:self forKeyPath:@"price"];
}
@end

// 使用 KVO
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Book *aBook = [Book new];
        aBook.price = 10.9;
        AClass * a = [[AClass alloc] init:aBook];
        aBook.price = 11; // 输出 price is changed
    }
    return 0;
}
```
KVO使用步骤：
>
- 1.通过```addObserver: forKeyPath: options: context:```为被监听的对象（它通常是数据模型）注册监听器。
- 2.(回调监听)重写监听器的```observeValueForKeyPath: ofObject: change: context:```方法
- 3.删除指定key路径的监听器 ：``` removeObserver: forKeyPath、removeObserver: forKeyPath: context:```

作用： KVO其实是一种观察者模式，利用它可以很容易实现视图组件和数据模型的分离，当数据模型的属性值改变之后作为监听器的视图组件就会被触发，激发时就会回调监听器本身。在ObjC中要实现KVO则必须实现NSKeyValueObServing协议，不过幸运的是NSObject已经实现了该协议，因此几乎所有的ObjC对象都可以使用KVO。

**键值编码KVC**<br>

Key Value Coding (简称KVC)：利用字符串方法去动态控制一个对象属性。KVC的操作方法由NSKeyValueCoding协议提供，而NSObject就实现了这个协议，也就是说Objc中几乎所有的对象都支持KVC操作，常用的KVC操作如下：
>
- 动态设置 ：setValue:属性值forKey:属性名（用于简单路径）、setValue:属性值forKeyPath:属性路径（用于复合路径，objectName.propertyName）
- 动态读取：valueForKey:属性名、ValueForPath:属性名（用于复合路径）


KVC的查找规则如下：（假设利用KVC对foo进行读取）：
>
- 如果是动态设置属性，则优先考虑调用setFoo方法，如果没有该方法则优先考虑搜索成员变量_foo，如果仍然不存在则搜索成员变量foo，如果最后仍然没搜索到则会调用这个类的```setValue:forUndefinedKey：```方法（注意搜索过程中不管这些方法，成员变量是私有的还是公共的都能正确设置）；
- 如果是动态读取属性，则优先考虑调用foo方法（属性foo的getter方法），如果没有搜索到则会优先搜索成员变量_foo，如果仍然不存在则搜索成员变量foo，如果最后仍然没有搜索到则会调用这个类的```valueforUndefinedKey:```方法（注意搜索过程中不管这些方法，成员变量私有的还是共有的都能正确读取）。

简单的KVC以下列形式访问属性：
```
@property (nonatomic, copy) NSString *personName;
```
取值
```
NSString *name = [object valueForKey:@"personName"]
```
设定：
```
[object setValue:@"zhangsan" forKey:@"personName"]
```
值得注意的是这个不仅可以访问作为对象属性，而且也能访问一些标量（例如int和float）和struct（例CGRect）。Foundation框架会为我们自动封装它们，举例来说：
```
@property (nonatomic) CGFloat height;
```
我们可以这样设置它：
```
[object setValue:@(20) forKey:@"height"]
```
KVC允许我们用属性的字符串名称来访问属性，字符串在这儿叫做键。

另外一些设计模式请参考[《刚刚在线设计模式详解》](http://www.superqq.com/blog/2015/06/16/ios-she-ji-mo-shi-xi-lie-:decorator-zhuang-shi-qi-mo-shi/)



### 五、知识点
**1、addObserver:forKeyPath:options:context:各个参数的作用分
别是什么，observer中需要实现哪个方法才能获得KVO回调？** <br>
```
// 添加键值观察
/*
1 观察者，负责处理监听事件的对象
2 观察的属性
3 观察的选项
4 上下文
*/
[self.person addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:@"Person Name"];
```

Observer中需要实现下列方法：
```
// 所有的 kvo 监听到事件，都会调用此方法
/*
 1. 观察的属性
 2. 观察的对象
 3. change 属性变化字典（新／旧）
 4. 上下文，与监听的时候传递的一致
 */
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context;
```

**2、如何手动触发一个value的KVO** <br>
所谓的“手动触发”是区别于“自动触发”：<br>
自动触发是指类似于这种场景：在注册KVO之前设置一个初始值，注册之后，设置一个不一样的值，就可以触发了。

自动触发KVO的原理：
>
键值观察通知依赖于NSObject的两个方法：```willChangeValueForKey:``` 和```didChangevlueForKey:```。在一个被观察属性发生改变之前，```willChangeValueForKey:```一定会被调用，这就会记录旧的值，而当发生改变后，```observeValueForKey:ofObject:change:context:```会被调用，继而```didChangevlueForKey:```也会被调用。

如果可以手动实现这些调用，就可以实现“手动触发”了。<br>
手动触发一个kvo，这个value是表示时间的self.now ,代码如下：
```
@property (nonatomic, strong) NSDate *now;
- (void)viewDidLoad {
   [super viewDidLoad];
   _now = [NSDate date];
   [self addObserver:self forKeyPath:@"now" options:NSKeyValueObservingOptionNew context:nil];
   NSLog(@"1");
   [self willChangeValueForKey:@"now"]; // “手动触发self.now的KVO”，必写。
   NSLog(@"2");
   [self didChangeValueForKey:@"now"]; // “手动触发self.now的KVO”，必写。
   NSLog(@"4");
}
```


**3、 KVC的keyPath中的集合运算符如何使用？** <br>
- 1.必须用在集合对象上或普通对象的集合属性上
- 2.简单集合运算符有@avg ，@count ，@max ，@min ，@sum
- 3.格式@“@sum.age”或@“集合属性.@max.age”





---
参考资料：<br>

[@iOSGit资料](https://hit-alibaba.github.io/interview/iOS/Cocoa-Touch/Event-Handling.html)<br>
[刚刚在线设计模式详解](http://www.superqq.com/blog/2015/06/16/ios-she-ji-mo-shi-xi-lie-:decorator-zhuang-shi-qi-mo-shi/)<br>
[@iOS程序犭袁的Git分享](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#14-synthesize%E5%90%88%E6%88%90%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E7%9A%84%E8%A7%84%E5%88%99%E6%98%AF%E4%BB%80%E4%B9%88%E5%81%87%E5%A6%82property%E5%90%8D%E4%B8%BAfoo%E5%AD%98%E5%9C%A8%E4%B8%80%E4%B8%AA%E5%90%8D%E4%B8%BA_foo%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8F%98%E9%87%8F%E9%82%A3%E4%B9%88%E8%BF%98%E4%BC%9A%E8%87%AA%E5%8A%A8%E5%90%88%E6%88%90%E6%96%B0%E5%8F%98%E9%87%8F%E4%B9%88)
