---
title: CocosCreator 引擎资源加载与释放原理简析
date: 2019-03-14 17:01:35
tags: Cocos
categories: 客户端
author: "[张韩(khanzhang)](http://khanzhang.cn)"
---

![图片取自 Zoommy](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/1552558246.59SmartPic.png)

本文主要内容如下：

1. 资源加载与释放部分代码所在
2. 调试、修改引擎源码的方法
3. 资源加载与释放原理简析

如要了解 CocosCreator 引擎资源加载与释放的原理，调试、修改引擎代码有助于对其原理进行理解。因此文中会先介绍 CocosCreator 引擎各部分及其文件夹，然后介绍调试、修改引擎源码的方法，最后对其原理进行分析。

如果你对阅读、调试修改源码不感兴趣，可直接跳转到第三部分阅读。如果你对此主题不感兴趣，可直接关闭网页。

> 本文基于 CocosCreator 1.10.2，另外 2.0.8 版本关于资源加载释放部分改动不大，也可适用

## 前言

Cocos Creator 的引擎部分包括 ```JavaScript```、```Cocos2d-x-lite``` 和 ```adapter``` 三个部分，各部分对应源码在（Mac 版）：

1. JavaScript：CocosCreator.app/Resource/engine（JS 引擎）
2. Cocos2d-x-lite：CocosCreator.app/Resource/cocos2d-x（Cococ2d-x 引擎）
3. adapter：CocosCreator.app/Resource/builtin/

其中 ```engine``` 文件夹下代码部分包含了引擎的 JS 层逻辑，而引擎的资源加载与释放部分代码就处于此文件夹中，路径为 ```cocos2d/core/load-pipeline/```，主要涉及 ```pipeline.js```、```loading-items.js```、```CCLoader.js```、```loader.js```、```uuid-loader.js``` 等文件。

另外，需要了解的是 CocosCreator 引擎中资源是有依赖关系的，比如 SpriteAtlas 资源的加载会依赖于多个 SpriteFrame 资源的加载，而 SpriteFrame 资源依赖于 Texture2D 资源。

下边对调试、修改引擎代码做简单介绍。

## 调试、修改引擎代码

1. 找到 ```CocosCreator``` 的 ```JavaScript``` 引擎所在目录 ```CocosCreator.app/Resource/engine```，将该文件夹复制到其他地方，我们将对复制后的代码进行调试和修改。然后在 CocosCreator 的项目设置中修改 ```JavaScript``` 引擎路径为复制后的路径。如果要调试 ```Cocos2d-x``` 部分，修改对应文件夹即可。如下图：
![定义 JavaScript 引擎路径](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/1552547553.94SmartPic.png)

2. 运行以下命令安装编译依赖

    ```shell
    # 在命令行中进入引擎路径
    cd [engine_path]/engine
    # 安装 gulp 构建工具
    npm install -g gulp
    # 安装依赖的模块
    npm install
    ```

3. 现在可以打开 ```engine``` 文件夹，对 JS 引擎部分进行修改。修改后，在该文件夹下运行 ```gulp build``` 命令即可编译修改的部分，然后刷新 CocosCreator 预览的网页即可

4. 至于调试其源码，可以直接在 ```Chrome``` 开发者工具中 ```Cmd + o``` （Mac 快捷键）呼出搜索框，输入并打开你需要调试的文件，然后即可打断点进行调试

## CocosCreator 引擎资源加载与释放简析

引擎资源加载与释放源码路径为 ```engine/cocos2d/core/load-pipeline/```，主要关注 ```pipeline.js```、```loading-items.js```、```CCLoader.js```、```loader.js```、```uuid-loader.js``` 这几个文件。

其中资源加载涉及 CCLoader、pipeline、loading-items、loader、uuid-loader 等多个类，而资源释放则主要是 CCLoader 中的 release 方法。

### 资源加载

