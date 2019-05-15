# OEDx 博客

<a href="https://travis-ci.org/oedx/oedx"><img alt="Travis" src="https://img.shields.io/travis/oedx/oedx.svg?style=flat-square"></a>

## 投稿方法

1. 安装 hexo ：

``` sh
npm install -g hexo-cli
```

如果在公司里头，可以选择使用 tnpm ：

``` sh
npm install @tencent/tnpm -g --registry=http://r.tnpm.oa.com --proxy=http://r.tnpm.oa.com:80 --verbose
tnpm install -g hexo-cli
```

2. fork 本仓库到你的 Github 账户下。
3. 克隆你的仓库。

``` sh
git clone https://github.com/你的Github账户/OEDx.git
```

4. 安装依赖：

``` sh
cd OEDx
npm install
```

5. 新建文章：

``` sh
hexo new "My New Post"
```

5. 写文章，同时可以在本地（http://localhost:4000） 开启预览：

``` sh
hexo server
```

5. 确认排版、插图无误后，提交你的文章。
6. 发 Pull Request ，等待管理员审核。
7. 当审核通过后，将自动发布到博客。

## 博客技巧

### 创建一篇新文章

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### 插图

1. 先把图放进 source/images/ 目录，为了避免与其他人的文章插图冲突，请单独为你的文章建一个与文章文件名同名的图片目录。例如：

``` plain
source/images
    |
    +--- cocos-based-high-performance-cross-platform-app-developing/
	| |
	| +-- abcmouse-legacy.png
	| +-- abcmouse-tencent.png
	| +-- ...
	|
	`--- your-post-title/
	  |
	  +-- your-image1.png
	  +-- your-image2.png
```

2. 然后在文章中用下述方式插图：

``` markdown
![ABCmouse](/images/cocos-based-high-performance-cross-platform-app-developing/abcmouse-legacy.png)
```

如果要限制高宽，可以改用 Hexo 的 `img` 插件插图。例如：

``` markdown
{% img /images/cocos-based-high-performance-cross-platform-app-developing/me-on-gmtc.jpg 500 500 我的GMTC首秀 %}
```

### 写好 front-matters

注意写好文章头部的 front-matters 。主要包含几个字段：

* title：文章标题
* date：文章编写时间
* tags: 标签
* categories：分类
* author：作者信息

其中，标签指的是你的文章和什么技术或主题相关，例如 `Cocos`、 `Flutter`、`React Native` 等。例如：

``` yaml
tags: Cocos
```

如果有多个标签，可以使用数组：

``` markdown
tags: [React Native, Redux]
```

分类要求指定为以下几种中的一种：

* 客户端
* 后台
* 前端
* 工程文化
* 其他

作者信息要求保持 `中文名(企业微信名)` 的格式。

``` markdown
author: 潘伟洲(josephpan)
```

如果有博客，允许带上链接：

``` markdown
author: "[潘伟洲(josephpan)](http://hahack.com)"
```

一个完整示例：

``` markdown
---
title: 基于 Cocos 的高性能跨平台开发方案
date: 2018-07-07 17:01:35
tags: Cocos
categories: 客户端
author: "[潘伟洲(josephpan)](http://hahack.com)"
---
```
