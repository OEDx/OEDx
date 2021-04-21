---
title: ABCBinding -- 简化Cocos和Native交互利器
date: 2021-04-20 11:31:56
tags: [Cocos, 热更新, 注解]
categories: 客户端
author: "黄蓓(bearhuang)"
---

> Cocos 和 Native 交互好复杂，能不能简单一些？我们尝试着使用注解来解决这个问题。

## 背景
我们在使用 Cocos 和 Native 进行交互的时候，发现体验并不是特别的友好。
如下所示，为我们项目当中的一段代码（代码已脱敏），当检测到发生了 js 异常，我们需要通知 Native 端去做一些处理。
```
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
1. 方法调用非常麻烦，特别是 Android 端，需要明确的指出方法的路径，方法名，参数类型，特别容易出错；
2. 一旦 Native 端的方法发生变化，Cocos层必须要同步修改，否则就会出现异常，甚至有可能 crash；
3. 不方便回调；

我们尝试着用注解的思路来解决这些问题，从而设计并实现了 ABCBinding 。ABCBinding 能够极大的简化 Cocos和 Native 的交互，方便维护。

## ABCBinding 的结构设计
ABCBinding 的结构如下图所示：

![](/images/ABCBinding/ABCBinding.png)

ABCBinding 包含 Native 和 TS 两个部分，Native 负责约束本地的方法，TS 负责对 Cocos 层提供调用 Native 方法的接口。
我们重新定义了一套 Cocos 和 Native 的交互模式：
1. 提供 Cocos 访问的 Native 方法必须为 public static，且参数必须为 Transfer（Transfer 为 SDK 提供的接口，能够在 Cocos 层和 Native 层传递数据）；
2. 方法必须使用 ABCBinder 注解修饰，并在注解内传入约定好的 tag，此 tag 用于唯一标识这个方法；
3. 使用 SDK 提供的 callNativeMethod 方法，传入约定好的 tag 调用 Native 方法。

例子：
如下所示，为调用 Native 方法去下载，我们只需要传入约定好的 tag：downloadFile，并传入参数，便可以进行下载了。
TS 层：
```
Binding.callNativeMethod('downloadFile', { url: 'https://xxx.jpeg' }
).then((result) => {
    this.label.string = `下载成功:path=${result.path}`;
}).catch((error) => {
    this.label.string = error.msg;
});
```
Native 层：
```
@ABCBinder("downloadFile")
public static void download(Transfer transfer){
    new Thread(new Runnable() {
        @Override
        public void run() {
            String url = transfer.get("url","");
            try{
                //下载中
	        ...
                //下载成功
                TransferData data = new TransferData();
                data.put("path","/xxx/xxx.jpg");
                transfer.onSuccess(data);
            }catch (Exception e){
                //失败
                transfer.onFailure(e);
            }
        }
    }).start();
}
```
通过例子可以看到，使用 ABCBinding 能够让 Cocos 和 Native 的交互简单很多，我们再也不用再传入复杂的路径和参数了，而且回调也变得很优雅。接下来我们来看看 ABCBinding 是如何实现的。

## 具体实现
从上面的例子我们可以看出，ABCBinding 是通过 tag 来实现 Cocos 和 Native 进行交互的，那 SDK 是如何通过 tag 来找到对应的方法呢？
### 通过 tag 找到 Native 方法
我们定义了编译时注解 ABCBinder，
```
@Retention(RetentionPolicy.CLASS)//编译时生效
@Target({ElementType.METHOD})//描述方法
public @interface ABCBinder {
    String value();
}
```
在编译期间会生成一个类 ABCBindingProxy，成员变量 executors 包含了所有对应的 tag 和方法。其中真正可执行的方法被包装在了 ExecutableMethod 接口当中。
```
//以下为自动生成的代码
private Map executors = new java.util.HashMap();

