---
title: 基于 Cocos 的高性能跨平台开发方案
date: 2019-05-10 17:01:35
tags: Cocos
categories: 客户端
---

大概从去年九月份开始，我们选择使用Cocos来作为我们一款产品 ABCmouse 的跨平台应用开发方案，在这个过程中，我们做了一系列的优化，也踩了一些坑，本文将对这个过程做一个回顾和总结。

本文的内容主要分三块来讲。首先简单介绍一下项目背景，接下来具体介绍下我们的实践过程，分享一些经验。最后给出新的开发方案和以前的效果对比。

## 项目背景

首先介绍一下我们的产品，ABCmouse 是美国知名的儿童英语在线学习领导品牌，在美国有超过百万家庭在使用，也获得了7万多个教师的推荐。

![ABCmouse](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/abcmouse-legacy.png)

这个应用采用的是典型的 Hybrid App 跨平台开发方案，里头基本全是 H5 的页面。

![ABCmouse是一款 Hybrid App](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/hybrid-app.png)

Hybrid App 最大的问题就是性能问题，用户经常会在页面加载上等待非常多时间。

我们统计了 ABCmouse 各个场景的平均加载耗时，发现平均都要花费大约三到四秒的时间。漫长的等待时间也对用户的学习积极性带来影响。

![ABCmouse启动耗时统计](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/performance-legacy.png)

从去年九月份开始，团队与 ABCmouse 的研发公司 Age of Learning 公司开展了战略合作，我们希望能够开发出一款针对中国儿童的英语学习应用——我们称之为 ABCmouse 腾讯版。我们希望它能提供更符合中国儿童使用习惯的学习路径，并在里头融入腾讯的社交元素，从而带动儿童外语学习的积极性。

![ABCmouse腾讯版](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/abcmouse-tencent.png)

从技术上，我们希望新版的 ABCmouse 能够在表现力、性能、效率和社交四大方面都能有更好的表现（这里的表现力指的是产品的界面和交互，能够做到更吸引中国的小朋友）。

通过初期技术预研后，我们决定使用 Cocos 来改造这个项目：

1. 跨平台。Cocos 支持使用同一套代码构建生成 Web、iOS、Android 等几个端，最新的版本还支持发布到微信小游戏、Facebook Instant Games 和 QQ 玩一玩；
2. 性能。Cocos 的原理是在 Activity 中绘制一个 OpenGL 的 SurfaceView ，并由其完成页面的渲染的。与基于 WebView 渲染的 Hybrid 应用相比，Cocos 的渲染速度更快，性能更好。
3. 效率。借助可视化的 Cocos Creator 工具，界面的开发和资源的管理非常便捷，设计团队也可以参与进来设计界面和动效，提升开发效率。
4. 表现力。ABCmouse 中包含了很多诸如游戏、画图、音乐等带游戏和娱乐性质的场景，而 Cocos 本身是为游戏开发设计的，更适合用在我们的产品中。

## 具体实践

在具体实践这一块，我准备分成架构篇、甜头篇、踩坑篇、优化篇四个部分来介绍。

### 架构篇

一图胜千言。我们整个系统架构可以用这张图来概括。

![新版ABCmouse的应用架构](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/infrastructure.png)

我们自底向上看，最底层是 native 层，Cocos2d-x 开发框架,在这一层提供了对 JavaScriptCore、SpiderMonkey、V8、ChakraCore 等多种可选的 JS 执行引擎的封装。在这基础上又架设了一层 JSB ，主要起到桥接作用。我们的应用也在底层封装了多种基础能力，包括支持直出的webview、自定义的视频播放器、音频播放器、支付、推送等。

再往上是 JS 层，在这一层 Cocos 提供了丰富的开发组件和 API，我们也扩展了多种组件，包括一些通用的UI组件、一个多端通用的音频播放器、一个带缓存和内存回收功能的图片加载器、常驻节点、上报、日志等组件。有些组件是依赖 native 层的。

