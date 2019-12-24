---
title: Cocos Creator 最佳实践：JavaScript兼容性问题规避
date: 2019-12-24 18:01:20
tags: [Cocos Creator]
categories: 客户端
author: "[潘伟洲(josephpan)](http://hahack.com)"
---

本文从 Cocos Creator 开发的角度出发，仔细探讨了关注 JavaScript API 兼容性的必要性，以及如何借助工具和 Polyfill 来规避 Cocos Creator 项目的兼容性问题。

## 一、引言：JavaScript虚拟机的差异性

不同的浏览器和移动设备所使用的 JavaScript 虚拟机（VM）千差万别，所支持的 API 也大相径庭。

我们来了解一下 Cocos Creator 在各个端所使用的 JavaScript VM ：

- 对于 iOS 客户端和 Mac 客户端：在 Cocos Creator 1.6 及以前，Cocos Creator 一直是使用非系统原生的 SpiderMonkey 作为 JS VM ；从 1.7 开始，Cocos Creator 引入了 JSB 2.0 ，支持了 V8、JavaScriptCore 等多种 JS VM 。于是 Cocos Creator 便将 iOS 端和 Mac 端的 JS VM 都改为了系统自带的 JavaScriptCore ，以达到节省包体的目的；到了 2.1.3 ，Cocos Creator 又将 Mac 端的 JS VM 切换到了 V8，以提升应用性能。
- 对于 Android 客户端和 Windows 客户端：在 Cocos Creator 1.6 及以前，Cocos Creator 同样是使用 SpiderMonkey 作为 JS VM ；从 1.7 开始，得益于 JSB 2.0 ，V8 成了 Android 和 Windows 客户端的 JS VM 。
- 对于 Web 端：使用浏览器本身的 JavaScript VM 来解析 JavaScript 代码。

![JSB 2.0 架构图](/images/cocos-creator-api-compat/api-compat-jsb2-arch.png)

从上可见，由于 JS VM 不同，同一份代码在不同的平台上运行可能会有很大差异。为了让我们的产品能够给尽可能多的用户使用，我们在开发阶段就需要时刻注意 JavaScript 的 API 兼容性。

举个例子：`fetch()` 方法是一个用来取代 `XMLHTTPRequest` 的 API。相比后者，它的优点在于可读性更高，且可以很方便地使用 Promise 写出更优雅的代码。

``` javascript
fetch(
    'http://domain/service',
    { method: 'GET' }
  )
  .then( response => response.json() )
  .then( json => console.log(json) )
  .catch( error => console.error('error:', error) );
```

然而，`fetch()` 方法不支持所有的 IE 浏览器，也无法在 2017 年以前的 Chrome、Firefox 和 Safari 版本上运行。当你的用户有很大一部分是上述的用户时，你就需要考虑禁止使用 `fetch()` API ，而重新回到 `XMLHTTPRequest` 的怀抱。

在开发阶段，人工保证 API 的兼容性是不可靠的。更可靠的方式是借助工具来自动化扫描。例如下面要介绍的 eslint-plugin-compat 。

## 二、使用 eslint-plugin-compat

eslint-plugin-compat 是 ESLint 的一个插件，由前 uber 工程师 Amila Welihinda 开发。它可以帮助发现代码中的不兼容 API 。

![使用 eslint-plugin-compat 扫描不兼容 API](/images/cocos-creator-api-compat/api-compat-1.png)

下面介绍如何在工程中接入 eslint-plugin-compat 。

### 2.1 安装 eslint-plugin-compat

安装 eslint-plugin-compat 和安装其他 ESLint 插件类似：

``` bash
$ npm install eslint-plugin-compat --save-dev
```

还可以顺便把依赖的 browserslist 和 caniuse-lite 一起安装了：

``` bash
$ npm install browserslist caniuse-lite --save-dev
```

### 2.2 修改 ESLint 配置

之后，我们需要修改 ESLint 的配置，加上该插件的使用：

