---
title: objc_msgSend 流程 3 - 动态方法决议 & 消息转发
date: 2020-09-26 23:59:19 
categories: 
- OC底层探索
tags:
- OC
- 底层
---

我们已经知道 objc_msgSend 的消息查找流程首先是 缓存 cache 查找，然后是去方法列表递归查找，若一直没有找到消息一般则会 crash 报错找不到该消息。

但是直接 crash 太过不友好，下面就进行探究苹果给我们的 3 次机会。

### 消息处理的流程图：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927100448970-724564669.png)

一、动态方法决议
========

1、通过简单代码切入
----------

简单运行下面代码

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925134338069-806551772.png)

运行崩溃：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925134407869-1251847689.png)

在 MyPerson.m 中添加 resolveInstanceMethod: 方法：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925140110744-1859072581.png) 

再次 run：发现一个问题 1，为什么这里调用了 2 遍 resolveInstanceMethod() 呢？

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925140158633-272472995.png) 

走到了 resolveInstanceMethod 中，我们在这里对找不到的方法进行处理，然后再次 run：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925141402368-1093209299.png)

关于 v@: 方法签名具体可参考 [OC 底层探索 05]() 。

上面代码运行后，不再崩溃，我们手动给 helloObj7 补充添加了 imp.

问题 2：这里处理后，为何 resolveInstanceMethod() 方法又只走了一次呢？

针对这个 resolveInstanceMethod() 调用次数的问题文章后面继续进行探究。

2、resolveInstanceMethod 源码分析 
-----------------------------

### 方法入口

#### 1. lookUpImpOrForward()：

这里的判断条件在 resolveInstanceMethod() 过程中，只会进入一次，原因参见下图中注释：

 ![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925151750883-791963356.png)

#### 2. resolveMethod_locked(): 

--> resolveInstanceMethod() / resolveClassMethod()

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925162114416-1825242574.png) 

上面代码中可以看到，在这里 resolveXXXMethod 后会再次执行 return loopUpImpAndForward(), 即:

--> 苹果给了一次机会 - 动态方法决议 - 无论是否处理都再查找一次 imp。

#### 3. resolveInstanceMethod():

--> lookUpImpOrNil() --> lookUpImpOrForward()



```
1 /***********************************************************************
 2 * resolveInstanceMethod
 3 * Call +resolveInstanceMethod, looking for a method to be added to class cls.
 4 * cls may be a metaclass or a non-meta class.
 5 * Does not check if the method already exists.
 6 **********************************************************************/
 7 static void resolveInstanceMethod(id inst, SEL sel, Class cls)
 8 {
 9     runtimeLock.assertUnlocked();
10     ASSERT(cls->isRealized());
11     SEL resolve_sel = @selector(resolveInstanceMethod:);
13     // resolve_sel --> lookUpImpOrNil() --> lookUpImpOrForward()
14     if (!lookUpImpOrNil(cls, resolve_sel, cls->ISA())) {
15         // Resolver not implemented.
16         // 查询没有 resolve_sel 这个 sel ，直接返回
17         return;
18     }
20     BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
21     bool resolved = msg(cls, resolve_sel, sel);// 发送消息，resolveInstanceMethod
23     // Cache the result (good or bad) so the resolver doesn't fire next time.
24     // +resolveInstanceMethod adds to self a.k.a. cls
25     IMP imp = lookUpImpOrNil(inst, sel, cls);// 再次去 lookUpImpOrForward 查询 imp，此imp是resolve补的
26     // lookUpImpOrNil -->lookUpImpOrForward(,,0|4|8=12)
28     if (resolved  &&  PrintResolving) {
29         if (imp) {
30             _objc_inform("RESOLVE: method %c[%s %s] "
31                          "dynamically resolved to %p", 
32                          cls->isMetaClass() ? '+' : '-', 
33                          cls->nameForLogging(), sel_getName(sel), imp);
34         }
35         else {
36             // Method resolver didn't add anything?
37             _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
38                          ", but no new implementation of %c[%s %s] was found",
39                          cls->nameForLogging(), sel_getName(sel), 
40                          cls->isMetaClass() ? '+' : '-', 
41                          cls->nameForLogging(), sel_getName(sel));
42         }
43     }
44 }
```



从上面源码，我们可以得知，动态决议所走流程是一个循环套：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200926154130341-836541216.png)

但是上面的 resolveInstanceMethod() 执行次数的问题我们还没有找到原因？下面进行细究。

查看打印时的堆栈信息：

### resolveInstanceMethod() 调 2 次的原因探究

第一次打印：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200926151224856-2063255058.png)

第二次打印：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200926151450909-1244391957.png)

通过堆栈信息，我们可看到，

第一次打印流程： objc_msgSend_uncached --> lookUpImpOrForward --> resolveInstanceMethod().

第二次流程：CoreFoundation: forwarding --> CF: -[NSObject(NSObject) methodSignatureForSelector:] --> __methodDescriptionForSelector 

--> class_getInstanceMethod() --> lookUpImpOrForward --> resolveInstanceMethod().