Cocos 层和 Native 层就通过 callStaticMethod 和 evalString 来完成互相调用。

有了这些基础后，再往上则可以开展具体的场景开发了。

为了帮助大家更好地理解 Cocos 的跨平台原理，我们可以拿 Cocos 的渲染原理和 React Native 做一个对比。

Cocos 的渲染原理是在 UI 线程将场景文件理解成场景树，然后交给 GL 线程渲染。也就是说，用户看到的大部分场景都是使用 OpenGL 或者 WebGL 绘制的，即使在不同的平台，也能够有完全相同的表现。

{% img http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cocos-rendering.png Cocos的跨平台绘制原理 %}

而 React Native 的渲染原理是将 JS/JSX 理解成 Virtual DOM，然后调用各自平台的 Widget 。由于不同的平台，底层的 Widget 表现是不同的，因此使用上可能会存在差异。这也是 React Native 为人诟病的一点。

{% img http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/rn-rendering.png React Native 的跨平台调用原理 %}

### 甜头篇

采用 Cocos 作为我们的跨平台开发框架后，我们尝到了不少甜头。

首先是跨平台带来的便利。我们使用一套代码可以生成到安卓、iOS、Web、微信小游戏等多种平台，并且在多个端达到了高度一致的体验。在 React Native 上经常遇到的 UI 体验不一致的问题，在 Cocos 开发中基本没有遇到过。

由于Cocos支持构建小游戏版本的应用，所以我们的项目也提供了小游戏版本。上周末已经有很多爸爸在微信小游戏里收到了他们的孩子使用 ABCmouse 制作的贺卡。值得一提的是，小游戏版本是我们两个开发在花了一周左右的时间内移植完成的。这里头主要的移植工作在于接入微信小游戏的登录授权，接入 VideoPlayer 和 InnerAudioContext 以分别支持视频播放和音频播放。

![微信小游戏上的父亲节贺卡](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/abcmouse-wxgame.png)

第二个甜头是开发效率的提升。

首先，Cocos 提供了可视化的 Cocos Creator ，使用它来管理和构建工程非常轻松。

![Cocos Creator](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cocos-creator.png)

其次，设计萌妹子也能直接使用 Cocos Creator 编辑动效，输出动效资源给开发，提高协作效率。

![设计妹子也使用 Cocos Creator 制作动画](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cooperation.png)

另外，Cocos Creator 支持直接在浏览器中预览调试场景，节省了大把构建编译的耗时。

![直接使用浏览器调试场景](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/browser-debug.png)

第三个甜头是热更新带来的便利。

Cocos 同时支持脚本和资源的热更新，这给我们修复线上问题、发布运营活动带来了很大便利。

此外，Cocos 的热更新可以做到 hot reload，无需冷重启，很好的保证了用户的体验。并且，Cocos 的热更新支持高度可定制，可以很方便的定制满足业务需要的热更新流程。

![ABCmouse 里的热更新](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/hot-upgrade.png)

第四个甜头是 Cocos 提供的强大的社区支持。Cocos 的开发团队来自中国，有着非常活跃的中文社区。

![Cocos 的中文论坛](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cocos-forum.png)

另外，使用 Cocos 开发小游戏也成了最主要的方式，可见 Cocos 的受欢迎程度，也侧面证明了这套开发框架的生命力。

![使用 Cocos 开发小游戏的占比](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cocos-on-wxgame.png)

## 踩坑篇

跨平台开发虽然方便，但是在一些具体的实践中难免也会踩到坑。

首先，Cocos 主要是面向游戏开发的，要使用它来开发应用，少不了需要开发一些 UI 组件。因此，我们在 Cocos 层开发了一系列的通用 UI 组件，包括对话框、选择器、表单、按钮、toast、loading 等组件，这些组件遵循一套规范化的接口标准，使用起来非常便捷灵活。


![ABCmouse里的通用UI组件](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/ui-components.png)