``` json
// .eslintrc.json
{
  "extends": "eslint:recommended",
  "plugins": [
    "compat"
  ],
  "rules": {
    //...
    "compat/compat": 2
  },
  "env": {
    "browser": true
    // ...
  },
  
  // ...
}
```

### 2.3 配置目标运行环境

通过在 package.json 中增加 `browserslist` 字段来配置目标运行环境。示例：

``` json
{
  // ...
  "browserslist": ["chrome 70", "last 1 versions", "not ie <= 8"]
}
```

上面的值表示 Chrome 版本 70 以上，或每种浏览器的最近一个版本，或者非 ie 8 及以下。这里的填写格式是遵循 browserslist （https://github.com/browserslist/browserslist ）所定义的一套描述规范。browserslist 是一套描述产品目标运行环境的工具，它被广泛用在各种涉及浏览器/移动端的兼容性支持工具中，例如 eslint-plugin-compat 、babel、Autoprefixer 等。下面我们来详细了解一下 browserslist 的描述规范。

browserslist 支持指定目标浏览器类型，并且能够灵活组合多种指定条件。

#### 指定目标浏览器类型 ####

browserslist 收录了如下一些浏览器，可以在条件中使用（注意大小写敏感）：

- `Android`：用于 Android WebView。
- `Baidu`：用于百度浏览器。
- `BlackBerry` 或 `bb`：用于黑莓浏览器。
- `Chrome`：用于 Google Chrome。
- `ChromeAndroid` 或 `and_chr`：用于 Android Chrome。
- `Edge`：用于 Microsoft Edge。
- `Electron`：用于 Electron framework。 将会被转换成 Chrome 版本。
- `Explorer` 或 `ie`：用于 Internet Explorer。
- `ExplorerMobile` 或 `ie_mob`：用于 Internet Explorer Mobile。
- `Firefox` 或 `ff`：用于 Mozilla Firefox。
- `FirefoxAndroid` 或 `and_ff`：用于 Android Firefox。
- `iOS` 或 `ios_saf`：用于 iOS Safari。
- `Node`：用于 Node.js。
- `Opera`：用于 Opera。
- `OperaMini` 或 `op_mini`：用于 Opera Mini。
- `OperaMobile` 或 `op_mob`：用于 Opera Mobile。
- `QQAndroid` 或 `and_qq`：用于 Android QQ 浏览器。
- `Safari`：用于 desktop Safari。
- `Samsung`：用于 Samsung Internet。
- `UCAndroid` 或 `and_uc`：用于 Android UC 浏览器。
- `kaios`：用于 KaiOS 浏览器。

#### browseslist 的条件语法 ####

browserslist 支持非常灵活的条件语法，下面给出一些例子作为参考（注意大小写敏感），供读者们举一反三。

