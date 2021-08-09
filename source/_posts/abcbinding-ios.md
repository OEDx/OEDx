---
title: ABCBinding--简化Cocos和Native交互利器（iOS篇）
date: 2021-05-20 11:18:5
tags: [Cocos, 热更新, 注解]
categories: 客户端
author: "[李剑飞(ezli)](https://lijianfei.com)"
---

> 在[ABCBinding--简化Cocos和Native交互利器](https://mp.weixin.qq.com/s/eB3MgmND3_msij18jDu0hQ)里主要介绍了`ABCBinding`在`JavaScript`侧和`Android`侧的设计和实现，本篇是其姊妹篇，主要介绍`iOS`侧是如何设计和实现的。

## 1. 背景介绍

### 1.1 Cocos 调用 iOS

在官方文档中，说明了如何在 iOS 平台上使用` Javascript` 直接调用 `Objective-C` 方法：

```js
jsException: function (scence, msg, stack) {
    if (cc.sys.isNative && cc.sys.os === cc.sys.OS_ANDROID) {
        jsb.reflection.callStaticMethod(
            "xxx/xxx/xxx/xxx/xxx",
            "xxx",
            "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V",
            scence, msg, stack
        );
    } else if (cc.sys.isNative && cc.sys.os === cc.sys.OS_IOS) {
        jsb.reflection.callStaticMethod(
            "xxx",
            "xxx:xxx:xxx:",
            scence, msg, stack
        );
    }
}
```

从代码当中我们可以看出有以下问题：

1. 方法调用比较麻烦，需要明确的指出方法的`Class`，方法名，参数类型，特别容易出错；
2. 一旦 `Native` 端的方法发生变化，`Cocos`层必须要同步修改，否则就会出现异常，甚至有可能 `crash`；
3. 不方便回调，这一点非常痛苦，`Cocos`作为`UI`层，`native`大部分是基于响应式给予`Cocos`响应，无法直接回调真是是噩梦；

我们尝试着用注解的思路来解决这些问题，从而设计并实现了 `ABCBinding` 。`ABCBinding` 能够极大的简化 `Cocos`和 `Native` 的交互，方便维护。

### 1.2 iOS 调用 Cocos 

通过 `evalString` 在 `C++ `/ `Objective-C` 中执行 `JavaScript` 代码。

```objc
Application::getInstance()->getScheduler()->performFunctionInCocosThread([=](){
    se::ScriptEngine::getInstance()->evalString(script.c_str());
});
```

这里需要注意：除非明确当前线程是 **主线程**，否则都需要将函数分发到主线程执行。

在`Cocos`调用不支持回调的情况下，我们只能通过`evalString`的方式，再调用`Cocos`，当然这也是唯一的途径。

## 2. 业内其他方案回顾

> 有这么一句话：如果说我看得远，那是因为我站在巨人的肩上。
>
> 这句话是牛顿的经典名言，意思是说他这一生的成就是因为有之前的巨人做基础。

### 2.1 React Native

我们知道，`React Native`可以调用`native`侧的方法。并且`RN`框架也给我们提供了这一能力，只要我们按照某些约定在`native`侧实现一个方法，那么就可以在`JS`侧顺利调用。如下实现了一个简单的`native`模块：

```objc
#import <Foundation/Foundation.h>
#import <React/RCTBridgeModule.h>

@interface NativeLogModule : NSObject<RCTBridgeModule>

@end

#import "NativeLogBridge.h"

@implementation NativeLogModule
RCT_EXPORT_MODULE()
RCT_EXPORT_METHOD(nativeLog:(id)obj) {
  NSLog(@"开始输出日志：");
  NSLog(@"%@",obj);
  NSLog(@"日志输出完毕！");
}
@end
```

以上是一段`OC`写的`native`代码，`NativeLogModule`遵守了`RN`提供的协议`RCTBridgeModule`。该协议中规定了一些宏和方法。遵守了这个协议，`NativeLogModule`就可以使用`RCT_EXPORT_MODULE()`宏将该类以`module`的方式暴露给`JS`，然后使用`RCT_EXPORT_METHOD`将`native`方法暴露给`JS`。

接下来看下`JS`侧是怎么调用`NativeLogModule的nativeLog`方法。

```js
import { NativeModules } from 'react-native';

NativeModules.NativeLogModule.nativeLog('记录点什么吧');
```

以上两行`JS`即可实现`native`方法的调用，第一行是导入`NativeModules`模块，第二行通过`NativeModule`调用`NativeLogModule`

的`nativeLog`方法。以上即可实现`JS`调用`Native`方法。但在学习`RN`之初，想必大家都有一个疑问，`Native`方法是怎么暴露给`JS`的呢？`JS`又是怎么调用这些`Native`方法的呢？

这里就不得不说RN中的两个宏了，`RCT_EXPORT_MODULE `和 `RCT_EXPORT_METHOD`。

#### 2.1.1 RCT_EXPORT_MODULE 模块导出

```objc
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
```

如上代码所示，`RCT_EXPORT_MODULE`宏背后是两个静态方法`+(NSString *)moduleName和+(NSString *)load`。`moduleName`方法简单的返回了`native`模块的类名，如果`RCT_EXPORT_MODULE`宏的参数是空，就默认导出类名作为模块名，如果参数不是空，就以参数名为模块名。`load`方法是大家耳熟能详的的，`load`方法调用`RCTRegisterModule`函数注册了模块。

#### 2.1.2 RCTRegisterModule 模块注册

让我们再来看看`RCTRegisterModule`函数的实现（该函数定义在`RCTBridge.m`中）：

```objc
static NSMutableArray<Class> *RCTModuleClasses;

void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
    RCTModuleClassesSyncQueue = dispatch_queue_create("com.facebook.react.ModuleClassesSyncQueue", DISPATCH_QUEUE_CONCURRENT);
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  dispatch_barrier_async(RCTModuleClassesSyncQueue, ^{
    [RCTModuleClasses addObject:moduleClass];
  });
}
```

很简单，`RCTRegisterModule`函数只做了3件事：

1. 创建一个全局的可变数组和一个队列（懒加载）；

2. 检查导出给`JS`模块是否遵守了`RCTBridgeModule`协议；

3. 把要导出的类添加到全局的可变数组中进行记录。

可见，在`app`启动后调用`load`方法时，所有需要暴露给JS的方法都已经被注册到一个数组中。到此为止，只是把需要导出给`JS`的类记录下来了，那这些类又是在什么阶段提供给`JS`的呢？`Native`侧全局搜索`RCTGetModuleClasses`函数，可以看到该函数在`RCTCxxBridge`的`star`t中被调用了：

```objc
- (void)start
{
  // 此处省略若干行...
  [self registerExtraModules];
  // Initialize all native modules that cannot be loaded lazily
  (void)[self _initializeModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];
  [self registerExtraLazyModules];
  // 此处省略若干行...
}
```

#### 2.1.3 RCT_EXPORT_METHOD 方法导出

```objc
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)
  
#define RCT_REMAP_METHOD(js_name, method) \
 _RCT_EXTERN_REMAP_METHOD(js_name, method, NO) \
 - (void)method RCT_DYNAMIC;

#define _RCT_EXTERN_REMAP_METHOD(js_name, method, is_blocking_synchronous_method) \
  + (const RCTMethodInfo *)RCT_CONCAT(__rct_export__, RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    static RCTMethodInfo config = {#js_name, #method, is_blocking_synchronous_method}; \
    return &config; \
  }

#define RCT_CONCAT2(A, B) A ## B
#define RCT_CONCAT(A, B) RCT_CONCAT2(A, B)
```

通过上面一系列的宏调用不难看出，`RCT_EXPORT_METHOD`最终做了2件事：

1. 定义一个实例方法；

2. 定义一个静态方法，该方法名的格式是`+(const RCTMethodInfo *)__rct_export__+js_name+___LINE__+__COUNTER__`

如果没有`js_name`参数，那么最终的格式就是`+(const RCTMethodInfo *)__rct_export__+___LINE__+__COUNTER__`

比如：`+(const RCTMethodInfo *)__rct_export__131` 其中13是`__LINE__`，1是`__COUNTER__`

而这个方法的实现是这样的：

```objc
+(const RCTMethodInfo *)__rct_export__+js_name+__LINE__+__COUNTER__ {
    static RCTMethodInfo config = {
        js_name,
        method,
        is_blocking_synchronous_method
    }
    return &config;
}
```

可以看出，最终把这个方法包装成了一个`RCTMethodInfo`对象，在运行时`RN`会扫描所有导出的`native module`中以`__rct_export__`开头的方法。

### 2.2 Flutter

`Flutter` 提供了 `Platform Channel `机制，让消息能够在 `native` 与 `Flutter` 之间进行传递。

每个` Channel` 都有一个独一无二的名字，`Channel` 之间通过 `name` 区分彼此。

`Channel` 使用 `codec `消息编解码器，支持从基础数据到二进制格式数据的转换、解析。

`Channe`l 有三种类型，分别是：

1. ` BasicMessageChannel`：用于传递基本数据 ;

2. `MethodChannel`： 用于传递方法调用，`Flutter` 侧调用 `native` 侧的功能，并获取处理结果;

3. `EventChannel`：用于向 `Flutter `侧传递事件，`native` 侧主动发消息给 `Flutter`。

这三种类型比较相似，因为它们都是传递数据，实现方式也比较类似;

我们从调用处开始进入 `Flutter engine`，一步步跟踪代码运行过程。

#### 2.2.1 函数注册

在 iOS 项目中，注册获取电量的` channel`

```objc
FlutterMethodChannel* batteryChannel = [FlutterMethodChannel
      methodChannelWithName:@"samples.flutter.io/battery"
            binaryMessenger:controller];
            
[batteryChannel setMethodCallHandler:^(FlutterMethodCall* call,
                                         FlutterResult result) {
    if ([@"getBatteryLevel" isEqualToString:call.method]) {
      int batteryLevel = [weakSelf getBatteryLevel];
      if (batteryLevel == -1) {
        result([FlutterError errorWithCode:@"UNAVAILABLE"
                                   message:@"Battery info unavailable"
                                   details:nil]);
      } else {
        result(@(batteryLevel));
      }
      ......
  }];
```

中间的跟踪代码咱就省去了，直奔核心代码。

`FlutterEngine` 将桥接事件转发给 `iosPlatformView` 的 `PlatformMessageRouter`

```objc
// PlatformMessageRouter
void PlatformMessageRouter::SetMessageHandler(const std::string& channel,
                                              FlutterBinaryMessageHandler handler) {
  message_handlers_.erase(channel);
  if (handler) {
    message_handlers_[channel] =
        fml::ScopedBlock<FlutterBinaryMessageHandler>{handler, fml::OwnershipPolicy::Retain};
  }
}
```

`PlatformMessageRouter` 的属性 `message_handlers_` 是个哈希表，`key` 是桥接名，`value` 放 `handle`。原生注册桥接方法，**其实就是维护一个 `map `字典对象**。这样，注册原生方法完成了。

#### 2.2.2  函数调用

先来看下 dart 侧如何调用获取电量的桥接方法

```dart
static const MethodChannel methodChannel =
      MethodChannel('samples.flutter.io/battery');
      
final int result = await methodChannel.invokeMethod('getBatteryLevel');
```

`invokeMethod`通过一系列调用，最终把消息转发给`PlatformMessageRouter`

```c++
void PlatformMessageRouter::HandlePlatformMessage(
    fml::RefPtr<blink::PlatformMessage> message) const {
  fml::RefPtr<blink::PlatformMessageResponse> completer = message->response();
  auto it = message_handlers_.find(message->channel());
  if (it != message_handlers_.end()) {
    FlutterBinaryMessageHandler handler = it->second;
    NSData* data = nil;
    if (message->hasData()) {
      data = GetNSDataFromVector(message->data());
    }
    handler(data, ^(NSData* reply) {
      if (completer) {
        if (reply) {
          completer->Complete(GetMappingFromNSData(reply));
        } else {
          completer->CompleteEmpty();
        }
      }
    });
  } else {
    if (completer) {
      completer->CompleteEmpty();
    }
  }
}
```

从`message_handlers_`中取出`channelName`对应的` handle` 并执行，`handle `完成后，将结果回调给` Flutter`。

### 2.3 总结套路

这里我们就梳理出了一般的业内套路，大伙儿其实也看出来了，无非就两步：

1. 函数注册，在启动的时候注册，数据结构一般使用字典；
2. 函数调用，通过消息转发或实现类似的调用协议。

## 3. ABCBinding 的结构设计

> 计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决。
> Any problem in computer science can be solved by another layer of indirection.

`ABCBinding` 的结构如下图所示：（可以看出，`ABCBinding`其实就是个中间层或中间件）

![ABCBinding](http://qiniu.lijianfei.com/uPic/ABCBinding.png)



## 4. ABCBinding iOS 侧如何使用

1. 添加到工程；

   ```ruby
   source 'http://git.code.oa.com/abcmouse/client/cocoapods/ABCCocoapodsRepos.git'
   pod 'ABCBinding'
   ```

2. 实现代理协议`ABCBindingCocosEvalStringProtocol`,具体实现参加`ABCBindingCocosEvalStringProtocol.h`

3. 引入`JS`到`Cocos`工程

	```typescript
	import { Binding } from '../ABCKit-lib/abc';
	```

4. 编写一个`ABCBinding`接口，例如，我们定义一个获取手机信息接口 我们约定名称 `getHardwareInfo` 无传入参数，返回手机型号，`OS`版本。在`TS`中调用：

   ```typescript
   import { Binding } from '../ABCKit-lib/abc';
   Binding.callNativeMethod('getHardwareInfo').then((hardwareInfo) => {
     //get hardwareInfo 
        console.log(hardwareInfo.brand);
        console.log(hardwareInfo.model);
        console.log(hardwareInfo.OsVersion);
   })
   ```

5. 在` iOS` 任意`.m`文件中编写实现，只需要实现`@ABCBinding(FUNCTION)和@end`即可，函数名称编译器会帮助实现；

   ![ABCBinding_iOS_Code](http://qiniu.lijianfei.com/uPic/ABCBinding_iOS_Code.gif)

   ```objc
   @ABCBinding(getHardwareInfo)
   + (void)getHardwareInfo:(NSDictionary *)JSParam Callback:(ABCBindingCallBack *)callback {
       callback.onSuccess({
         @"brand":@"brand",
         @"model":@"model",
         @"OsVersion":@"OsVersion"
       });
   }
   @end
   ```

## 5. ABCBinding iOS 侧的一些实现细节

了解了业内的套路，我们来尝试实现`ABCBinding`的函数注册和函数调用;

### 5.1 使用` __attribute__`实现编译期自动注册

首先是函数注册，我们同样使用一个`map`数据结构来存储函数的注册的`class`和`method`;

我们这里的方案是编译期写入注册信息，在代码上，一个启动项最终都会对应到一个函数的执行，所以在运行时只要能获取到函数的指针，就可以触发启动项。方案核心思想就是在编译时把数据（如函数指针）写入到可执行文件的` __DATA` 段中，运行时再从 `__DATA` 段取出数据进行相应的操作（调用函数）。

程序源代码被编译之后主要分为两个段：程序指令和程序数据。代码段属于程序指令，`data` 和 `.bss` 节属于数据段。

![](http://qiniu.lijianfei.com/uPic/3.png)

`Mach-O` 的组成结构如上图所示，包含了 `Header`、`Load commands`、`Data`（包含 `Segment` 的具体数据），我们平时了解到的可执行文件、库文件、`Dsym` 文件、动态库、动态链接器都是这种格式的。

相关技术实现：

1. ` __attribute__`

   `Clang` 提供了很多的编译函数，它们可以完成不同的功能，其中一项就是 `section()` 函数，`section()` 函数提供了二进制段的读写能力，它可以将一些编译期就可以确定的常量写入数据段。在具体的实现中，主要分为编译期和运行时两部分。在编译期，编译器会将标记了 `attribute((section())) `的数据写到指定的数据段中，例如写一个`{key(key可以代表不同的启动阶段), *pointer}` 对到数据段。到运行时，在合适的时间节点，在根据 key 读取出函数指针，完成函数的调用。

   `Clang Attributes` 是 `Clang` 提供的一种源码注解，方便开发者向编译器表达某种要求，参与控制如 `Static Analyzer`、`Name Mangling`、`Code Generation` 等过程，一般以` __attribute__(xxx) `的形式出现在代码中；为方便使用，一些常用属性也被 `Cocoa` 定义成宏，比如在系统头文件中经常出现的 `NS_CLASS_AVAILABLE_IOS(9_0)` 就是 `__attribute__(availability(...)) `这个属性的简单写法。编译器提供了我们一种` __attribute__((section("xxx段，xxx节")`的方式让我们将一个指定的数据储存到我们需要的节当中。

   `used`的作用是告诉编译器，我声明的这个符号是需要保留的。被`used`修饰以后，意味着即使函数没有被引用，在`Release`下也不会被优化。如果不加这个修饰，那么`Release`环境链接器会去掉没有被引用的段。

   通常情况下，编译器会将对象放置于`DATA`段的`data`或者`bss`节中。但是，有时我们需要将数据放置于特殊的节中，此时`section`可以达到目的。

   `constructor`：顾名思义，加上这个属性的函数会在可执行文件（或` shared library`）`load`时被调用，可以理解为在` main()` 函数调用前执行。

2. 编译期写入数据

   首先我们定义函数存储的结构体，如下，`function` 是函数指针，指向我们要写入的函数。

   ```C++
   struct ABCBinding_Function {
       void (* _Nonnull function)(void);
   };
   ```

   将包含函数指针的结构体写入到我们指定的数据区指定的段 `__DATA`, 指定的节 `__ABCBinding`，方法如下(这里我们定义了个宏)：

   ```objc
   #define ABCBinding(JSFunc)\
   ...\
   static void _ABCBinding##JSFunc##load(void); \
   __attribute__((used, section("__DATA,__ABCBinding"))) \
   static const struct ABCBinding_Function __FABCBinding##JSFunc = (struct ABCBinding_Function){(void *)(&_ABCBinding##JSFunc##load)}; \
   static void _ABCBinding##JSFunc##load(){ \
       dispatch_barrier_async(ABCBinding.moduleClassesSyncQueue, ^{\
           [ABCBinding.moduleClasses setObject:NSStringFromClass(JSFunc_name(ABCBinding##JSFunc)) forKey:NSStringFromSelector(@selector(JSFunc:Callback:))];\
       });\
   }
   ```

   将工程打包，然后用 `MachOView `打开 `Mach-O` 文件，可以看出数据写入到相关数据区了，如下

   ![](http://qiniu.lijianfei.com/uPic/image-20210518195746740.png)

3. 运行时读出数据。

   如果要到 `main` 之前的阶段，之前我们是使用 `load` 方法，现在使用` __attribute__` 的 `constructor `属性也可以实现这个效果，而且更方便，并且此时**所有 `Class `都已经加载完成**。相关代码如下：

   ```objc
   __attribute__((constructor))
   void premain() {
       NSMutableArray<ABCBindinLaunchModel *> *arrayModule = modulesInDyld();
       for (ABCBindinLaunchModel *item in arrayModule) {
           IMP imp = item.imp;
           void (*func)(void) = (void *)imp;
           func();
       }
   }
   ```

   这里我们调用了`func()`，`func()`里实现了向`map`中注册`class`和`method`，至此，我们就完成了函数的注册。

### 5.2 反射 + Runtime 函数调用

接下来就是函数调用，在`iOS`首先想到的就是`Runtime`函数调用，代码如下：

```objc
+ (NSNumber*)executeWithMethodName:(NSString*)methodName
                            args:(NSString*)args
                        callback:(NSString *)callback {
    // 拿到需要反射的字符串
    NSString *method = [NSString stringWithFormat:@"%@:Callback:",methodName];
    NSString* className = nil;
    if (_moduleClasses) {
       // 这里就是从我们的map中读取相关class,函数名称是又JS方传过来的
        className = [_moduleClasses objectForKey:method];
    }
    if (!className) {
        return @(EXECUTE_RESULT_ERROR_CODE_METHOD_NOT_DEFINED);
    }
    
    // 反射得到运行时class和方法选择子
    SEL sel = NSSelectorFromString(method);
    Class cls = NSClassFromString(className);
    
    if (![cls respondsToSelector:sel]) {
        return @(EXECUTE_RESULT_ERROR_CODE_METHOD_NOT_DEFINED);
    }
    
    NSError *error = nil;
    NSDictionary *dicArgs = [ABCBindingUtil dictWithJsonString:args error:error];
    if (error) {
        return @(EXECUTE_RESULT_ERROR_CODE_PARAMS_ERROR);
    }
    
    ABCBindingCallBack *bindingCallback = [[ABCBindingCallBack alloc] init];
    bindingCallback.JSCallBackName = callback;
    // 执行函数
    SuppressPerformSelectorLeakWarning(
        [cls performSelector:sel withObject:dicArgs withObject:bindingCallback];
    );
    
    return @(EXECUTE_RESULT_SUCCESS);
}
```

### 5.3 宏定义实现类注解，约束代码规范

众所周知，在`Objective-C`中是没有注解的，但是看到`Java`里注解用的那个爽，咱们`iOSer`也是真馋了。

这里看下`ABCBinding`完整的宏定义：

```objc
#define JSFunc_name(JSFunc) NSClassFromString(@#JSFunc)

#define ABCBinding(JSFunc)\
protocol ABCBinding##JSFunc##Protocol <NSObject> \
@required\
+ (void)JSFunc:(NSDictionary*)JSParam Callback:(ABCBindingCallBack*)callback;\
@end\
@interface ABCBinding##JSFunc : NSObject<ABCBinding##JSFunc##Protocol> @end\
@implementation ABCBinding##JSFunc\
- (void)dealloc{}\
static void _ABCBinding##JSFunc##load(void); \
__attribute__((used, section("__DATA,__ABCBinding"))) \
static const struct ABCBinding_Function __FABCBinding##JSFunc = (struct ABCBinding_Function){(void *)(&_ABCBinding##JSFunc##load)}; \
static void _ABCBinding##JSFunc##load(){ \
    dispatch_barrier_async(ABCBinding.moduleClassesSyncQueue, ^{\
        [ABCBinding.moduleClasses setObject:NSStringFromClass(JSFunc_name(ABCBinding##JSFunc)) forKey:NSStringFromSelector(@selector(JSFunc:Callback:))];\
    });\
}
```

![](http://qiniu.lijianfei.com/uPic/ABCBinding_iOS_Code.gif)

为了能实现类似注解的功能，`ABCBinding`使用了`@protocol xxx @end`的关键字。这样就实现了三个功能：

1. 可以在任意`.m`文件中使用，独立于文件的其他实现；
2. 使用`@required`关键字，实现了编译器代码提示和代码生成；
3. 由于每个`JSFunc`被定义成了一个类，那么如果`JSFunc`相同的话，编译器也会报类重复的错，实现了函数的唯一性。

## 6. 关于App Store 提审风险

在`Cocos`的[官方文档](https://docs.cocos.com/creator/manual/zh/advanced-topics/oc-reflection.html)中有一段这个描述：

**警告**：苹果 `App Store` 在 2017 年 3 月对部分应用发出了警告，原因是使用了一些有风险的方法，其中 `respondsToSelector:` 和 `performSelector:` 是反射机制使用的核心 API，在使用时请谨慎关注苹果官方对此的态度发展，相关讨论：[JSPatch](https://github.com/bang590/JSPatch/issues/746)、[React-Native](https://github.com/facebook/react-native/issues/12778)、[Weex](https://github.com/alibaba/weex/issues/2875)

小弟个人是经历了`JSPatch`的从兴起到衰落的完整过程，关于这个警告，苹果重点是避免某些不法分子通过反射调用私有`API`，从而侵害用户的隐私数据。（`JSPatch`这里是完全可以做得到绕过苹果的审核机制，所以被封杀了）。

而`React-Native`和`Weex`的相关应用目前是可以在`App Store`上架的。其原因也是这些框架中，并没有提供没有任何校验的反射调用。

而`ABCBinding`这里每一个`brigde`都是一个参与编译的`class`类，苹果静态代码扫描是完全可以扫得到的。而这里使用的反射和和`Runtime`调用也都是系统公开的`API`，所以这里整体就是使用系统公开的`API`调用可以编译的类，并不会调用私有`API`。

## 7. 致谢

一起开发`Cocos`端和`Android`端的小伙伴：@hughxnwang @bearhuang