概括来讲，CCLoader 是供上层直接使用加载、释放资源的类，整合、封装了 pipeline、loading-items、loader 等类；pipeline 包含多个 pipe，这里的 pipe 是指实现了加载资源的单位（如 loader），pipeline 对资源的处理最终是调用 pipe.handle 实现的，pipeline 自身实现的是资源缓存和让资源依次流过管道，即每个资源依次经过每个 pipe 的处理；loading-items 配合 pipeline，实现了加载状态维护、依赖资源入队等内容；asset-loader、downloader、loader 则实现了资源加载具体功能，不同的加载器负责不同资源的加载。

简化来看，一次资源的加载流程底层的调用函数如下：

1. cc.loader.loadRes()
2. CCLoader.load()
3. loading-items.append
4. pipeline.flowIn
5. pipeline.flow
6. pipe.handle（pipe 对应于 asset-loader、downloader、loader）
7. 加载完成，进行回调

加载流程中其实还有一种情况需要注意：loader 其实会根据资源类型将加载任务分发给不同的类，如 uuid 类型会交给 uuid-loader。uuid-loader 加载时会加载该资源的依赖资源，其过程为：

1. uuid-loader.loadUuid
2. uuid-loader.loadDepends
3. pipeline.flowInDeps
4. loading-items.append
5. 后续流程和正常流程一致

以下会根据加载流程分析一下是如何加载资源的。

#### CCLoader

CCLoader 继承自 pipeline，是用户可以直接调用来加载、释放资源的类，它对 pipeline、loading-items、loader 进行了整合封装，并提供了 load、loadRes、loadResDir、release、releaseRes 等方法供用户使用。资源加载相关的便是 load、loadRes 等方法，而所有方法最终是调用 load 方法来进行资源加载的。load 方法如下：

```JavaScript
proto.load = function(resources, progressCallback, completeCallback) {
    // 省略部分代码

    _sharedResources.length = 0;
    for (var i = 0; i < resources.length; ++i) {
        var resource = resources[i];

        // 省略部分代码

        var res = getResWithUrl(resource);
        if (!res.url && !res.uuid)
            continue;
        var item = this._cache[res.url];
        _sharedResources.push(item || res);
    }

    var queue = LoadingItems.create(this, progressCallback, function (errors, items) {
        callInNextTick(function () {
            // 省略部分代码
        });
    });
    LoadingItems.initQueueDeps(queue);
    queue.append(_sharedResources);
    _sharedResources.length = 0;
```

可以看到 load 方法中，所做的就是创建一个 LoadingItem，然后对其进行初始化，之后调用其 append 方法。这里 LoadingItem 简单来看就是对**待加载资源**的一层封装。下边介绍 loading-items。

#### loading-items

loading-items 主要是完善了每个资源对象的属性，包括资源内容、url 地址等，同时维护每个资源的加载状态、依赖关系等。以下是 append 方法代码。

```JavaScript
proto.append = function (urlList, owner) {
    // 省略代码

    this._appending = true;
    var accepted = [], i, url, item;
    for (i = 0; i < urlList.length; ++i) {
        // 省略代码，主要是对每个资源的依赖关系、循环依赖、是否完成等进行处理

        // Queue new items
        if (isIdValid(url)) {
            item = createItem(url, this._id);
            var key = item.id;
            // No duplicated url
            if (!this.map[key]) {
                this.map[key] = item;
                this.totalCount++;
                // Register item deps for circle reference check
                owner && owner.deps.push(item);
                LoadingItems.registerQueueDep(owner || this._id, key);
                accepted.push(item);
                // console.log('+++++ Appended ' + item.id);
            }
        }
    }
    this._appending = false;

    // Manually complete
    if (this.completedCount === this.totalCount) {
        this.allComplete();
    }
    else {
        this._pipeline.flowIn(accepted);
    }
    return accepted;
};
```

从代码可以看出，append 所做的是：先对资源列表 urlList 里每一个元素做字段完善（create）、依赖处理（registerQueueDep），然后调用 pipeline.flowIn 将所有未加载完成的资源放入 pipeline 之中。下边介绍 pipeline。

#### pipeline

