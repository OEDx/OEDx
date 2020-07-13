---
title: Cocos Creator 常用扩展分享
date: 2020-06-30 16:31:28
tags: [Cocos, Cocos Creator, 插件]
categories: 客户端
author: "[付多兴(nextfu)](https://www.github.com/potato47)"
---

在使用 Cocos Creator 开发项目的过程中，为了提高开发效率我们开发了很多扩展插件，本文介绍常用的几款扩展，抛砖引玉，希望给大家带来帮助。

## 网页扩展：运行时查看场景节点树

Cocos Creator 本地项目通常会在 Chrome 上调试运行，借助 Chrome 强大的开发者工具，我们可以对网页的性能、网络、脚本逻辑等进行调试优化。然而对于游戏来说，场景中的节点组件信息并没有办法直观的获取和修改，无法快速定位问题。为了解决这个痛点，我们可以修改 Cocos Creator 预览时的网页模版，让其显示更多的场景信息。

下面是修改过后的预览游戏界面

![ccc-devtools 效果预览](/images/cocos-creator-extensions/ccc-devtools-preview1.gif)

扩展了如下功能：
- 场景节点树实时显示，节点、组件属性实时显示更改
- 可视化缓存资源
- 标记场景中节点位置
- 输出节点、组件引用到控制台

源码：

