---
title: Cocos Creator 纹理压缩插件化
date: 2019-06-20 19:40:29
tags: [Cocos Creator]
categories: 客户端
author: xiangwenlai
---

>在一些游戏类应用或者移动应用中，图片资源所耗的内存往往是整个App内存占用的大头。在早期开发过程中，为了追求更好的体验效果，往往使用了很多高清的资源，这些高清资源往往占用了很大的内存。 而对于移动端设备来说，内存是有限的，尤其对于一些早期的设备，比如第一代的iPad，RAM才512MB，如果要兼容这些老款的设备，无疑对内存的使用要变得很节制。特别对于iOS设备来说，App的内存不能无限制的使用下去，如果内存占用过多，系统在发出内存警告以后如果还无法释放出更多的空闲内存出来，后面就意味着你的App已经送上了断头台，有随时被系统咔嚓掉的风险。为提升App的稳定性以及执行的效率，降低内存占用是每个开发人员一直在追求的目标。

## 如何降图片内存

通常设计师给的图片格式有PNG、JPG、TGA等，比如png图片，一张1024\*1024分辨率的图片，文件大小才几百KB甚至更小，这些图片是经过特殊编码，减少了冗余信息，降低了存储空间的占用，易于网络传输。然而这些格式的图片读到内存以后，并不能直接交个GPU进行绘制，需要通过CPU解码成位图以后，才能给GPU绘制。以32位png格式图片，分辨率1024\*1024图片为例，解码成位图以后占用的内存大小为4 \* 1024 \* 1024 \* 8 bit，占用内存为4MB。降低图片内存占用，有下面常见方法：

### 常见方法

* 降低图片分辨率：根据实际情况，在不影响整体体验效果的同时，适当降低分辨率。
* 减小图片每个像素的位数：比如将32位png图片转成了24位、16位，甚至更低，根据实际效果决定。
* 使用调色板技术：每个像素只是调色板的索引，大幅降低内存占用。这个只能对颜色数少的图片使用，颜色丰富的图片，颜色失真，效果很差。
* 纹理压缩：使用标准的压缩算法对纹理进行压缩。

## 纹理压缩

简单来说，就是将原始位图经过标准的算法进行编码压缩，压缩后的数据能够直接被GPU读取解码渲染。减少了CPU解码的过程以及转成bitmap的内存占用。因此使用纹理压缩，可有效降低内存的占用以及提升渲染效率。

### 常用的纹理压缩格式

#### DXT

DXT纹理压缩格式来源于S3(Silicon & Software Systems)公司提出的S3TC，基本思想是把4x4的像素块压缩成一个64或128位的数据块，是有损压缩方式。 S3TC算法有五种变化DXT1-DXT5，一般在Windows设备上面使用，支持的GPU: Windows\Android(Nvidia Tegra and Intel Bay Trail)。 五种算法的格式对比如下表：

![](/images/cocos-texturecompression-plugin/ttt.png)

#### ETC

Ericsson Texture Compression，是由 Khronos 支持的开放标准，在移动平台中广泛采用。它是一种为感知质量设计的有损算法，其依据是人眼对亮度改变的反应要高于色度改变。类似于DXT，ETC也是把4x4的像素块压缩成一个64或128位的数据块，也是有损压缩。
ETC有两种压缩格式：ETC1和ETC2。

##### ETC1