private ABCBindingProxy() {
  executors.put("test2",new ExecutableMethod() {
    @Override
    public void execute(Transfer t) {
      com.example.abcbinding.MainActivity.test2(t);
    }
  });
  executors.put("test1",new ExecutableMethod() {
    @Override
    public void execute(Transfer t) {
      com.example.abcbinding.MainActivity.test1(t);
    }
  });
}
```

```
public interface ExecutableMethod {
    void execute(Transfer t);
}
```

因此我们只需要通过 executors 和 tag 就可以找到了对应的方法了，接着我们看看 TS 是如何与 Native 进行交互的，
以下是 SDK 里面 TS 层的部分代码，SDK 屏蔽了调用的具体细节，将请求的参数转变成为 json 字符串，并将相关的参数传递给 SDK 内部的方法 execute，由 execute 将请求转发给真正的方法。

``` 
public callNativeMethod(methodName: string, args ?: Record<string, string | number | boolean>): Promise < any > {
    ...
    if (cc.sys.isNative && cc.sys.os === cc.sys.OS_ANDROID) {
        let resultCode = jsb.reflection.callStaticMethod(
            'com/tencent/abckit/binding/ABCBinding', 
            'execute', 
            '(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)I',
            methodName, 
            typeof args === 'object' ? JSON.stringify(args) : '', cbName);
        ...
    } else if (cc.sys.isNative && cc.sys.os === cc.sys.OS_IOS) {
        let retId = jsb.reflection.callStaticMethod(
            'ABCBinding', 
            'execute:methodName:args:callback:',
            methodName, 
            typeof args === 'object' ? JSON.stringify(args) : '', cbName);
        ...
    } else {
        let msg = 'no implemented on this platform';
        reject(msg);
    }
}
```
这样，我们就解决了通过 tag 找到对应方法的问题。但还有两个问题需要解决：
如何约束 Native 方法？以及如何保证 tag 的唯一性？

### 约束 Native 方法
ABCBinding 规定，提供 Cocos 访问的 Native 方法必须为 public static，参数必须为 Transfer，且 tag 必须要保持唯一性，那怎么来约束呢？
在代码的编译期间，我们会去检查所有被 ABCBinder 修饰的方法，若发现这个方法并不符合我们的规范，则会直接报错。
如下所示:
**1.参数错误**
```
@ABCBinder("test1")
public static void test1(Transfer transfer, int a) {
    Log.i("测试", "test()");
}
```
![](/images/ABCBinding/test1.png)

**2.方法非 static**
```
@ABCBinder("test2")
public void test2(Transfer transfer) {
    Log.i("测试", "test()");
}
```
![](/images/ABCBinding/test2.png)

**3.方法非 public**
```
@ABCBinder("test3")
protected static void test3(Transfer transfer) {
    Log.i("测试", "test()");
}
```
![](/images/ABCBinding/test3.png)

**4.tag 重复**
```
@ABCBinder("helloworld")
public static void test1(Transfer transfer) {
    Log.i("测试", "test()");
}