[https://github.com/potato47/ccc-devtools](https://github.com/potato47/ccc-devtools)

此页面使用了 [vue](https://github.com/vuejs/vue) + [vuetify](https://github.com/vuetifyjs/vuetify) 开发，对于单页面应用来说 vue 是非常好的选择，大家也可以基于这种方式来定制自己项目的预览界面。

## VS Code 扩展：JavaScript 代码支持函数跳转

Cocos Creator 支持 JavaScript 和 TypeScript 两种语言，如果你是用 VS Code 来开发 Cocos Creator 的 js 项目，那你的编程体验应该不是很好，因为 Cocos Creator 的组件脚本是一套自定义的结构，

```
const mylog = require('mylog');

cc.Class({
  properties: {
    node1: cc.Node,
    node2: cc.Node,
    label1: cc.Label,
    label2: cc.Label,
  },
  start: function() {
    this.method1();
  },
  method1: function() {
    console.log('method1');
  },
  method2: function() {
    console.log('method2');
  },
});
```

在这个结构下，VS Code 不能识别 `this`，当你在 start 方法里输入 `this.` 的时候无法准确的获得可以访问的属性和方法，也无法通过点击方法名或者模块名来跳转到定义位置。

好在 VS Code 有丰富的扩展 API，通过学习[文档](https://code.visualstudio.com/api/language-extensions/programmatic-language-features)，我们开发出了一款让 js 代码支持函数跳转，属性提示的插件。 大家可以在 VS Code 插件商店里搜索 “Cocos Creator JS“ 来下载使用。

![插件商店下载](/images/cocos-creator-extensions/vscode-cc-js-preview.png)

下面是预览效果

![js代码支持函数跳转](/images/cocos-creator-extensions/vscode-cc-js-preview1.gif)

![js代码提示](/images/cocos-creator-extensions/vscode-cc-js-preview2.gif)

![模块跳转](/images/cocos-creator-extensions/vscode-cc-js-preview3.gif?40)

源码：

[https://github.com/potato47/vscode-cocos-creator-js](https://github.com/potato47/vscode-cocos-creator-js)

## 编辑器扩展：微信小游戏子包依赖检查

得益于 Cocos Creator 优秀的跨平台能力，我们的项目上线了 Android、iOS、Web 和微信小游戏平台。由于微信平台对代码包大小有限制，在上线微信小游戏时我们使用了代码分包功能，但是项目开发过程中有些模块互相耦合，导致分包后主包与子包或者子包之间有依赖，在被依赖包加载前就对其进行导入会导致程序出错。为了解决这个问题我们开发了一款 Creator 插件，可以对子包依赖自动检测。效果如下:

![子包依赖检查插件预览](/images/cocos-creator-extensions/wx-subpackage-helper-preview1.png)

点击检查依赖按钮后，插件会自动检查所有子包间的依赖，并可视化的显示出来，点击文件名定位到脚本位置，根据修改建议修改即可。

源码：

[https://github.com/potato47/wx-subpackage-helper](https://github.com/potato47/wx-subpackage-helper)

## 编辑器扩展：快速打开场景

在使用 Cocos Creator 编辑器过程中个人体验最不好的就是资源管理器里的搜索功能了，在 1.x 版本搜索资源时每输入一个字母都会调用一次全局过滤资源的函数，输入过程中就会感觉到持续的卡顿，在 2.x 版本稍微优化了一下体验，改为敲回车再执行过滤函数，但在资源繁多的项目里搜索依然会卡顿。为了解决这个不大但影响心情的问题，我们开发了一款搜索插件，可以快速搜索打开场景或者预制体资源。无需等待，无需鼠标。

![quick-open插件预览](/images/cocos-creator-extensions/quick-open-preview1.gif)

源码：

[https://github.com/potato47/cocos-creator-quick-open-x](https://github.com/potato47/cocos-creator-quick-open-x)

## 控制台扩展：控制台查看节点树，节点属性

通过 Chrome 的开发者工具我们可以直接对原生平台中 Cocos Creator 的 JavaScript 代码进行远程调试，但一些 UI 相关问题依然不好定位，如果能在控制台里查看节点树就会方便很多。这里就给大家分享一个控制台查看节点树节点属性的脚本。

```
(function () {
  if (cc.tree) return;
  cc.tree = function (key) {
    let index = key || 0;
    let treeNode = function (node) {
      let nameStyle =
        `color: ${node.parent === null ||
          node.activeInHierarchy ? 'green' : 'grey'}; 
        font-size: 14px;font-weight:bold`;
      let propStyle =
        `color: black; background: lightgrey;margin-left: 5px;
        border-radius:3px;padding: 0 3px;font-size: 10px;
        font-weight:bold`;
      let indexStyle =
        `color: orange; background: black;margin-left: 5px;border-radius:3px;
        padding:0 3px;fonrt-size: 10px;font-weight:bold;`
      let nameValue = `%c${node.name}`;
      let propValue =
        `%c${node.x.toFixed(0) + ',' + node.y.toFixed(0) + ',' +
        node.width.toFixed(0) + ',' + node.height.toFixed(0) + ',' +
        node.scale.toFixed(1)}`;
      let indexValue = `%c${index++}`;
      if (node.childrenCount > 0) {
        console.groupCollapsed(nameValue + propValue + indexValue, nameStyle,
          propStyle, indexStyle);
        for (let i = 0; i < node.childrenCount; i++) {
          treeNode(node.children[i]);
        }
        console.groupEnd();
      } else {
        console.log(nameValue + propValue + indexValue, nameStyle, propStyle,
          indexStyle);
      }
    }
    if (key) {
      let node = cc.cat(key);
      index = node['tempIndex'];
      treeNode(node);
    } else {
      let scene = cc.director.getScene();
      treeNode(scene);
    }
    return '属性依次为x,y,width,height,scale.使用cc.cat(id)查看详细属性.';
  }
  cc.cat = function (key) {
    let index = 0;
    let target;
    let sortId = function (node) {
      if (target) return;
      if (cc.js.isNumber(key)) {
        if (key === index++) {
          target = node;
          return;
        }
      } else {
        if (key.toLowerCase() === node.name.toLowerCase()) {
          target = node;
          return;
        } else {
          index++;
        }
      }
      if (node.childrenCount > 0) {
        for (let i = 0; i < node.childrenCount; i++) {
          sortId(node.children[i]);
        }
      }
    }
    let scene = cc.director.getScene();
    sortId(scene);
    target['tempIndex'] = cc.js.isNumber(key) ? key : index;
    return target;
  }
  cc.list = function (key) {
    let targets = [];
    let step = function (node) {
      if (node.name.toLowerCase().indexOf(key.toLowerCase()) > -1) {
        targets.push(node);
      }
      if (node.childrenCount > 0) {
        for (let i = 0; i < node.childrenCount; i++) {
          step(node.children[i]);
        }
      }
    }
    let scene = cc.director.getScene();
    step(scene);
    if (targets.length === 1) {
      return targets[0];
    } else {
      return targets;
    }
  }
  cc.where = function (key) {
    let target = key.name ? key : cc.cat(key);
    if (!target) {
      return null;
    }
    let rect = target.getBoundingBoxToWorld();
    let bgNode = new cc.Node();
    let graphics = bgNode.addComponent(cc.Graphics);
    let scene = cc.director.getScene();
    scene.addChild(bgNode);
    bgNode.position = rect.center;
    bgNode.group = target.group;
    bgNode.zIndex = cc.macro.MAX_ZINDEX;
    let isZeroSize = rect.width === 0 || rect.height === 0;
    if (isZeroSize) {
      graphics.circle(0, 0, 100);
      graphics.fillColor = cc.Color.GREEN;
      graphics.fill();
    } else {
      bgNode.width = rect.width;
      bgNode.height = rect.height;
      graphics.rect(-bgNode.width / 2, -bgNode.height / 2,
        bgNode.width, bgNode.height);
      graphics.fillColor = new cc.Color().fromHEX('#E91E6390');
      graphics.fill();
    }
    setTimeout(() => {
      if (cc.isValid(bgNode)) {
        bgNode.destroy();
      }
    }, 2000);
    return target;
  }
})();
```
将上述代码粘贴到控制台，然后输入 `cc.tree()`, 神奇的事情就发生了

![控制台查看节点树](/images/cocos-creator-extensions/cc-console-utils.png)

每个节点后边会附带常见的几个属性和唯一id，通过 `cc.cat(id)` 即可获得这个节点的引用，提高调试效率。

## 参考

- [Cocos Creator 扩展编辑器文档](https://docs.cocos.com/creator/manual/zh/extension/)
- [Cocos Creator 自定义网页预览文档](https://docs.cocos.com/creator/manual/zh/advanced-topics/custom-preview-template.html)
- [VS Code 插件开发文档](https://code.visualstudio.com/api/get-started/your-first-extension)