ETC1是早期推出的纹理压缩格式，也是兼容性最好的格式。基本上上能够兼容所有支持OpenGLES2.0的Android设备，但是iOS设备都不支持。ETC1不支持Alpha通道，如果需要支持Alpha，需要经过特殊的处理才能支持：将图片的Alpha通道分离出来，跟24位RGB图合成一张原来2倍大小的图，通过自定义shadder来绘制。Cocos Creator里面的做法可以参考教程：[Cocos Creator 支持ETC1 + Alpha 纹理压缩](https://oedx.github.io/2019/05/15/cocos-creator-support-etc1-alpha/)。

##### ETC2

ETC2 是ETC1的扩展，向下兼容ETC1，对RGB压缩质量较好并且支持Alpha通道。ETC2相比ETC1压缩质量更高，从视觉上看也是接近了原图的效果。不过ETC2的兼容性不太好，OpenGLES3.0才把它纳入标准，而且对硬件要求也较高，iOS设备需要苹果A7以上的设备才能支持，Android大部分设备都支持，但是不同厂商差异很大，经过测试，发现有些Android 6.0的设备都不支持。两种格式对比如下：


![](/images/cocos-texturecompression-plugin/ttt1.png)


### PVRTC
PowerVR Texture Compression，PVRTC格式与基于块的压缩格式，比如S3TC、ETC的不同之处是，它使用2张双线性放大的低分辨率图，根据精度和每个像素的权重，融合到一起来呈现纹理，并且2-bpp和4-bpp都支持ARGB数据。PVRTC格式压缩比高，也是有损压缩。PVRTC有2bpp和4bpp格式，2bpp每个像素2bit，压缩率很高，但是质量较差。4bpp每个像素4bit，质量相对更好一些。具体用哪个，需要根据实际情况而定。
PVRTC压缩对图片的规格要求较高，图片的大小必需是正方形而且边长必需是2的N次幂。该纹理压缩对iOS兼容性好，支持所有的iOS设备以及部分使用PowerVR GPU的Android设备。
>后来Imagenation 公司推出了PVRTC2纹理压缩格式，它是对PVRTC进行的改进，添加了NPOT(非2方)和贴图集支持，而且压缩质量也有很大的提高。但是必需有PowerVR Series5XT或Series6 GPU 才支持，苹果设备没有支持，估计这个格式也是凉凉了。

![](/images/cocos-texturecompression-plugin/ttt2.png)

#### ASTC

自适应可伸缩纹理压缩(Adaptive Scalable Texture Compression,ASTC)是一种基于像素块的有损纹理压缩算法，由ARM的Jørn Nystad及其他人员共同设计。在2012年8月6日，ASTC被Khronos Group正式采纳为OpenGL和OpenGL ES的正式扩展。ASTC同样是基于block的压缩方式，但块的大小却较支持多种尺寸，比如从基本的4x4到12x12，而且块的宽高也不限于pot，比如6x5；每个块内的内容用128bits来进行存储，因而不同的块就对应着不同的压缩率。兼容性方面，OpenGLES3.0以上支持，支持多数Android 高端机器以及iPhone 6以上机型。压缩率如下表：

|Block Size|Bits Per Pixel|Comp.Ratio|
|:---:|:---:|:---:|
|4x4| 8.00|4:1|
|5x4| 6.40|5:1|
| 5x5 | 5.12|6.25:1|
|6x5| 4.27|7.5:1|
|6x6| 3.56|9:1|
|8x5| 3.20|    10:1|
|8x6| 2.67|12:1|
|10x5|     2.56| 12.5:1 |
| 10x6 | 2.13 |15:1|
| 8x8 |     2.00|16:1|
|10x8|     1.60|20:1|
|10x10| 1.28|25:1|
|12x10| 1.07|30:1|
|12x12| 0.89|36:1|


## Cocos Creator 纹理压缩插件化

从上面纹理压缩的介绍可以看出，没有任何一种格式是完美的，都有它使用的场景以及局限性在里面。ABCmouse是一款类游戏的儿童英语教学App，使用的Cocos Creator进行开发并发布到全平台。由于App需要兼容老款低端的设备，因此我们采用了混合压缩的方式：

* iOS采用PVR + ETC2进行纹理压缩
* Android采用ETC1进行纹理压缩
* 根据具体情况，有些图片如果无法保证质量，将采用原图而不进行纹理压缩，以保证设计需要的效果

### 为什么要插件化

了解Cocos Creator的开发应该清楚，每次新开发功能，修改图片或者脚本以后，如果需要看看在真机上的效果，都需要在Cocos Creator上触发一次构建。建构完成以后，会在build目录下生成一个包含所有图片的目录。如果需要纹理压缩，需要手动触发一下纹理压缩的工具，这种流程非常频繁而且过于机械化。因此，我们考虑将纹理压缩功能以插件的形式集成到Cocos Creator，在Cocos Creator构建完成以后自动触发纹理压缩，使整个流程更加**自动化**。

### 原理

Cocos Creator插件能力，官方有文档可以参考。关键的一步，就是监听Cocos Creator构建完成的事件 `editor:build-finished`，在监听到这个事件以后启动纹理压缩。

```
'editor:build-finished': function (event, target) {
    if (packerOpened) {
        process.nextTick(() => {
            texPacker.handleNoPack();
            texPacker.generateImgResourceMap();
            Editor.log('开始压缩纹理...');
            texPacker.startPack();
        });
    } else {
        Editor.log('纹理压缩已关闭，不进行纹理压缩。');
    }
}
```

我们也在插件中提供了一些菜单选项，可以临时打开或者关闭纹理压缩，开启关闭纹理压缩菜单：

```
'packtexture:opentexturepack'() {
    packerOpened = true;
    Editor.log('纹理压缩已开启');
},

'packtexture:closetexturepack'() {
    packerOpened = false;
    Editor.log('纹理压缩已关闭');
},
```

纹理压缩以后，我们发现压缩以后的纹理比原始的JPG、PNG图片更大，影响到安装包的大小。因此，最后在纹理压缩完成以后再采用了gzip压缩，最后安装包不会明显增长，甚至有所降低：

```
packETCAplha: function (srcPath, sfile, dfile, callBack, dfile2) {
    let packTemp = sfile.replace(/\.[^/.]+$/, "");
    let outputPath = packTemp + '.pkm';
    let shell = 'etcpack ' + sfile + ' ' + srcPath + ' -c etc -aa';
    exec(shell, {
        encoding: 'utf8',
        env: process.env
    }, (err, stdout, stderr) => {
        let finish = true;
        if (fs.existsSync(outputPath)) {
            fs.createReadStream(outputPath)
            .pipe(zlib.createGzip())
            .pipe(fs.createWriteStream(dfile));
            if (dfile2) {
                fs.createReadStream(outputPath)
                .pipe(zlib.createGzip())
                .pipe(fs.createWriteStream(dfile2));
            }
            fs.unlinkSync(outputPath);
        } else {
            finish = false;
            let buffer = fs.readFileSync(sfile);
            fs.writeFileSync(dfile, buffer);
            Editor.log('压缩ETC+Alpha 失败, ' + sfile);
        }
        if (callBack) {
            callBack(finish, dfile);
        }
    });
},
```

### 增量压缩

有些比较大的图片，压缩一次的耗时是比较长的，需要几秒钟。对于一个稍具规模的项目来说，图片数量有很多，压缩一次耗时很长，需要几十秒甚至几分钟时间。如果每次都全量重新压缩一次的话，耗费时间太久，影响开发的效率。

因此，我们根据图片的md5值进行对比，只对新增加或者有修改的图片进行压缩，无修改的图片直接使用上次压缩的结果，大大节省了纹理压缩的时间，提升了开发效率。

全量压缩，可以看出每一项压缩完耗时都较长：

![](/images/cocos-texturecompression-plugin/pack1.gif)


增量压缩，每项检查都很快：

![](/images/cocos-texturecompression-plugin/pack4.gif)


我们这个插件已经开源[CocosCreator 纹理压缩插件](https://github.com/OEDx/ccc-texturecompression)。

## 总结

纹理压缩是降低图片内存占用节省GPU带宽的有效手段，对于图片占比很多的场景还是可以考虑使用纹理压缩。本文介绍了当下常见的一些纹理压缩格式，简要分析了其特点。具体选择哪种压缩格式的时候，需要结合实际情况，采用一种或者多种格式想结合的方式，获得最终想要的结果。后面简单介绍了纹理压缩工具以插件形式和Cocos Creator结合得例子，希望对有遇到类似问题的人有些帮助。