开发完 UI 组件后，我们发现这些组件的加载也存在问题。和原生应用开发不同，这些UI组件本质上都是挂载在场景里头的节点，如果没有调度的话，可能存在同时弹出多种弹窗和对话框的情况，整个场景就会变得很混乱。

![没有调度的情况下，可能出现场景混乱](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/dialogmanager-before.png)

为了解决这种问题，我们写了一个针对 Cocos 的弹窗调度器，统一由它来调度弹窗，避免了弹窗的混乱。

![有调度的情况](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/dialogmanager-after.png)

我们接下来遇到的另一个坑是 VideoPlayer 的置顶问题。

前面提到，Cocos 的场景是在 GL 上绘制的。例如，对于 Android 平台，Cocos 开启了一个 OpenGL 的 SurfaceView 来进行场景绘制。而这个 GLSurfaceView 不能直接支持渲染视频，所以，Cocos 提供了一个 VideoPlayer 组件用于播放视频。这个 VideoPlayer 是独立且置顶的一层。

这带来的一个问题是：无法在视频上绘制 UI 。

比如我们希望视频播放器里头能加上我们自定义的按钮、进度条，如果是直接在 Cocos 层对 VideoPlayer 进行封装的话，会发现这些 UI 元素会被视频本身遮盖，达不到定制界面的目的。


![VideoPlayer 的置顶问题](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/videoplayer-cocos.png)

最终我们放弃了直接使用 Cocos 提供的 VideoPlayer 组件，而是在底层为各个端开发视频播放器，并各自实现界面的定制。


![改为各端实现 VideoPlayer](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/videoplayer-customized.png)

视频播放问题解决了，我们又遇到了音频播放的问题。

由于应用中有非常多的音乐、音效、语音，为了减小包大小，大部分的语音素材放在 CDN 上，需要的时候才从 CDN 上拉取播放。少部分常见的音效会直接打进应用包中。而 Cocos 自带的 AudioEngine 组件在 Native 端只支持本地资源的播放。因此，我们又封装了一个跨平台的音频播放器，可以自动根据指定的音频路径决定使用播放方式：

- 对于 Web 端或者 Native 端的本地资源文件，直接使用 AudioEngine 来播放。
- 对于 Native 端的远程音频，使用 Native 的播放器来播放。
- 对于小游戏环境，则使用小游戏的 InnerAudioContext 来播放。

由于对外的接口只有一套，开发者无需考虑具体的平台和底层播放器的选择。并且可以使用同样的接口来统一管理不同的音频。

![跨平台的 AudioPlayer](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/audioplayer.png)

最后我们遇到的一个比较严重的问题是 local reference table overflow error 问题。

为了复用 Native 端的能力，我们在 Cocos 层大量地使用反射机制来调用 Native 端提供的方法。然而，我们经常会遇到 local reference table overflow error 错误导致的界面卡死问题。

```
A/art: art/runtime/indirect_reference_table.cc:138] JNI ERROR (app bug): local reference table overflow (max=512) 
A/art: art/runtime/indirect_reference_table.cc:138] local reference table dump: 
A/art: art/runtime/indirect_reference_table.cc:138] Last 10 entries (of 512): 
A/art: art/runtime/indirect_reference_table.cc:138] 511: 0x12e45170 java.lang.String "" 
A/art: art/runtime/indirect_reference_table.cc:138] 510: 0x12dd33c0 java.lang.Class<com.tencent.abcmouse.report.DcReport> 
A/art: art/runtime/indirect_reference_table.cc:138] 509: 0x12e45180 java.lang.String "" 
A/art: art/runtime/indirect_reference_table.cc:138] 508: 0x12f89490 java.lang.String "59" 
A/art: art/runtime/indirect_reference_table.cc:138] 507: 0x135a4f40 java.lang.String "1522668817662" 
A/art: art/runtime/indirect_reference_table.cc:138] 506: 0x12e89400 java.lang.String "onLoad" 
A/art: art/runtime/indirect_reference_table.cc:138] 505: 0x12e451d0 java.lang.String "" 
A/art: art/runtime/indirect_reference_table.cc:138] 504: 0x12c8bc00 java.lang.Class< 
A/art: art/runtime/indirect_reference_table.cc:138] 503: 0x12e451f0 java.lang.String "" 
A/art: art/runtime/indirect_reference_table.cc:138] 502: 0x134627f0 java.lang.String "1522668817664" 
A/art: art/runtime/indirect_reference_table.cc:138] Summary: 
A/art: art/runtime/indirect_reference_table.cc:138] 1 of android.opengl.GLSurfaceView$GLThread 
A/art: art/runtime/indirect_reference_table.cc:138] 222 of java.lang.Class (7 unique instances) 
A/art: art/runtime/indirect_reference_table.cc:138] 289 of java.lang.String (289 unique instances)
```