通过查看堆栈信息我们可以知道 resolveInstanceMethod() 第二次是因系统的消息签名机制后被调起的，**[CoreFoundation 中做了什么呢](#define_01)？**文章底部对其进行分析。

tip：关于类方法动态方法决议有个点要注意下，我们直接把 resolveClassMethod() 给到当前类进行处理是不会生效的，这里涉及到 isa 走位和继承链关系，类方法是元类的实例方法根元类的父类是 NSObject。详见：[OC 底层探索 04]().

### 动态方法决议的存在有什么意义呢？

个人想法

1、这里可以做一些埋点、问题统计等优化处理；

2、我们或许可以用其进行 封装 切面？例如：封装 SDK，定义的方法全部以 XXX_ 为前缀，针对未实现方法崩溃的问题，进行处理并记录上报问题点。但是，我们知道方法调用优先级是 子 > 父，如果出现在局部子类对 resolve 方法做了处理，那么封装其实便不会走，也浪费封装，所以在此处进行封装是不合适的。所以，这些类似操作应该在更具有可操作性的场景方法中使用，一般在 resolveInstanceMethod 这里不作处理。留下个小问题：什么场景适于使用切面呢？如何使用呢？

二、消息转发
======

动态方法决议的机会不作处理，此后会走到消息转发的流程，以下代码均以 MyPerson.m 为示例操作。

1、快速转发
------

forwardingTargetForSelector:

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    NSLog(@"快速转发这里需要 sel == %@",NSStringFromSelector(aSelector));
//    return [super forwardingTargetForSelector:aSelector];
    return [MyChild alloc];// 消息转给 MyChild
}
```

在此方法中我们只需给 SEL - aSelector 一个 imp 即可。

2、签名转发
------

methodSignatureForSelector:

forwardInvocation:



```
///消息签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSLog(@"需要消息签名 sel == %@",NSStringFromSelector(aSelector));
//    return [super methodSignatureForSelector:aSelector];
    // 给一个方法签名
    // v@: --> 返回类型void，参数类型id,SEL
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}
- (void)forwardInvocation:(NSInvocation *)anInvocation {// Invocation 启用 调用
    NSLog(@"签名过来了");
}
```



如上代码，运行程序：

当前 invocation 中包含了什么？

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925191912860-1335910778.png)

我们签名中 return 了一个方法签名，invocation 是这个抛出来事务，但我们并未对 “helloObj7” 方法进行处理，它目前仍是没有 imp 的，系统为什么会不再崩溃呢？

**消息签名的慢速转发机制**相当于，系统允许，对于此签名的事务，可以处理，也可不处理。相当于抛出去就不必管它了，任其随意游离在哪里；我这里不处理了，爱谁处理谁处理。像飘在空中的云 大家都可以看见，同时也可以不看。

但是这里**不处理**，我们**就浪费掉了一个事务**，你调用到了它，响应却不处理没利用它，**耗费了性能，同样也是业务层面的一种浪费**。

**! 慢速转发相对快速转发更灵活，权限更大**。

对 invocation 进行处理，让它去处理任一方法 (我们可以根据实际业务需求对此方法做成统一处理某件事)：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925193300765-294033375.png)

注意一点：

如果在这里把 “helloObjc7” 这个 sel 给到 invocation，然后对 invocation 启动后，会造成无线循环调用，一直去找这个不存在的“helloObjc7” --> crash.

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925192854607-926054207.png)

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200925193618324-453582650.png)

三、信息找寻 - 流程验证
=============

我们回到代码未做任何处理的初始状态, 运行：
----------------------

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927102310794-208946553.png)

通过堆栈信息查看为什么会报错，我们可以发现在 main 函数后、报错前，还有 2 个方法，它们具体做了什么如何查看呢？它们都是属于 CoreFoundation 框架的，我们去找 foundation 的源码：[CF 相关源码](https://opensource.apple.com/source/CF/) 链接.

将源码拖到 Visual Studio Code 中 ([VSCode 下载](https://code.visualstudio.com/))，搜索__forwarding_pre_0___, 找不到！没有相应开源源码 --> 反汇编。

反汇编 
----

Hopper 是一个可以将可执行文件反汇编成伪代码、控制流程图等，帮助我们可以进行文件可视性分析的反汇编工具。

### 1、通过 image list 读取全部加载的镜像文件，找到 CoreFoundation 的位置：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927004545958-1585824807.png)

### 2、前往文件，找到 coreFoundation 可执行文件：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927004812195-254042418.png)

### 3、开启 hopper ：

1）Try the Demo

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927005043765-87561684.png)

2）将可执行文件拖入 hopper： 

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927005342407-159381277.png)

3）功能界面如下：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927005709388-1078719224.png) 

### 4、开始找寻我们需要的信息

**1）搜索 __forwarding_prep_0___** --> 进入 ___forwarding___的伪代码

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927010436689-1325551716.png)

___forwarding___：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927011220557-163528555.png)

消息快速转发 forwardingTargetForSelector 没有实现怎跳转到 loc_6459b 位置，走到消息签名方法：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927011838265-924280737.png) 

1.1）消息签名 methodSignatureForSelector 未实现，跳转 loc_6490b: 报错

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927012424416-111922867.png)

1.2）消息签名 methodSignatureForSelector 实现了则继续往下走：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927110047125-1578811468.png) 

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927110924497-1081109061.png)

判断 forwardingInvocation 响应则继续向下走 (loc_6476f)：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927112005835-1953432837.png)

invocation 不响应则报错:

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927113049698-1463561371.png)  

从上面流程也验证了我们顶部的消息流程图。 

**2）如法炮制，搜索 __methodDescriptionForSelector**

--> class_getInstanceMethod：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927013225720-405215138.png) 

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200927013315246-1830353422.png)

class_getInstanceMethod 源码如下，我们可以看到内部调用了 lookUpImpOrForwarding：

![](https://img2020.cnblogs.com/blog/842658/202009/842658-20200926233912276-1108756902.png)

这里，可验证为何上面 resolveInstanceMethod() 调用了 2 次。
