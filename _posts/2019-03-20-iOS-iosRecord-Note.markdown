---
layout: post
title: "Unity iOS 插件原理实现篇"
date: 2019-03-20 22:20:24.000000000 +09:00
tags: iOS项目疑难杂症集
---
### 概述

>Unity 作为当前最流行的游戏引擎，它具有非常强大的游戏功能。但当需要接入原生 iOS SDK或 原生 objective-c 交互时， Unity开发者难免会遇到 C 和 C# 交互的难题。本篇就介绍了 Unity 如何优雅得和 iOS进行交互。

### 常见场景

>1. 接入代理商的登录和支付功能，但代理商只提供  iOS 平台 SDK。
>2. 一些优秀的插件或工具只提供 iOS 平台 SDK。
>3. 一些游戏变现好的广告 SDK 只提供 iOS 平台 SDK。（比如 YumiMediationSDK）
>4. 需要通过 Objective-C 调用 iOS 原生类库代码。

### Unity 调用 iOS 原生方法
>C#调用iOS的代码，实际上是C#提供了一种调用C（非C++）代码的机制，而在iOS环境中，C代码是可以与苹果的Objective-C代码进行混合编译的，这样，就实现了C#调用iOS代码的功能

#### IL2CPP 简单介绍
> 我们称之为IL2CPP的技术有两个不同的部分。
>
> - 提前（AOT）编译器
> - 用于支持虚拟机的运行时库
>   AOT编译器将中间语言（IL）（.NET编译器的低级输出）转换为C ++源代码。 运行时库提供服务和抽象，如垃圾收集器，对线程和文件的独立于平台的访问，以及内部调用的实现（直接修改托管数据结构的本机代码）。

![ll2cpp示例](/image/ll2cpp.png "示例图")
> Unity 脚本(C#、JavaScript)编译成中间语言（IL，动态库等）
> 利用IL2CPP程序，将IL转换成C++语言
> 与之前不同，编译到 XCode 之后不再包含动态库(.dll)文件
> 理论上讲，直接用C++编译运行机器码，要比 mono 运行环境代码快许多，不得不说 Unity 为了性能也是够拼了
> 想要了解跟多，可以查看：http://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/


#### DllImport 机制
C#提供了DllImport机制，来实现C#和C语言的方法调用

1. 无参无返回值

   ```c#
   [DllImport("__Internal")]
   internal static extern void CallFunction();
   ```
2. 有参无返回值

   ```c#
   [DllImport("__Internal")]
   internal static extern void SetGameVersion(string appVersion);
   ```
3. 有参有返回值

   ```c#
   [DllImport("__Internal")]
   internal static extern string GetGameName(string identify);
   ```

**DllImport 的语法规则如下所示** 
```c#
[DllImport("__Internal")]
internal static extern ReturnValue FunctionName(string parameter1, int parameter1);
```
由于是C语言，支持的参数及其数据类型是非常有限的，诸如int、float、double、char等这类两种语言中都存在的基础数据类型，是可以实现直接映射的，但是C#中的string对应到C中，就是char*了

#### iOS 实现 DllImport
##### 1. C语言环境下实现交互

1. C# DllImport 声明

```c#
[DllImport ("__Internal")]
private static extern string SomeFunction();
```

2. iOS C语言环境实现
   在 .mm 的文件中
```c
extern 'C' {
  char* SomeFunction() {
      char* retString = (char*)malloc(4);
      memset(retString, 0, 4);
      strncpy(retString, "abc", 4);
      return retString;
  }
}
```
**因为当前方法的返回值是 string 类型，没有办法直接去返回需要转成char * 的格式返回，需要在 C 环境下给字符分配空间**

##### 2. Objective-C 语言环境下实现交互
> 原理
> 利用指针地址原理，在C#和OC环境下指针地址一致，在不同的环境中都可以找到对应的指针地址从而找到对应的对象完成方法调用

1. C# DllImport 传入C#返回OC对象的声明
```c#
[DllImport("__Internal")]
internal static extern IntPtr InitInstance(IntPtr unityObject); //IntPtr 返回值是一个OC对象
```
2. iOS Objective-C语言环境实现
   在 .m的文件中实现
```objective-c
/// Type representing a Unity  client.
typedef const void *InstanceClientRef;
/// Type representing a instance.
typedef const void *InstanceRef;

InstanceRef InitInstance(InstanceClientRef *client)
{
    OCInstance *ocInstance = [[OCInstance alloc] init];
    // 在这里需要把ocInstance 这个对象缓存起来，别让他释放 
    return (__bridge InstanceRef)ocInstance;
}
```

OCInstance 是一个objective-c对象，有.h 和.m 文件


### iOS 调用 Unity 原生方法
这种方式一般用于 iOS 给 unity 一些方法回调

#### UnitySendMessage

OC 调用 C#

```C
extern "C"{
    extern void UnitySendMessage(const char *, const char *, const char *);
}
```

使用：

```objective-c
UnitySendMessage("UnityManager","CallUnityFunction","");
```

C# 实现代码
在 UnityManager 这个类中实现 'CallUnityFunction' 这个方法 就可以调用成功了

```c#
public class UnityManager{
    public void CallUnityFunction()
    {
        // Implement CallUnityFunction
    }
}

```
#### MonoPInvokeCallback
ObjCRuntime.MonoPInvokeCallbackAttribute Class
>Attribute used to annotate functions that will be called back from the unmanaged world.
>MonoPInvokeCallback特性参数是定义的非托管delegate

1. C# 中定义 Delegate

```c#
public class ManagerClient : IManagerClient{
    internal delegate void InstanceDidReceiveCallback(IntPtr instace);

    [MonoPInvokeCallback(typeof(InstanceDidReceiveCallback))]
    private static void MonoDidReceiveCallback(IntPtr instace)
    {
        // oc callback method
    }
}

```
在C#中定义好MonoPInvokeCallback修饰的方法 需要传入OC原生方法中

```c#
[DllImport("__Internal")]
internal static extern void SetCallbacks(
    IntPtr ocInstace,
    ManagerClient.MonoDidReceiveCallback receivedCallback);
```

2. Objective-C 实现MonoPInvokeCallback

利用delegate的性质，把C# 传入的MonoDidReceiveCallback 方法直接赋值到OC的方式属性中

.m 中的方法

```objective-c
typedef void (*MonoDidReceiveCallback)(InstanceClientRef *instanceClient);

void SetCallbacks(InstanceRef ocInstance,MonoDidReceiveCallback receivedCallback,){
  OCInstance *ocInstance = (__bridge OCInstance *)ocInstance;
  ocInstance.receivedCallback = receivedCallback;
}
```
在 OCInstance 类中直接调用以下方法就可以在unity MonoPInvokeCallback 修饰的方法得到消息了
```objective-c
if (self.receivedCallback) {
    self.receivedCallback(self.unityClient);
}
```

上述这些 Unity 和 iOS 相互传递消息的原理和实现是我在实现Unity插件用到的，希望对你有帮助 ^_^

---
参考资料：<br>
[An introduction to IL2CPP internals](https://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/)<br>
[YumiMediationSDK-Unity 插件](https://github.com/yumimobi/YumiMediationSDK-Unity/tree/master/Assets/YumiMediationSDK/Platforms/iOS)
