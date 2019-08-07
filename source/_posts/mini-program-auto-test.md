---
title: 小程序自动化测试实践
date: '2019-08-07 15:40:29'
tags: [小程序, 自动化]
categories: 测试
author: 崔邹森(billcui)
---

## 一、缘起-为什么要进行小程序自动化测试

微信小程序生态日益完善，很多小程序项目页面越来越多，结构越来越复杂，业务逻辑也更加多样。以腾讯课堂小程序为例，目前腾讯课堂小程序部分页面结构和不同业务场景下的表现如下图所示：

![腾讯课堂业务](/images/mini-program-auto-test/腾讯课堂业务.png)

可以看到在核心功能上主要页面对于不同业务场景有众多不同的表现，因此在开发与发布的过程中需要手动验证大量测试用例以保证小程序按预期表现运行，善于利用工具的程序员当然会想：

这种重复的工作能不能交给程序自动进行呢？

web开发中对于这类测试问题已经有了很多自动化解决方案比如Selenium、Puppeteer，思路大体相同，都是让浏览器按照指定顺序自动在页面上完成点击、输入等操作，再将操作后的页面表现与想要得到的结果进行比较得到测试结论（断言）。那小程序中有没有一种方案能够按照这种思路实现自动化操作并提供页面信息用于断言呢？为了微信底层安全考虑，小程序环境一直比较封闭，留给开发者操作的余地很小，自动化操作基本无法实现，但5月出现了这样[一个npm包](https://www.npmjs.com/package/miniprogram-automator)，介绍了miniprogram-automator工具，给了小程序开发者希望。



## 二、缘遇-初试miniprogram-automator

基于miniprogram-automator的文档描述简单总结一下，当通过命令打开开发版微信开发者工具的自动化接口并连接自动化接口后，此工具可提供以下能力：

- **MiniProgram**：获取小程序信息（页面堆栈、系统信息、页面内容），控制小程序（跳转页面、切换tab、调用方法）
- **Page**：获取页面信息（路径、元素、数据、结构），控制页面（设置渲染数据、调用方法）
- **Element**：获取元素信息（属性、样式、内容、位置），操控元素（点击、长按、调用方法）

所以小程序自动化控制的实现依赖于开发版小程序开发者工具以及miniprogram-automator工具。小程序开发者工具命令行用来打开指定自动化操作服务端口。（开发者工具版本需高于v1.02.1906042）。miniprogram-automator工具用来操作开发者工具中运行的小程序并获取所需的信息。对于测试需求可以结合jest框架进行测试用例的组织和断言。 

不多废话，看完文档用一下：

**Ø  调用开发者工具命令行打开项目与指定自动化操作服务端口**

```bash
PS D:\内测\微信web开发者工具> ./cli.bat --auto D:\weApp\testMiniprogram --auto-port 9420
Initializing...
idePortFile: C:\Users\billcui\AppData\Local\微信开发者工具\User Data\Default\.ide
starting ide...
IDE server has started, listening on http://127.0.0.1:35510
initialization finished
Open project with automation enabled success D:\weApp\testMiniprogram
```

**这一行命令需要注意的有**：

1. 文档要求开发者工具版本号必须高于v1.02.1906042，最好是最新的内测版工具，我是在v1.03.1906062运行成功的；
2. 运行这行命令之前需要先打开开发者工具菜单中的**设置->安全设置->服务端口**；
3. 自动化端口是独立于服务端口的(比如终端打印出的35510其实是服务端口)，必须要看到`Open project with automation enabled success D:\weApp\testMiniprogram`这行提示才算是成功打开了自动化端口(9420)。

命令运行成功后，开发者工具会自动打开项目，并弹出提示

![1561688637769](/images/mini-program-auto-test/1561688637769.png)

**Ø  创建test.js，代码中引入miniprogram-automator工具，连接自动化操作端口**

```javascript
const automator = require('miniprogram-automator');

const miniProgram = automator.connect({
  wsEndpoint: 'ws://localhost:9420',
})
```

**Ø  利用miniprogram-automator提供的接口操作小程序从首页重启并进行相关操作**

```javascript
const automator = require('miniprogram-automator');

const miniProgram = automator.connect({
  wsEndpoint: 'ws://localhost:9420',
}).then(async miniProgram => {
  // 从首页重启
  const page = await miniProgram.reLaunch('/pages/index/index');
  // 从页面获取my-button组件
  const button = await page.$('my-button');
  // 打印出my-button的wxml信息
  console.log(await button.wxml());
}).catch(e => {
  console.log('catch a error', e);
});
```

**Ø  利用miniprogram-automator获取操作后页面相关信息，利用jest进行组织和断言**

```javascript
// index.spec.js
const automator = require('miniprogram-automator');

describe('课堂小程序自动化测试', () => {
  let miniProgram;
  // 运行测试前调用
  beforeAll(async () => {
    miniProgram = await automator.connect({
      wsEndpoint: 'ws://localhost:9420',
    });
  });
  // 运行测试后调用
  afterAll(() => {
    miniProgram.disconnect();
  });
  // 测试内容
  it('nohost检测', async () => {
    const page = await miniProgram.reLaunch('/pages/index/index');
    const nohost = await page.$('nohost');
    expect(nohost).toBeNull();
  });
});
```

运行`jest index.spec.js`， 如果页面中不存在nohost组件则测试通过，结果如图所示：

![img](/images/mini-program-auto-test/企业微信截图_15616886023578.png)



## 三、缘聚-自动化测试在课堂微信小程序中的应用

腾讯课堂微信小程序引入自动化测试主要是为了解决开发、预发布环境、正式环境需要反复多次打开用例课程页面，操作繁琐，耗费大量人力的问题。针对课堂小程序checklist，尽可能利用自动化测试程序完成测试验证，减少手动操作，也可以避免人为检测的遗漏。

利用miniprogram-automator工具和jest框架，自动化测试主要能力为按照指定顺序模拟**打开指定页面、点击、滚动**等操作和设置page的data渲染数据，然后对特定的**页面结构、数据、组件属性**等信息进行断言，判断是否符合预期。

下面以腾讯课堂微信小程序的课程详情页为例来详细说明在实际项目中如何实现自动化测试：

课程详情页的UI主要分为视频部分，详情部分以及底部的购买按钮，未购买课程时表现主要有以下几种：

免费课程详情页、付费课程详情页

<img src="/images/mini-program-auto-test/免费课详.jpg" width="50%" height="50%"><img src="/images/mini-program-auto-test/付费课详.jpg" width="50%" height="50%">

优惠券课程详情页、限时优惠课程详情页

<img src="/images/mini-program-auto-test/优惠券.jpg" width="50%" height="50%"><img src="/images/mini-program-auto-test/限时优惠.jpg" width="50%" height="50%">

假如对于 **未购买的无优惠活动的付费课程详情页** 的测试目标如下：

1. 按钮应显示“立即购买”，点击购买按钮可跳转到支付页
2. 点击试学按钮可正常播放试学视频
3. 未购买课程时点击课程视频无法播放

实现这个测试，在`x.spec.js`文件中首先需要要按照上文的步骤引入miniprogram-automator，在beforeAll中连接已经打开自动化端口的微信小程序项目。（这里不再重复代码，见上一章）下面直接看测试内容的代码。

1. **按钮显示和点击跳转支付页测试**

   ```javascript
   // 打开页面，通过url传参
   const page = await miniProgram.reLaunch(`/pages/course/course?cid=${commonPayCid}`);
   // 获取按钮组件信息
   const basicApplyButton = await page.$('.basic--buy');
   // 判断按钮显示内容
   expect(await basicApplyButton.wxml()).toContain('立即购买'); 
   // 模拟点击按钮
   await basicApplyButton.tap();
   // 等待页面跳转
   await page.waitFor(1500);
   // 获取当前页面路径
   const currentPage = await miniProgram.currentPage();
   // 判断跳转后路径是否正确
   expect(currentPage.path).toContain('pages/order/order');
   // 跳转回来
   await miniProgram.navigateBack();
   ```

 目前miniprogram-automator提供了两种方法获取到页面中的组件：`page.$` 和 `page.$$` 。经过实验发现两者的selector支持通过组件名和类名选择组件，但对于自定义组件内部的结构，就不能直接这样拿到了。

   课程详情页的底部按钮其实是一个自定义组件，并且还嵌套了子自定义组件，我们看一下底部按钮的wxml结构:

   ![企业微信截图_15617087319540](/images/mini-program-auto-test/企业微信截图_15617087319540.png)

   红色框框就是想要获取的目标，尝试一下直接通过 `page.$('.bottom-btn')` 或 `page.$('.buy')` 返回的都是undefined，那怎么获取呢？我们先来看看botton-button内部是什么样子的。

   ```javascript
     const basicApplyButton = await page.$('bottom-button');
     console.log(await basicApplyButton.wxml());
   ```

   获取bottom-button并打印它的wxml字符串看一下：

   ```html
   "<view class="bottom-button--bottom-button-space" wx:nodeid="17"><view class="bottom-button--bottom-button-wrapper" wx:nodeid="261"><basic is="components/discount-button/components/basic/basic" wx:nodeid="262"><view wx:nodeid="263"><view class="basic--bottom-button-container" wx:nodeid="264"><view class="basic--bottom-btn basic--buy" wx:nodeid="265">      立即购买    </view></view></view></basic></view></view>"
   ```

   发现了什么！小程序实际运行时，自定义组件内部的类名都加上了组件名前缀，再试试 `page.$('.basic--buy')` 发现果然成功获取到了，所以虽然表面上miniprogram-automator只能操作和获取page中的内容，但自定义组件内部的结构实际上也是以某种方式存在于page中的。

   接下来看一下跳转，可以直接获取到对应组件后调用`.tap()`方法来模拟点击，这里需要注意的是，由于微信小程序开发者工具中点击打开新页面耗时较长，需要等待页面加载一会，不然接下来获取当前页面路径的时候页面还没跳转过去就拿不到不到新页面路径了。等待的时长可以根据经验给个稍大的比较安全的值。

2. **点击试学按钮可正常播放试学视频**

   ```javascript
   const player_video = await tapTcplayer(page, '.player-task');
   expect(await player_video.wxml()).toContain('video-current-time'); // 试学
   ```

   由于微信开发者工具的限制，云点播会降级为tcplayer播放，tcplayer内部的核心组件其实是 `<video>` 组件，wxml结构如下：

   ![img](/images/mini-program-auto-test/企业微信截图_15617118027035.png)

   如何判断视频是否成功播放呢？

   我们先按照上面的方法获取播放成功的video组件的wxml字符串看看

   ```html
   "<video class="component-video-video--player_video" controls="" danmu-list="[]" initial-time="0" object-fit="contain" poster="https://10.url.cn/qqc..." src="http://113.96.98.148/vedu.tc.qq.com/AtmkzyWCuq..." autoplay="" wx:nodeid="446"><div class="video-container" wx:nodeid="447"><div class="video-bar full" style="opacity: 1;" wx:nodeid="457"><div class="video-controls" wx:nodeid="458"><div class="video-control-button pause" wx:nodeid="459"><div parse-text-content="" class="video-current-time" wx:nodeid="460">00:02<div class="video-progress-container" wx:nodeid="462"><div class="video-progress" wx:nodeid="463"><div style="left: -21px;" class="video-ball" wx:nodeid="464"><div class="video-inner" wx:nodeid="465"><div parse-text-content="" class="video-duration" wx:nodeid="466">06:09<div class="video-fullscreen" wx:nodeid="468"><div style="z-index: -9999" class="video-danmu" wx:nodeid="453"></video>"
   ```

   惊了！原生 `<video>` 组件内部竟然是 `<div>` ，我们还可以注意到一个关键的class: video-current-time 内部数值为00:02，这不是当前播放进度吗？刚好可以用来判断视频有没有播放成功，就是它了！

   对比发现播放失败时根本不会出现class为video-current-time的div，所以直接用是否包含video-current-time来判断了。

3. **未购买课程时点击课程视频无法播放**

   点击非试看课程时，无法播放视频。由于不播放视频时页面中只显示cover封面图，不attatch`<video>`组件，所以直接用获取视频组件的结果进行 `toBeNull()` 判断即可。结合上面所有的代码如下：

   ```javascript
async function tapTcplayer(page, className = '.task-item') {
     const taskItem = await page.$(className);
     await taskItem.tap();
     await page.waitFor(3000);
     const playercover = await page.$('.player-cover');
     const player_video = await playercover.$('.component-video-video--player_video');
     return player_video;
   }
   it('付费课程详情页按钮显示、跳转、点播、试学功能测试', async () => {
       const page = await miniProgram.reLaunch(`/pages/course/course?cid=${commonPayCid}`);
       const basicApplyButton = await page.$('.basic--buy');
       expect(await basicApplyButton.wxml()).toContain('立即购买'); // 按钮显示
       await basicApplyButton.tap();
       await page.waitFor(1500);
       const currentPage = await miniProgram.currentPage();
       expect(currentPage.path).toContain('pages/order/order');
       await miniProgram.navigateBack();
       const player_video = await tapTcplayer(page);
       expect(player_video).toBeNull(); // 未报名不能播放视频
       const player_video_new = await tapTcplayer(page, '.player-task');
       expect(await player_video_new.wxml()).toContain('current'); // 试学
     }, 20000);
   ```
   

可以看到实际上先测试了播放课程功能，再测试了试学功能，这是为什么呢？

这是一个坑：由于播放课程失败时会有showModel弹窗提示，这个弹窗是不在wxml结构中的，无法用自动化控制工具点击关闭，实际测试中这个弹窗会阻塞下一个测试项的第一步：页面跳转，导致下一个测试项直接打不开页面导致失败，只能等待一段时间再跳转，所以直接把弹窗放在测试试学功能之前，就不会影响下一个测试项了。

还有一个需要注意的地方，在项目中，点击播放后5秒不触发进度刷新的方法就会上报视频播放失败，实际测试发现一般3秒即可正常播放，所以只等待3秒，3秒后未成功播放的视为播放失败。

最后，jest默认一个测试项的时长不能大于5秒，这项测试既有页面跳转又有视频播放，明显会超出5秒的限制，实际耗时约为15秒左右，所以修改时长限制为20000毫秒。

运行测试脚本结果如下：

![img](/images/mini-program-auto-test/企业微信截图_15617135006069.png)

目前实现的测试功能如下：

- nohost检测
- 首页数据拉取、显示、跳转测试
- 付费课程详情页按钮显示、跳转、点播、试学功能测试
- 优惠券按钮显示、领取功能测试 
- 限时优惠按钮显示测试
- 免费课程详情页按钮显示、报名、点播功能测试 
- 分类页展示、跳转列表页、跳转详情页测试

Checklist中功能测试的完成情况如下：完成度为65%

| review点                   | 自动化测试 | 备注                     |
| ----------------------- | ---------- | ------------------------ |
| 是否去除nohost插件                                | 支持       |                          |
| 首页是否正常显示                                  | 支持       |                          |
| pc首页小程序登陆是否正常                          | 暂不       | 信息授权无法自动完成     |
| 安卓支付能力是否正常                              | 暂不       | webview内部无法获取信息  |
| 分类页是否正常显示                                | 支持       |                          |
| 是否可以正常登陆                                  | 暂不       | 信息授权无法自动完成     |
| 课程表是否正常展示，学习进度/直播状态是否正常显示 | 支持       | 待完善                   |
| 课程详情页是否可以正常展示                        | 支持       |                          |
| 扫码/分享是否正常唤起小程序                       | 暂不       | 开发者工具不支持 |
| 付费课直播是否可以正常播放（上云跟腾讯视频）      | 暂不       | 开发者工具不支持直播     |
| 免费课直播是否可以正常播放（上云跟腾讯视频）      | 暂不       | 开发者工具不支持直播     |
| 免费课录播是否可以正常播放（上云跟腾讯视频）      | 部分支持   | 开发者工具降级到tcplayer |
| 付费课录播是否可以正常播放（上云跟腾讯视频）      | 部分支持   | 开发者工具降级到tcplayer |
| 试学任务是否可以正常播放                          | 支持       |                          |
| 详情页视频是否正常播放                            | 支持       |                          |
| 营销工具相关显示是否正常                          | 支持       |                          |
| 是否能正常完成支付逻辑                            | 暂不       | webview内部无法获取信息  |
| 类目筛选是否正常                                  | 支持       | 待完善                   |
| 是否可以正常搜索且列表显示正常                    | 支持       | 待完善                   |
| 本地加载耗时是否保持1s内                          | 支持       |                          |



## 四、缘续-遇到的问题与功能限制

1. 获取页面中的组件只能采用 `page.$()` 或 `page.$$()` 方法，经尝试选择器仅支持组件名和类名。无法直接获取自定义组件内部组件元素，需要在类名前增加前缀。实际项目的页面中大量使用自定义组件，对于自定义组件内部的结构判断非常不方便，只能通过`wxml()`方法将自定义组件内部结构打印出来才能确认内部的子组件的实际情况。且无法调用自定义组件内部的方法。

2. Jest的snapshot功能对于结构相对固定的组件或页面是一种非常好的测试方式，但用起来有坑。在小程序中snapshot的对照内容通常是通过组件的wxml方法打印的字符串，但实际在运行时，wxml方法返回结果可能会不同，组件可能会被自动添加上wx:node-id属性，但有时返回字符串中又不添加，会导致snapshot测试不通过。

3. 目前只能在开发者工具环境下测试，导致直播功能无法测试且云点播会自动降级为腾讯视频点播，直播也无法测试。

4. 登陆、扫码等功能无法测试，因为自动化控制工具无法扫描和点击授权弹窗。

5. `<web-view>` 组件获取不到任何内部信息，也无法自动化控制。

希望这些问题后续能够得到解决~~