最初，我们怀疑是反射调用使用得太频繁导致。因此，我们对诸如打 log、事件上报等 Native 方法进行了频率限制，例如使用缓冲的方法将多个 log 合并后再打印。

然而，虽然这个做法减少了界面卡死的发生，但依然没有彻底杜绝问题的再次出现，就像是一个定时炸弹一样，威胁着我们应用的稳定性。

通过阅读引擎的代码，我们发现 Cocos 的引擎在反射阶段处理字符串参数时，使用了 `NewStringUTF()` 方法将其转换为 JNI 层的字符串，然而在调用执行完成后并没有相应地使用 `DeleteLocalRef()` 释放该字符串的引用，从而导致了引用表的溢出。

``` java
static bool JavaScriptJavaBridge_callStaticMethod(se::State& s)
{
    ……
    if (argc > 3) {
        ……
 
        if (call.isValid() && call.getArgumentsCount() == (argc - 3)) {
            ……
            for (int i = 0; i < count; ++i) {
                int index = i + 3;
                switch (call.argumentTypeAtIndex(i)) {
                    ……
                    case JavaScriptJavaBridge::ValueType::STRING:
                    default:
                        std::string str;
                        seval_to_std_string(args[index], &str);
                        jargs[i].l = call.getEnv()->NewStringUTF(str.c_str());  // 这里没有释放！！！
                        break;
                }
            }
            ok = call.executeWithArgs(jargs);            
            if (jargs) delete[] jargs;
            ……
```

了解到这个原因后，我们给 Cocos 的引擎提交了一个 pull request，修复了这个问题。

![](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/pull-request-description.png)

![pull request](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/pull-request-commit.png)

## 优化篇

虽然 Cocos 比起纯 Hybrid 的方案在性能上已经占据了优势，但是比起 native 还是有一些差距的。下面就说说我们在开发过程中尝试过的一些优化，让我们的应用做到接近原生的体验。

### 高性能的 ScrollView

官方 ScrollView 组件需要配合 layout 组件，当一次加载大量的子节点组件，或者分帧加载单个子节点组件时，初始化 ScrollView 节点视图会比较慢，在加载完成前存在拖动掉帧的问题。另外，一次性加载所有节点，也会导致内存资源的浪费。

下图这个场景是 ABCmouse 里的二级资源页，由于一次性加载了太多子节点，当屏幕滚动时，帧率降到了 8 fps 左右，给人的感受是非常卡顿。

![官方 ScrollView 处理大量子节点导致滑动卡顿](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/scrollview-legacy.png)

我们对 ScrollView 进行了重写，基本的优化思路是：一次仅加载页面可容纳的少量数目子节点。并在滚动过程中，回收不可视的子节点组件并重用。

具体来说，ScrollView 大多数情况下表现为列表组件和宫格组件，以列表组件为例，可以根据子节点数目和子节点大小，计算出整个 ScrollView 内容的宽高，同时计算出屏幕可视区域最多可以容纳的子节点行数 rows，加载时仅加载 rows + 2 个子节点组件，其中添加的 2 个字节点组件作为滚动回收缓冲。

