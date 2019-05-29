---
title: 前端学Serverless系列--性能调优
date: "2019-05-29 16:47:35"
tags: [Serverless, SCF, 性能调优]
categories: 前端
author: "龙永霞(yongxialong)"
---

导语：Serverless云函数的优点是不怕高并发，理论上无限自动扩容，缺点是冷启动特性导致冷启动的时延比较高。那么实际上性能如何，并且是否还有性能优化的空间和手段呢？

最近试点Serverless的一个项目是从原有的node服务迁移到腾讯云函数Serverless的。既然是项目迁移，那么就要对比一下迁移前后的性能了。

## 压测方案

从测试同事那很快就找到压测大师这个工具，压测大师配置和报告都还算比较完善，是腾讯出的，内部的话用企业微信登录直接免费可以使用。

缺点是：发请求的时间间隔有限制，最大只能隔1分钟，那就无法纯测试云函数的冷启动了，不过每个阶段都有自动增加人数，页能间接测试到冷启动到性能，看到测试结果时不时飙升到高耗时，那就多数就是冷启动的耗时冷。

压测大师链接：https://wetest.qq.com

如果要测200以内的并发人数的话，可以直接测试了。如果需要测高于200人数的并发量的话，需要先进行域名验证(具体可以看压测大师提示并操作即可)，就是放一个key到你的域名根目录下，可以让压测大师可以访问到，不然让你随便压别人的域名怎么办？

这又难倒了我。因为腾讯云自动生成的api网关链接下必然会有http://yourdomain<font color=#FF0000>/release/</font>这个环境的路径，根本无法将key放到根目录下。

腾讯云也提供了一个路径映射的功能，可以将这个<font color=#FF0000>/release/</font>的路径去掉，但是这个功能绑定在了自定义域名中，就是说，首先你得有一个自己的域名。

![](/images/serverless-performance/domain.png)

这个问题和腾讯云的产品反馈之后，将这个需求加到了排期中。

暂时我从同事那借用了一个子域名先进行继续将测试工作进行下去。

注意：需要在腾讯云备案了的域名。可以绑定一个子域名，在个人的域名管理界面中，通过cname指向腾讯云的API网关域名即可。

如果你要支持https的话，还需要申请个证书。有免费的可以申请，同一个主域名可以申请20个。申请完成之后，将证书上传到腾讯云即可。

![](/images/serverless-performance/crt.png)

如果不需要https都话，直接选http就可以了，准备工作都OK了。

## 评测及优化
选择测试的CGI，初步是要比较两者的链路，所以选取一个简单的逻辑，只有一个数据库查询的CGI。

压力从低到高进行尝试。

刚尝试到200并发人数到时候，成功率就有点不对劲了，<font color=#F00>只有80%多</font>？

![](/images/serverless-performance/fail.png)

### 分析问题

后来查了一下云函数的日志，问题就比较明了了。

![](/images/serverless-performance/resource_limit.png)

虽然说是理论上可以无限扩容，但是也要配置给不给你这个上限。

### 优化方案

如果有需求可以联系腾讯云的同事进行相关资源的申请。

在提高限额和初始化一个资源池待用之后，再来压测一下：

![](/images/serverless-performance/add_resource.png)

![](/images/serverless-performance/add_resource_bad.png)

从上图可以看到成功率上去了，但是平均耗时和耗时长的部分还是不少。

### 分析问题
将耗时比较长的拿出来和云函数的开发一起分析一下，耗时耗在哪里了。

1、 有资源池，冷启动的情况下：

        1）资源申请：代码比较大，代码解压+load代码 约2.5s+（解压2.4s），如果加上下载估计会在3s多。

        如果调用很频繁，正常情况会有2、3百个请求（资源池容器数量）资源申请时间在2.5-3s之间； 如果调用不频繁，耗时2.5s-3s的请求数会更多一些。

        2）函数调用：代码本身执行时间：70ms~115ms左右，路径耗时从5ms~600ms不等

2、热启动：也就是代码本身执行的时间，这个看代码本身要做的逻辑。机器不是瓶颈，主要是逻辑和需要调用第三方应用的耗时。我这个压测例子的耗时就会在50ms之内，20ms左右。

3、现在有一些大于5s的，那些耗时都在路径上或者有可能在单节点处理能力上。具体消耗在哪里暂时看不出来。

### 优化方案

1、设置实例保留，减少冷启动。这个最有效，降幅最大，相当于是保留了一个进程随时响应请求。

2、设置合适的资源池数量，可以大大降低冷启动的耗时。

3、减少代码量，我这个例子代码量从58M降低了26M，主要是将并不常用而且代码量比较大的库拆分成一个单独的云函数服务。

理论上结果应该会好看很多了？

实际上耗时最大的请求的确有所改善，但是平均值和90%的值还是被一些高耗时拉高。

![](/images/serverless-performance/add_instance_persist.png)

但是实际上压测大师测的结果依然没有达到很理想，下面汇总一下截止目前阶段的结果。

![](/images/serverless-performance/add_instance_persist_summary.png)

### 继续分析问题：

理论上云函数服务该做的优化都做了，而且理论表现不会这么差才对？

后来云的同事用python写了脚本来自己压测，发现200并发的平均耗时在200ms之内，而且超过200ms的请求也在可数之内。

由于两个测试的结果不一致，从自己写的脚本和wetest压测方案两方展开分析。