* `> 5%`：表示要兼容全球用户统计比例 > 5% 的浏览器版本。`>=`、`<` 及 `<=` 也都是可用的。
* `> 5% in US`：表示要兼容美国用户统计比例 > 5% 的浏览器版本。这里的 `US` 是美国的 Alpha-2 编码 [^1]。也可以换成其他国家/地区的 Alpha-2 编码。例如，中国就是 `CN` 。
* `> 5% in alt-AS`：表示要兼容亚洲用户统计比例 > 5% 的浏览器版本。这里的 `alt-AS` 表示亚洲地区[^1]。
* `> 5% in my stats`：表示要兼容自定义的用户统计比例 > 5% 的浏览器版本。
* `cover 99.5%`：表示要兼容用户份额累计前 99.5% 的浏览器版本。
* `cover 99.5% in US`：同上，但通过 Alpha-2 编码来加上国家/地区的限定。
* `cover 99.5% in my stats`：使用用户的数据。
* `maintained node versions`：所有官方还在维护的 Node.js 版本。
* `current node`：Browserslist 现在正在使用的 Node.js 版本。
* `extends browserslist-config-mycompany`：表示要兼容 browserslist-config-mycompany 这个 npm 包的查询结果。
* `ie 6-8`：表示要兼容 IE 6 ~ IE 8 的版本（即 IE 6、IE 7 和 IE 8）。
* `Firefox > 20`：表示要兼容 > 20 的 Firefox 版本。`>=`、`<` 及 `<=` 也都是可用的。
* `iOS 7`：表示要兼容 iOS 7 。
* `Firefox ESR`：表示要兼容最新的 Firefox ESR 版本。
* `PhantomJS 2.1 and PhantomJS 1.9`：表示要兼容 PhantomJS 2.1 和 1.9 版本。
* `unreleased versions` 或 `unreleased Chrome versions`：表示要兼容未发布的开发版本。后者则具体指明是要兼容未发布的 Chrome 版本。
* `last 2 major versions` 或 `last 2 iOS major versions`：表示要兼容最近两个主要版本所包含的所有小版本。后者则具体指明是要兼容 iOS 的最近两个主要版本所包含的所有小版本。
* `since 2015` 或 `last 2 years`：自 2015 年或最近两年到现在所发布的所有版本。
* `dead`：官方不再维护或者超过两年没有更新的浏览器版本。
* `last 2 versions`：每种浏览器的最近两个版本。
* `last 2 Chrome versions`：Chrome 浏览器的最近两个版本。
* `defaults`：Browserslist 的默认规则（`> 0.5%, last 2 versions, Firefox ESR, not dead`）。
* `not ie <= 8`：从前面的条件中再排除掉低于或者等于 IE 8 的浏览器。

在阅读这些规则的时候，推荐访问 http://browsersl.ist 输入相同的命令进行测试，可以直接得出符合条件的浏览器版本。

![在 browserl.ist 上测试条件](/images/cocos-creator-api-compat/api-compat-browserslist.png)

> 细心的读者可能会发现最后一条的查询结果会报错，这是因为 `not` 操作需要放在一个查询条件之后（下文会介绍）。你可以从其他规则中随意挑一条规则来组合，例如 `ie 6-10, not ie <= 8` 将会筛出 IE 9 和 IE 10 。

#### browseslist 的条件组合

browserslist 支持多种条件的组合，下面我们来了解 browseslist 的条件组合方法。

- `,` 和 `or` 都可以用来表示逻辑 “或”。例如，`last 1 version or > 1%` 与 `last 1 version, > 1%` 等价，都表示每种浏览器的最近 1 个版本，或者 > 1% 的市场份额。“或” 操作相当于集合论中的并集。
- `and` 用来表示逻辑 “与”。例如，`last 1 version and > 1%` 表示每种浏览器的最近一个版本，且 > 1% 的市场份额。“与” 操作相当于集合论中的交集。
- `not` 用来表示逻辑 “非”。例如 `> .5% and not ie <= 8` 表示 > 1% 的市场份额且排除 ie 8 及以下的版本。“非” 操作相当于集合论里头的补集，所以 `not` 不能作为第一个条件，因为你总需要知道“补”的是什么的“集”。

三种条件组合类型可以用下面的表格来示意：

| 条件组合类型 |  示意图 | 示例 |
| ---------- | ------ | ---- |
|`or`/`,` 组合 <br> （并集） | ![](/images/cocos-creator-api-compat/api-compat-union.png)  | `> .5% or last 2 versions` <br> `> .5%, last 2 versions` |
| `and` 组合 <br>（交集） | ![](/images/cocos-creator-api-compat/api-compat-intersection.png) | `> .5% and last 2 versions` |
| `not` 组合 <br>（补集） | ![](/images/cocos-creator-api-compat/api-compat-complement.png) | `> .5% and not last 2 versions` <br> `> .5% or not last 2 versions` <br> `> .5%, not last 2 versions` |

#### 配置你的 browserslist

了解了以上规则后，我们可以来配置适用于我们的工程的 browserslist 。

举个例子：假如我们的项目希望在 iOS 8 及以上，或者版本号 49 及以上且市场份额大于 0.2% 的 Chrome 桌面浏览器运行，那么可以使用如下的规则：