pipeline 包含多个 pipe，pipe 存储在 _pipes 数组中，而且数组中的 nextPipe 赋值给了 lastPipe.next，以此将 pipe 链接成单向链表结构。这里的 pipe 是指实现了加载资源的单位（如 loader），pipeline 对资源的处理最终是调用 pipe.handle 实现的，pipeline 自身实现的是资源缓存和让资源依次流过管道，即每个资源依次经过每个 pipe 的处理。在 CCLoader 初始化 pipeline 时，为它添加了 AssetLoader、Downloader、Loader 三个 pipe，每个资源都会依次经过这三个 pipe 的处理。pipeline 中的 _cache 数组是对每个资源的缓存。

以下是其 flowIn 方法：

```JavaScript
proto.flowIn = function (items) {
    var i, pipe = this._pipes[0], item;
    if (pipe) {
        // Cache all items first, in case synchronous loading flow same item repeatly
        for (i = 0; i < items.length; i++) {
            item = items[i];
            this._cache[item.id] = item;
        }
        for (i = 0; i < items.length; i++) {
            item = items[i];
            flow(pipe, item);
        }
    }
    else {
        for (i = 0; i < items.length; i++) {
            this.flowOut(items[i]);
        }
    }
};
```

flowIn 方法所做其实就是：先将每个资源 item 缓存到 _cache 中，然后对每个 item 调用 flow 方法。这里要注意的是 flow 的 pipe 参数为 _pipes[0]，即先将资源交给 pipeline 的第一个 pipe。以下是 flow 方法：

```JavaScript
function flow (pipe, item) {
    var pipeId = pipe.id;
    var itemState = item.states[pipeId];
    var next = pipe.next;
    var pipeline = pipe.pipeline;

    if (item.error || itemState === ItemState.WORKING || itemState === ItemState.ERROR) {
        return;
    }
    else if (itemState === ItemState.COMPLETE) {
        if (next) {
            flow(next, item);
        }
        else {
            pipeline.flowOut(item);
        }
    }
    else {
        item.states[pipeId] = ItemState.WORKING;
        // Pass async callback in case it's a async call
        var result = pipe.handle(item, function (err, result) {
            // 省略代码
        });
        // If result exists (not undefined, null is ok), then we go with sync call flow
        if (result instanceof Error) {
            item.error = result;
            item.states[pipeId] = ItemState.ERROR;
            pipeline.flowOut(item);
        }
        else if (result !== undefined) {
            // Result can be null, then it means no result for this pipe
            if (result !== null) {
                item.content = result;
            }
            item.states[pipeId] = ItemState.COMPLETE;
            if (next) {
                flow(next, item);
            }
            else {
                pipeline.flowOut(item);
            }
        }
    }
}
```

flow 方法中对资源的处理分为三部分：

- 如果资源状态为 ERROR，则直接返回
- 如果资源状态为 COMPLETE 已完成，如果后续还有 pipe 就将其交给下一个 pipe 处理
- 如果资源状态为空，说明此资源从未进行加载，则调用 pipe.handle 方法对其进行加载。加载完成后，先将加载结果交给 item，即```item.content = result```，然后会对其加载状态 states 进行修改，最后如果后续还有 pipe 就将其交给后续 pipe 处理。

接下来再来看一下 pipe.handle 方法。从以下代码可以看出， CCLoader 初始化时，提供了三个 pipe：asset-loader、downloader、loader。实际上 pipe.handle 调用的其实是 assetLoader、downloader、loader 的 handle 方法，下边以 loader 为例进行介绍。

```JavaScript
function CCLoader () {
    var assetLoader = new AssetLoader();
    var downloader = new Downloader();
    var loader = new Loader();

    Pipeline.call(this, [
        assetLoader,
        downloader,
        loader
    ]);
    // 省略代码
}
```

#### loader

loader 是 CCLoader 中添加到 pipeline 的 pipe 之一，其作用就是加载资源，pipeline 会直接调用其 handle 方法来加载资源。类似的类有 asset-loader、downloader。而 loader 相对特殊是它其中又会根据资源类型不同，调用不同的类去实现加载功能。比如 uuid 类型会调用 uuid-loader 来加载。这里提到的 uuid-loader 在加载资源的时候，会去判断资源是否有依赖资源，如果有会将依赖资源添加到 pipeline 中，进而对依赖资源进行加载。