下图是对上述思路的图例。当手势向上，内容往下滚动时，一旦最上排的子节点组件不可视，就立马将它们回收掉并将其重用于将要渲染的子节点组件中。


![高性能 ScrollView：滚动前](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/scrollview-before-scrolling.png)


![高性能 ScrollView：滚动后](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/scrollview-after-scrolling.png)


这么做的优点在于：一次仅加载页面可容纳的少量数目子节点，并且逐帧加载，能极大提升展示和滚动性能，另外大大减少了内存占用。

经过优化后，不管二级资源页场景里有多少元素需要展示，整体的帧率都维持在 60 fps 左右，非常流畅。


![高性能 ScrollView](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/scrollview-new.png)

### 内存优化

内存占用过高也是 Cocos 开发过程中很容易遇到的问题。如果没有优化好内存占用，很可能就会引发黑屏或者 OOM。


![内存不足导致黑屏](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/black-screen.png)


要优化内存占用，有几个思路。第一个思路是把内存消耗大以及没有回收的元凶先找出来对症下药。

于是，我们仿照 Cocos 的监视器也写了一个内存监视器，利用它来找出疑似存在内存泄漏的场景。


![内存监视器](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/memory-calculator.png)


对于每一个场景，我们也对每个节点的内存占用做了一个排名，找出靠前的，分析是否合理，并进行针对性的优化。比如把原图缩小，把无需透明像素的png图转换成JPG图，等等。


![节点内存排名](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/calculate-memory.png)


第二个思路是为图片渲染开启纹理压缩，从而大幅度降低图片渲染的内存占用。Cocos 提供了 ETC1、PVR 等几种纹理压缩方案，其中，PVR 兼容性最好，内存消耗也最低，但是质量较差；ETC1 不支持 iOS 的低端机型，质量也较差。我们又对 Cocos2d-x 进行扩展，增加了 ETC2 纹理压缩，这种方案的优势比起 ETC1 而言，压缩质量更好。


![三种纹理压缩方式](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/texture-compressors.png)


下图可以看到 ETC2 和 PVR 压缩质量和内存占用的直观对比。
对比原图，我们可以看出 ETC2 的压缩结果与原图相差不大，但内存减少了 75% 。而 PVR 的压缩结果相比 ETC2 言在细节方面少了很多，内存则减少了 87.5% 。


![PVR / ETC2 效果对比](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/etc2-vs-pvr.png)


针对兼容性问题，我们设计了一种混合纹理压缩方案：对于高质量要求的纹理，如果该机型能支持ETC2，就使用ETC2纹理压缩；如果不支持，就将该纹理进行大小减半压缩；对于低质量要求的纹理，使用兼容性好的PVR纹理压缩。单图渲染的内存消耗可以降低接近 75%~87.5%。


![混合纹理压缩方案（专利申请中）](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/mixture-compress.png)


纹理压缩是一项耗时的任务，所以我们把这项任务放在项目构建完后进行，而不是在客户端运行的时候才动态压缩。

我们编写了一个扩展工具，在构建完成后自动进行纹理压缩任务。后面我们发现这个工具压缩完一遍纹理要花费大概3分钟的时间，我们又改进成了增量压缩的方式，一次压缩任务缩短到10秒左右。


![纹理压缩工具（即将开源）](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/compress-plugin.png)


### drawcall 优化

每一帧的渲染耗时直接影响到整个应用的性能，而和渲染耗时相关的操作是 drawcall 。

什么是 drawcall 呢？我们可以看这张图来了解一下。在一帧的渲染过程中，场景会先被解析成场景树。场景树的每一个节点依次加入渲染队列中等待交付 GPU 渲染。GPU 接收渲染指令并执行的操作就叫做一次 drawcall。在一帧里头，drawcall 越少，性能当然就越好。


![理解 drawcall](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/drawcall-1.png)