``` json
  // ...
  "browserslist": [
    ">.2% and chrome >= 49",
    "iOS >= 8"
  ],
```

完成后，可以使用 `npx browserslist` 来测试你配置的 browserslist 。

``` bash
$ npx browserslist
chrome 78
chrome 77
chrome 76
chrome 75
chrome 74
chrome 73
chrome 72
chrome 63
chrome 49
ios_saf 13.0-13.2
ios_saf 12.2-12.4
ios_saf 12.0-12.1
ios_saf 11.3-11.4
ios_saf 11.0-11.2
ios_saf 10.3
ios_saf 10.0-10.2
ios_saf 9.3
ios_saf 9.0-9.2
ios_saf 8.1-8.4
ios_saf 8
```

也可以访问 <https://browsersl.ist/> 上输入条件测试结果。

#### 测试效果

完成了 browserslist 规则的配置后，我们就可以结合 ESLint 扫描工程中的 API 兼容问题。同时 VS Code 插件也可以即时提示不兼容的 API 调用。

![在 VS Code 即时扫描不兼容 API](/images/cocos-creator-api-compat/api-compat-vscode.png)

## 三、使用 eslint-plugin-builtin-compat

eslint-plugin-compat 的原理是针对确认的类型和属性，使用 caniuse (http://caniuse.com) 的数据集 caniuse-db 以及 MDN（https://developer.mozilla.org/en-US/ ）的数据集 mdn-browser-compat-data 里的数据来确认 API 的兼容性。但对于不确定的实例对象，由于难以判断该实例的方法的兼容性，为了避免误报，eslint-plugin-compat 选择了跳过这类 API 的检查。

例如，`foo.includes` 在不确定 `foo` 是否为数组类型的时候，就无法判断 `includes` 方法的兼容性。在下图中，我们在使用上面的 browserslint 配置的情况下，`includes` 方法的兼容问题并没有被扫描出来：

![eslint-plugin-compat 无法检测实例对象的 API](/images/cocos-creator-api-compat/api-compat-includes-1.png)

然而，从 caniuse 上可以查知，`Array.prototype.includes()` 方法不能被 iOS 8 兼容：

![Array.prototype.includes() 并不能被 iOS 8 兼容](/images/cocos-creator-api-compat/api-compat-includes-2.png)

> 实际上，Cocos Creator 的 engine 项目自 2.1.3 版本开始，就已经针对 `Array.prototype.includes()` 方法加入了 Polyfill ，从而彻底规避了该 API 的兼容问题。在本节后面介绍 Polyfill 的时候我们将介绍如何避免该 API 的误报。

为了避免漏报这种问题，我们可以结合另一个兼容检查插件 eslint-plugin-builtin-compat 。该插件同样借助 mdn-browser-compat-data 来进行兼容扫描，与 eslint-plugin-compat 不同的是，该插件不会放过实例对象，因此它会把所有 `foo.includes` 的 `includes` 方法当成是 `Array.prototype.includes()` 方法来扫描。可想而知，这个插件可能会导致误报。因此建议将其告警级别改为 warning 级别。

### 3.1 安装 eslint-plugin-builtin-compat

``` bash
$ npm install eslint-plugin-builtin-compat --save-dev
```

### 3.2 修改 ESLint 配置

与 eslint-plugin-compat 类似，我们可以修改 ESLint 的配置，加上该插件的使用。但由于该插件容易误报，因此只建议将其告警级别改为 warning 级别：

``` json
// .eslintrc.json
{
  "extends": "eslint:recommended",
  "plugins": [
    "compat",
    "builtin-compat"
  ],
  "rules": {
    //...
    "compat/compat": 2,
    "builtin-compat/no-incompatible-builtins": 1
  },
  "env": {
    "browser": true
    // ...
  },
  
  // ...
}
```

加入该插件后，可以发现 `Array.prototype.includes()` 方法将会被该插件告警：

![eslint-plugin-builtin-compat 可以检测实例对象的 API](/images/cocos-creator-api-compat/api-compat-includes-3.png)

## 四、使用 Polyfill 解决兼容问题

靠 ESLint 在开发阶段扫描出 API 兼容问题固然是一种防治兼容性问题的手段，但如果团队里的同事并不认真注意 ESLint 的扫描结果，甚至没有将 ESLint 作为代码合入扫描的一环的话，就有可能会有漏网之鱼继续肆虐。

因此，一种更为一劳永逸的方法是为一些常用的 API 补上相应 Polyfill 。这样一方面可以为不兼容的浏览器版本添加上支持，另一方面又可以使得团队成员安心地使用新的 API ，提高开发效率。

### 4.1 Cocos Creator engine 里的 Polyfill

实际上，Cocos Creator 的 engine 项目也内置了很多常见 API 的 Polyfill ：

![cocos-creator/engine 项目内置的 Polyfill](/images/cocos-creator-api-compat/api-compat-polyfill.png)

其中就包括了 `Array.prototype.includes()` ：

![Array.prototype.includes() 的 Polyfill](/images/cocos-creator-api-compat/api-compat-polyfill-array.png)

因此，如果使用 2.1.3 以上版本的 Cocos Creator 构建带有 `Array.prototype.includes()` 方法的工程，编译出来的应用将可以顺利在 iOS 8 机器上运行。这是因为 `Array.prototype.includes()` 在构建时被统一被 “翻译” 成了 engine 项目里提供的方法。

相应地，为了避免 Polyfill 里的 `isArray` 、`find`、`includes` 等 API 被 eslint-plugin-builtin-compat 误报，可以在 .eslintrc 中将这些 API 加入该插件的排除列表中：

``` json
// .eslintrc.json
{
  "extends": "eslint:recommended",
  "plugins": [
    "compat",
    "builtin-compat"
  ],
  // ...
  "settings": {
    "builtin-compat-ignore": ["ArrayBuffer", "find", "log2", "parseFloat", "parseInt", "assign", "values", "trimLeft", "startsWith", "endsWith", "repeat"]
  }
  // ...
}
```

### 4.2 自行增加 Polyfill

engine 项目里的 Polyfill 并不能覆盖所有的 API 。如果你希望使用的某个不兼容 API 并没有包含在 engine 项目中，那么就得考虑给你自己的项目补上该 API 的 Polyfill 。

例如，`string.prototype.padStart()` 和 `string.prototype.padEnd()` 两个 API 分别提供了用于字符串的头部和尾部补全的便利方法：

``` javascript
'x'.padStart(5, 'ab')  // 'ababx'
'x'.padStart(4, 'ab')  // 'abax'
'x'.padEnd(5, 'ab')  // 'xabab'
'x'.padEnd(4, 'ab')  // 'xaba'
```

而这两个方法只在 iOS 10 及以上版本才被支持：

![string.prototype.padStart() 方法只在 iOS 10 以上才被支持](/images/cocos-creator-api-compat/api-compat-padstart.png)

#### 寻找 Polyfill

如何寻找这两个方法的 Polyfill 呢？一个最权威的来源就是 MDN 站点（https://developer.mozilla.org/en-US/ ）。以 `string.prototype.padStart()` 为例，我们可以在站点右上角的搜索框中输入 `padStart` ：

![在 MDN 中搜索 padStart](/images/cocos-creator-api-compat/api-compat-padstart-search.png)

之后敲回车进入搜索，在搜索结果中点击最匹配的结果：

![padStart 搜索结果](/images/cocos-creator-api-compat/api-compat-padstart-search-2.png)

就进入了 `string-prototype-padStart` 的文档页，在左侧的导航栏中可以看到有 `Polyfill` 的栏目：

![文档页里的 Polyfill 栏目](/images/cocos-creator-api-compat/api-compat-polyfill-padstart.png)

点击它即可跳转到对应的 Polyfill 实现：

![string-prototype-padStart 的 Polyfill](/images/cocos-creator-api-compat/api-compat-polyfill-padstart.png)

#### 编写自定义的 Polyfill 脚本

找到了 `string.prototype.padStart()` 和 `string.prototype.padEnd()` 两个 API 的 Polyfill 后，我们在自己的工程中编写一个自定义的 Polyfill 脚本。例如叫做 ABCPolyfill.js ：

``` javascript
/**
 * ABCPolyfill.js
 * 补一些 polyfill，解决若干兼容问题
 */

var ABCPolyfill = function () {

    console.log('ABC polyfill');

    if (!String.prototype.padStart) {
        String.prototype.padStart = function padStart(targetLength, padString) {
            targetLength = targetLength >> 0; //truncate if number, or convert non-number to 0;
            padString = String(typeof padString !== 'undefined' ? padString : ' ');
            if (this.length >= targetLength) {
                return String(this);
            } else {
                targetLength = targetLength - this.length;
                if (targetLength > padString.length) {
                    padString += padString.repeat(targetLength / padString.length); //append to original to ensure we are longer than needed
                }
                return padString.slice(0, targetLength) + String(this);
            }
        };
    }

    if (!String.prototype.padEnd) {
        String.prototype.padEnd = function padEnd(targetLength,padString) {
            targetLength = targetLength>>0; //floor if number or convert non-number to 0;
            padString = String((typeof padString !== 'undefined' ? padString : ' '));
            if (this.length > targetLength) {
                return String(this);
            }
            else {
                targetLength = targetLength-this.length;
                if (targetLength > padString.length) {
                    padString += padString.repeat(targetLength/padString.length); //append to original to ensure we are longer than needed
                }
                return String(this) + padString.slice(0,targetLength);
            }
        };
    }

};

module.exports.ABCPolyfill = ABCPolyfill;
```

接下来，我们要在应用启动后加载执行这个 Polyfill 脚本里的 `ABCPolyfill()` 方法，自动打上这两个 API 的 Polyfill 。我们可以再编写一个应用初始化脚本，例如叫做 ABCInit.js ，该脚本用于在应用初始化时执行一些指定工作。

``` javascript
/**
 * ABCInit.js
 * 应用启动时的一些初始化工作
 */
import ABCPolyfill from 'ABCPolyfill';

// 初始化操作
function doInit() {
  ABCPolyfill.ABCPolyfill();
}

(function () {
    doInit();
})();
```

之后可以在你的工程的初始场景里脚本组件中引用该脚本即可生效：

``` javascript
/**
 * 工程的初始场景挂载的脚本组件
 */

require('ABCInit');

// ...
```

为了避免 eslint-plugin-builtin-compat 误报，可以将 `padStart` 和 `padEnd` 也追加进排除名单中：

``` json
// .eslintrc.json
{
  "extends": "eslint:recommended",
  "plugins": [
    "compat",
    "builtin-compat"
  ],
  // ...
  "settings": {
    "builtin-compat-ignore": ["ArrayBuffer", "find", "log2", "parseFloat", "parseInt", "assign", "values", "trimLeft", "startsWith", "endsWith", "repeat", "padStart", "padEnd"]
  }
  // ...
}
```

## 五、小结

1. 时刻注意 API 兼容性；
2. 使用 eslint-plugin-compat 检查静态类型的不兼容 API ，并将告警级别设为错误；
3. 使用 eslint-plugin-builtin-compat 检查动态类型的不兼容 API，并将告警级别设为警告；
4. 考虑为不兼容 API 增加 Polyfill 。

最后，本文是我们正在编写的书籍 **《CocosCreator 最佳实践》** 中的一篇，书籍正在加紧编写中，敬请期待。

## 脚注

[^1]: 所有国家/地区的 Alpha-2 编码可以在这里查询：https://www.iban.com/country-codes 。所有的国家/地区/洲的编码也可以在 node_modules/caniuse-lite/data/regions 里找到。
