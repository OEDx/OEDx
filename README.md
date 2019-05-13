# OEDx 博客

<a href="https://travis-ci.org/oedx/oedx"><img alt="Travis" src="https://img.shields.io/travis/oedx/oedx.svg?style=flat-square"></a>

## 投稿方法

1. 安装 hexo ：

``` sh
npm install -g hexo-cli
```

如果 npm 过慢，可以选择使用 tnpm 。

2. fork 本仓库到你的 Github 账户下。
3. 克隆你的仓库。

``` sh
git clone https://github.com/你的Github账户/OEDx.git
```

4. 新建文章：

``` sh
hexo new post my-title
```

5. 写文章，同时可以在本地（http://localhost:4000） 开启预览：

``` sh
hexo server
```

5. 提交你的改动。
6. 发 Pull Request 等待管理员审核。
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