loader 的部分代码如下：

```JavaScript
var defaultMap = {
    // Images
    'png' : loadImage,
    'jpg' : loadImage,
    'bmp' : loadImage,

    // 省略代码

    'uuid' : loadUuid,
    'prefab' : loadUuid,
    'fire' : loadUuid,
    'scene' : loadUuid,

    'default' : loadNothing
};

// 省略代码

Loader.prototype.handle = function (item, callback) {
    var loadFunc = this.extMap[item.type] || this.extMap['default'];
    return loadFunc.call(this, item, callback);
};
```

可以看出 loader 会根据资源类型不同，进而调用对应的类去实现加载功能。如 uuid 类型资源，调用的是 uuid-loader 的 loadUuid 方法。而至于 loadUuid 方法中，则做了 deserialize 资源、解析处理依赖资源等。

### 资源释放

CCLoader 对于资源释放提供了 release、releaseRes、releaseAsset、releaseResDir、releaseAll 方法，但实际上所有方法最终是调用 release 方法来实现资源释放的，在 release 调用之前所做的就是获取资源对应的 uuid，然后调用 release 方法。以下是 release 方法：

```JavaScript
proto.release = function (asset) {

    if (Array.isArray(asset)) {
        for (let i = 0; i < asset.length; i++) {
            var key = asset[i];
            this.release(key);
        }
    }
    else if (asset) {
        var id = this._getReferenceKey(asset);
        var item = this.getItem(id);
        if (item) {
            var removed = this.removeItem(id);
            asset = item.content;
            if (cc.Class.isInstanceOf(asset, cc.Asset)) {
                if (asset.nativeUrl) {
                    this.release(asset.nativeUrl);  // uncache loading item of native asset
                }
                asset.destroy();
            }
            if (CC_DEBUG && removed) {
                this._releasedAssetChecker_DEBUG.setReleased(item, id);
            }
        }
    }
};
```

从代码中可以看到，release 分为两部分：

1. 如果资源是数组，则对数组中的每个元素调用 release
2. 第二部分是释放资源的关键，所做的工作可以简化如下：
    1. 获取资源在 pipeline._cache 中的缓存 item，并在 _cache 中移除该 item
    2. 拿到资源的实际内容，即 item.content，这也是 loader 加载出来的资源结果。如果 content 是 Asset 类型，则调用其 destroy 方法进行释放，同时释放其依赖的 nativeUrl 资源
    3. 如果是 debug 模式，则会在 released-asset-checker 中标记该资源为已释放

> 注：
>
> 1. 这里的 released-asset-checker 是一个辅助类，其中一个作用就是检测已释放资源是否仍被其他资源引用
> 2. release 方法其实仍存在缺陷，缺陷之一是无法释放 SpriteAtlas 所依赖的 SpriteFrame 资源，在释放后 SpriteFrame 资源仍存在与 pipeline._cache 中。这导致释放后再次加载同一个 SpriteAtlas 资源的加载不完整问题，该问题的表现是第二次加载 SpriteAtlas 后播放帧动画黑块现象。原因是：加载到 SpriteAtlas 依赖的 SpriteFrame 资源部分时，根据 pipeline.flow 方法代码会发现由于此时 ```SpriteFrame.states = { "AssetLoader": 2, "Downloader": 2, "loader": 2}```，这里 2 代表加载状态 COMPLETE，就是说该资源在所有 pipe 的状态均是已加载完成，资源直接通过了三个管道，最后执行 flowOut 方法，没有继续加载 SpriteFrame 所依赖的 Texture2D 资源，导致显示黑块。

## 参考文章

- [引擎定制工作流程](https://docs.cocos.com/creator/manual/zh/advanced-topics/engine-customization.html)
- [CocosCreator API 文档](https://docs.cocos.com/creator/1.10/api/zh/)
- 本文图片使用 Alfred Workflow：SmartPic 一键上传获取链接，[点击查看 SmartPic](http://www.packal.org/workflow/smartpic)