Cocos 针对 drawcall 优化已经提供了一种自动合并技术：比如，上图中的渲染指令 1、2 来自贴图 A，3、4 来自贴图 B ,5、6、7 来自贴图 C，这些指令会被分别合并优化，最终只产生 3 次 drawcall。我们要做的就是利用好这个自动合并技术。

首先可以找出浪费 drawcall 的节点对症下药。一般可以通过把节点的 active 属性设为 false 看看 drawcall 有没有大量减少来判断。

接下来我们可以利用好 Cocos 的合并技术。

- 对于静态的 Sprite ，可以使用合并图集来减少 drawcall 。例如使用 Cocos Creator 自带的 AutoAtlas 或者第三方工具 TexturePacker 。
- 文本的动态绘制也是 drawcall 浪费的重灾区。对于 Label，可以使用 BMFont 位图字体来取代普通文本，减少 drawcall 。



![Cocos 提供的 AutoAtlas 自动图集功能](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/autoatlas.png)


![使用 BMFont 位图字体优化 Label 的 drawcall](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/bmfont-label.png)

目前这套优化方案还不能满足动态资源和动画的优化，我们也期待 Cocos 能够把 batching 技术做得更完善。

另外，还有另外一个需要注意的地方：小心避免跨层切换合图。Cocos 是按照节点层级顺序依次提交渲染指令的，如果不注重层级顺序，可能会导致贴图的切换从而浪费不必要的 drawcall 。

例如，下图中的渲染指令 4 使用的是贴图 C，直接卡在了渲染指令 3 和 5 之间，导致贴图 B 的渲染指令没法合并，从而浪费了多余的 drawcall。通过调整节点层级可以避免这个问题。


![避免跨层切换贴图](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/drawcall-2.png)


### Hybrid 页面优化

我们的应用里头目前依然存在一些原来的版本遗留下来的 H5 页面构成的场景，对于这些 H5 页面，我们也使用了一些比较常规的 Hybrid 优化技术，来达到首屏直出的要求。

因为已经有很多现有的优化方案了，所以这一块我并不打算细讲。简单为大家罗列几个技术点吧：

- 一个是使用离线缓存，对一些常用的 H5 场景，也可以离线打包进应用里头，优化首次启动速度。
- 一个是并行加载在 WebView 启动的同时并行地去拉资源，这样可以避免等待 WebView 初始化耗时对页面加载的影响。另外，还可以对一些 H5 页面进行预加载，减少等待。
- 一个是可以对页面进行少量标注，只增量更新需要动态变的部分。


![Hybrid 页面优化技术点](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/hybrid-optimize.png)


通过这一系列的优化，我们的应用里头的 H5 页面的加载耗时也能够控制在 1 秒以内。


![Hybrid 页面优化效果对比](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/hybrid-optimized-result.png)

## 整体效果对比

最后我们来看一下整体的改造效果。

项目整体的 Cocos 化率目前占到了 56%，剩下的还有 40% 的 H5 的页面（主要是一些小游戏），还有像视频这种 native 场景。


![ABCmouse 中的场景占比，其中只有Games和Videos不是Cocos](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/cocos-rewrite-rate.png)

对比原来的场景启动耗时，经过一系列改造和优化后的场景都能控制在 1 秒内启动。


![启动耗时对比（左：老版ABCmouse；右：新版ABCmouse）](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/performance-compare.png)

直接看数据不够直观，我们可以看一下原来加载耗时最长的一个场景，经过改造后做到了秒开。


![涂色场景-改造前](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/coloring-legacy.gif)


![涂色场景-改造后](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/coloring-new.gif)

而腾讯版本的包大小也比原来的版本小了 64% 。


![老版本和新版本的包大小对比](http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/size-compare.png)

欢迎扫码体验新版本的 ABCmouse ：

{% img http://hahack.com/images/cocos-based-high-performance-cross-platform-app-developing/wxgame-code.png 400 400 扫码体验ABCmouse的小游戏版本 %}