1）脚本尽量去还原wetest的方案，如异地部署，阶梯增加并发，并且每个阶段维持30s。

2）直接找到了wetest的后台同事进行问题沟通，尝试了长连接，短连接，多IP压力源，去掉日志打印等操作。

结果：这个压测结果其实已经达到了我们预设的优化目标，平均在200ms之内。

以下还是wetest的压测结果（短连接+多IP）。

![](/images/serverless-performance/wetest_correct.png)

其中还有一个很有意思的结果，长连接下的压测结果：

![](/images/serverless-performance/wetest_correct_long_connect.png)

**那么到底是什么原因导致了这个结果的差异呢？**

<font color=#F00>“网络连接耗时”</font>

在这个例子中，将长连接改成短连接，从一地压测改成多IP压测，效果最为明显，去掉日志打印也一定程度减少了压测源的性能损耗。

## 用户侧对比评测
既然地域不同，网络不同，耗时差异这么大，那么真实的用户耗时到底如何呢？

![](/images/serverless-performance/user_performance.png)

红色的是Serverless，黄色的是原来的NodeServer。

### 分析问题

#### 1、5月5日针对Serverless做了优化：将非简单请求改成简单请求，减少一次“预检”请求，加上dns-prefetch；加上了NodeServer原来的请求耗时进行对比。

刚开始是因为设置了header头，浏览器认为这个是非简单请求。

非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。更详细可以查阅：[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

![](/images/serverless-performance/preflight.png)

图为一个请求产生了两个请求，第一个是预检请求，其中method为OPTION， 返回状态码是204。


如果要用到非简单请求的时候，服务端响应CORS的设置要注意到：

![](/images/serverless-performance/cors_setting.png)

如果单选具体的GET，POST，可以在API网关中的API管理中设置是否支持CORS，如果需要支持多个的请求方法的话，就只能后端业务处理。

#### 2、优化之后，发现Serverless的请求依然比NodeServer慢100ms左右。

继续分析发现：Serverless的请求会比NodeServer请求多一个Initial connection和SSL的时间。

为什么NodeServer没有呢，因为我们测试的页面和发请求的页面是同域的，所以这部分时间省了。为什么之前Serverless没有发现呢，因为socket连接之后没有那么快销毁，在页面重复刷新的时候，反而是看不到这个耗时的。

需要清理一下socket： chrome://net-internals/#sockets

就可以复现了，那么这个耗时差距的原因就很明显了。

我做了一个简单页面进行测试，发现这样的结果。Serverless和NodeServer的全程耗时其实差不太多，但是TCP连接和SSL时间会长很多。

这是为什么呢？我测试的这个Serverless是部署在上海的，而我在深圳进行测试的。

我再做了一个测试，加了一个部署在广州的云函数。

结果也很明显了。

异地访问部署的问题，因为原有的NodeServer接入了STGW。

STGW全称Secure Tencent Gateway，腾讯安全云网关，是一套实现多网统一接入，支持自动负载均衡的系统。由于STGW提供了就近接入和SSL加速，所以这两部分时间都相应有不少的优化。

那么Serverless能不能用多地部署，就近接入来解决这个问题呢。

目前Serverless的云函数和API网关都是地域隔离的。也就是说广州的API网关对应广州的云函数，不能一个网关对多个地域的云函数。这个需求和腾讯云的产品沟通之后，纳入到以后的规划建设当中。

那么我们目前有没有折中的方案呢？

我们设想了一些中转的方案：

![](/images/serverless-performance/stgw.png)

最后我们选择了最右侧的方案，主要是简单。

而且还可以继续通过在header中设置preconnect减少TCP握手和SSL的时间。

`<link rel="preconnect" href="//example.com">`

preconnect的兼容性情况：

![](/images/serverless-performance/caniuse.png)

使用之前：

![](/images/serverless-performance/no_preconnect.png)

使用了之后的效果：

![](/images/serverless-performance/preconnect.png)

可以看到Initial connection 和 SSL的时间是可以直接节省掉了，在网络差的情况下，这部分节省的时间更为显著。



## 小结一下

**Serverless云函数性能评测和优化结果：**

经过优化之后，Serverless云函数的响应性能已经达到了需要即时返回场景的可用状态。

在API网关监控到到耗时（不包括网络时间和握手时间）

![](/images/serverless-performance/api_monitor.png)


1、压测方案、问题分析
   
    1）wetest是腾讯公司支持的通用压测方案，要用好也要对里面提供的设置了解清楚用法和用途。

    2）测试不限于压测工具，本地脚本，浏览器network分析，线上用户test，等等都可以尝试用来评测分析。

2、性能优化方案

**云方面的优化：**

    1）上限设置

    2）专用资源池

    3）实例保持

    4）架构升级（测试中）

    5）就近接入（待规划）

**代码方面的优化：**

    1）代码逻辑和第三方调用

    2）拆分不常用功能，减少代码量

    3）使用预加载减少connect的时间（preconnect）

    4）接入STGW进行转发

## 最后
1、云函数目前还在起步阶段，应用的过程中还会遇到这样那样的问题，好在腾讯云的同事们（@masonlu等）积极响应，主动和我们一起寻找问题解决问题，最终一起共同成长。

2、云函数目前的特点就是不适合对时延要求比较高的应用，不适合有状态的应用，但是这个不是必然的，这两个问题有来合适的方案之后，就不会再是问题。