@ABCBinder("helloworld")
public static void test2(Transfer transfer) {
    Log.i("测试", "test()");
}
```
![](/images/ABCBinding/test4.png)

### 优雅的回调
SDK 会在编译期间自动生成 callJs 方法，所有的回调都是通过 callJs 方法实现。Native 方法只需要调用 Transfer 所提供的回调接口，便可以轻松的将结果回调给 Cocos。由于 callJs 代码是自动生成，所以 SDK 不需要直接依赖 Cocos 库，只需要业务方依赖即可。

```
//以下为自动生成的代码
public void callJs(final String globalMethod, final TransferData params) {
  org.cocos2dx.lib.Cocos2dxHelper.runOnGLThread(new Runnable() {
    @Override
    public void run() {
      String command = String.format("window && window.%s && window.%s(%s);", 
	  globalMethod,globalMethod, params.parseJson());;
      org.cocos2dx.lib.Cocos2dxJavascriptJavaBridge.evalString(command);;
    }
  });
}
```

ABCBinding 提供了onProcess，onSuccess 和 onFailed 回调方法，
以下为 downloadFile 接口的的回调示例：
TS 层：
```
Binding.withOptions({
    //回调onProcess
    onProgress: (progress) => {
        let current = progress.current;
        this.label.string = `progress:${current}`;
    }
}).callNativeMethod(
    'downloadFile',
    { url: 'https://xxxx.jpg' }
).then((result) => {
    //回调onSuccess
    this.label.string = `下载成功:path=${result.path}`;
}).catch((error) => {
    //回调onFailed
    if (error) {
        this.label.string = error.msg;
    }
});
```
Native 层：
```
@ABCBinder("downloadFile")
public static void download(Transfer transfer){
    new Thread(new Runnable() {
        @Override
        public void run() {
            String url = transfer.get("url","");
            try{
              	//模拟下载过程
                for(int i =0;i<100;i++) {
                     transfer.onProgress("current", i);
                }
	       //下载成功
                TransferData data = new TransferData();
                data.put("path","/xxx/xxx.jpg");
                transfer.onSuccess(data);
            }catch (Exception e){
                //失败
                transfer.onFailure(e);
            }
        }
    }).start();
}
```

## 其他 feature
### 抹平系统差异
使用 ABCBinding 无须再判断当前系统是 Android 还是 IOS ，只需对应 Native 方法的 tag  保持一致即可。
```
Binding.callNativeMethod('isLowDevice').then(({isLowDevice}) => {
  console.log(isLowDevice);
})
```
### 无需关心线程切换
Native 方法使用 Transfer 回调时，ABCBinding 会自动切换到 Cocos 线程执行，无需业务方关心。
```
@ABCBinding("getHardwareInfo")
public static void getHardwareInfo(Transfer transfer) {
    TransferData data = new TransferData();
    data.put("brand", Build.BRAND);
    data.put("model", Build.MODEL);
    data.put("OsVersion", Build.VERSION.RELEASE);
    data.put("SdkVersion", Build.VERSION.SDK);
    transfer.onSuccess(data);
}
```
### 支持超时
ABCBinding 支持设置超时，其中超时的时间单位为秒，如下所示，超时会回调到 catch 方法当中。
``` 
Binding.withOptions({
    timeout: 120,
    onProgress: (progress) => {
        let current = progress.current;
        this.label.string = `progress:${current}`;
    }
}).callNativeMethod(
    'downloadFile',
    { url: 'https://xxxx.jpg' }
).then((result) => {
    this.label.string = `下载成功:path=${result.path}`;
}).catch((error) => {
    if (error.code == ERROR_CODE_TIMEOUT) {
         console.log("超时");
    }
});
```
## 彩蛋：在热更新当中的应用
我们在使用 Cocos 热更新服务的过程中发现，怎么确定 Cocos 热更新包能够发布到哪个 App 版本，是个难题。Cocos 热更新包能不能在这个 App 版本上正确运行，跟两个因素有关，Cocos 版本和 Native 接口。
**Cocos 版本**
如果 App 和热更包的 Cocos 版本不一致，那么很有可能这个热更包无法在 App 上运行，除非官方做了一些兼容处理。不过这个因素可控，Cocos 的版本不会频繁的升级，而且我们知道  Cocos 版本和 App 版本的对应关系。
**Native 接口**
如果热更包调用了某个 Native 的接口，但是这个接口在有些版本上不存在，那该热更包就无法在这个版本的 App 上运行。
在腾讯开心鼠的业务场景当中，Cocos 版本不会频繁变更，但是每个版本的 Native 代码可能会相差较大，人工来核对每个版本的 Native 接口变更是一件极为费时费力的事情。
那么 ABCBinding 能帮助我们做什么呢？
### 让热更包兼容所有版本的 App 
首先我们去除 Cocos 版本的因素，因为这个因素可控，且业务方无法解决。
ABCBinding 知道本地有哪些接口可用，所以当 Cocos 调用了一个不存在的接口时，我们会返回一个特殊的 code，这样热更包只需要在内部做兼容处理就可以了。
如下所示：
``` 
Binding.callNativeMethod('downloadFile', { url: 'https://xxx.jpeg' }
).then((result) => {
     //处理逻辑
}).catch((error) => {
    if(error.code == ERROR_CODE_METHOD_NOT_DEFINED){
        console.log("未找到该方法");
    }
});
```
### 元素绑定
既然热更新跟这两个元素有关，我们就可以通过这两个元素，让 App 和热更包进行匹配，如果能够匹配，那么这个热更包就可以下发到这个版本的 App 上。
**Cocos 版本**：我们在打包的时候就可以确定；
**Native 接口**：在打包的过程中，ABCBinding 可以将当前所支持的接口，按照约定的方式生成一个元素。
例如：本地的接口有 test1，test2，test3 我们可以将接口按照指定的方式排序拼接取 md5 值，生成第二个元素。
这样当 App 和热更新包的两个元素能够匹配时，就能够下发了。
![](/images/ABCBinding/factor.png)

## 致谢
一起开发的小伙伴：
@hughxnwang @ezli



