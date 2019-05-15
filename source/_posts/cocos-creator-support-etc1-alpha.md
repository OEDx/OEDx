---
title: Cocos Creator 支持ETC1 + Alpha 纹理压缩
date: 2019-05-15 14:40:54
tags: [Cocos, 纹理压缩]
categories: 客户端
author: "[郑桂涛(timorzheng)](http://timorzheng.com)"
---

> ABCmouse是使用Cocos Creator（后面统称CC）开发的App，图片内存占用巨大，在Android低内存机器上容易造成OOM，纹理压缩后的图片可以直接在GPU加载渲染，减少占用内存。而本文基于CC V1.10.x版本做相应的分析及其改造，使项目支持ETC1(Ericsson Texture Compression)+Alpha纹理压缩


### 1、背景
ABCmouse项目中用到的图片资源随着项目开发越来越多，手机Graphic内存占用越来越高，特别是在低端机器上容易Out Of Memory，而项目使用的CC版本是基于V1.10.x较低版本，官网已经升级到V2.0.9，版本差异大升级困难，故而需要在原版上进行支持纹理压缩。

### 2、what、why、how ETC纹理压缩
**What**：ETC是把4x4的像素块压缩成一个64或128位的数据块，是**有损压缩**，移动平台游戏比较常用的压缩方式之一（iOS设备中支持的是PVR压缩和ETC2，在Android中支持的是ETC压缩）。而ETC又分为ETC1和ETC2，ETC2是向下兼容ETC1，对RGB的压缩质量更好，并且支持透明通道，当然对软件硬件也是有一定的要求。而各种Android设备基本都支持ETC1，**ETC1不支持透明通道**。

ETC1、ETC2对比如下：

| 类型 | 是否支持Alpha | 支持OpenGLES版本 | Android版本支持 | 压缩率 | 压缩质量 |
| :------:| :------: | :------: | :------:| :------: | :------: |
| ETC1 | No | OpenGLES 2.0+ | Android 2.2 (API level 8) and higher | 6:1 | 好 |
| ETC2 | Yes | OpenGLES 3.0+ | Android 4.3 (API level 18) and higher | 6:1 | 较好 |

参考OpenGL ES 版本Android占比分布：https://developer.android.com/about/dashboards/index.html#Screenshttps://developer.android.com/guide/topics/graphics/opengl.html

硬件方面：虽说ETC2支持Android版本4.3+，但并非所有的4.3机器都支持ETC2，这取决于手机厂商定制的GPU型号是否支持。

**Why**：而使用纹理压缩有什么好处呢？纹理压缩后的图片，不经过CPU解码，直接使用GPU加载渲染，大大提高了加载速度。
为此，很多游戏App采用ETC1+Alpha来解决ETC1不支持透明通道的问题。

**How**：那么，如何生成ETC1+Alpha呢？首先移动端是无法直接在移动端生成etc，需要额外的制作工具。这里推荐使用Mali Texture Compression Tool，这个工具可以生成ETC1和带透明通道的ETC1，下载地址：https://developer.arm.com/tools-and-software/graphics-and-gaming/graphics-development-tools/mali-texture-compression-tool/downloads

### 3、实现方案
基于我们项目中，目前Android配置的minSdkVersion是19，那么是否可以考虑直接使用ETC2呢？抱着测试的心态，先使用上面介绍的Mali工具，使用以下命令直接将图片压缩为ETC2格式：
```
etcpack srcfile outfile -c etc2 -f RGBA
```
 具体CC编译后的Android包如何加载ETC2可以参看文章：https://forum.cocos.com/t/cocos-etc2/49061  这里不再详述。
然而前面已经提到的有些低端机器不支持ETC2的加载，这导致了渲染黑屏。通过CC引擎检测是否支持ETC2 log我们可以看到，该手机硬件不支持，具体log如下：
```
D/cocos2d-x: {
        gl.supports_OES_packed_depth_stencil: true
        gl.supports_vertex_array_object: true
        gl.supports_BGRA8888: false
        cocos2d.x.version: Cocos2d-x-lite v1.8.2
        gl.supports_discard_framebuffer: true
        cocos2d.x.compiled_with_profiler: false
        gl.supports_PVRTC: false
        cocos2d.x.build_type: DEBUG
        gl.renderer: Adreno (TM) 530
        gl.supports_OES_depth24: true
        gl.supports_ETC1: true
        gl.supports_OES_map_buffer: false
        cocos2d.x.compiled_with_gl_state_cache: true
        gl.version: OpenGL ES 3.2 V@313.0 (GIT@984b9a6, Ibe1bf21abc) (Date:06/04/18)
        gl.supports_NPOT: true
        gl.supports_ETC2: false
        gl.max_texture_units: 96
        gl.vendor: Qualcomm
        gl.max_texture_size: 16384
    }
cocos2d-x: cocos2d: Hardware ETC2 decoder not support.
```
考虑到需要兼容部分低端机型，这里不得不放弃ETC2的使用，转而尝试使用ETC1+Alpha。同样的，使用Mali工具对图片进行纹理压缩命令如下：（使用命令生成出来的纹理上半部分是原始图片（无alpha信息），下半部分是alpha信息图片）
```
etcpack srcfile outfile -c etc -aa
```

![原始图](/images/cocos-creator-support-etc1-alpha/etc_mipmap0_combined.png)

![ETC1+Alpha纹理压缩后预览图](/images/cocos-creator-support-etc1-alpha/etc_mipmap0_color.png)


可以看到纹理压缩后的图片高度是原来图片高度的2倍，那么最关键的就是如何让其渲染成原始图片。答案是使用Shader对其进行渲染。
首先了解一下CCSprite、CCSpriteFrame、CCTexture2D之间的关系（图来自CC官网文档）：
![assets](/images/cocos-creator-support-etc1-alpha/cc.png)

从图中可以看到，我们肉眼看到的是CCSprite渲染出来的图片，CCSpriteFrame为精灵的某一帧，CCTexture2D为图片纹理数据，也对应上从Cocos js加载图片到sprite中代码：
```
properties: {
   sprite: cc.Sprite,
}
...
cc.loader.load(path, function (err, texture) {
    this.sprite.spriteFrame = new cc.SpriteFrame(texture);
});
```
 了解了以上概念后，纹理压缩后出来的文件就是对应的CCTexture2D，如何让其显示到CCSprite上就是我们要做的处理。
我们先看一下整体的实现流程图（从构建-->纹理压缩-->打包apk-->图片渲染）：
![流程图](/images/cocos-creator-support-etc1-alpha/flow.jpg)


### 4、开发过程遇到的问题
如果直接使用CC v1.10.x版本的代码直接加载etc1+alpha文件，那么将出现各种问题：

- 图片移位
- 渲染黑块
- 图片遮罩不生效
- 自定义Shader不生效
- cocos js获取Texture的height变成实际显示高度的2倍

**✔️ cc.loader.load动态加载的图片将出现移位**
解决的最关键一步是获取Texture2D的pixelFormat为ETC格式时需要update SpriteFrame的几个属性，具体实现需要修改c++层的CCSpriteFrame.cpp进行适配：
```
void SpriteFrame::setTexture(Texture2D * texture)
{
    if( _texture != texture ) {
        CC_SAFE_RELEASE(_texture);
        CC_SAFE_RETAIN(texture);
        if(texture->getPixelFormat() == Texture2D::PixelFormat::ETC){
            int texHigh = texture->getPixelsHigh();
            if(texHigh == _rect.size.height){
                _rect.size.height *= 0.5;
                _rectInPixels = CC_RECT_POINTS_TO_PIXELS(_rect);
                Size size = CC_SIZE_POINTS_TO_PIXELS(texture->getContentSize());
                size.height *= 0.5;
                _originalSizeInPixels = size;
                _originalSize = CC_SIZE_PIXELS_TO_POINTS( _originalSizeInPixels);
            }
        }
        _texture = texture;
    }
}
```
 可以看到我们将_rect.size.height和contentSize.height减半并更新_rect、_rectInPixels、_originalSizeInPixels、_originalSiz属性

**✔️ 渲染黑块**
解决渲染黑块，使用shader对etc纹理进行渲染，将遮罩部分作为Alpha值加到原图上
顶点着色器 ccShader_PositionTexture_Ect1Alpha.vert:
```
const char* ccPositionTexture_Ect1Alpha_vert = STRINGIFY(
attribute vec4 a_position;
attribute vec2 a_texCoord;
attribute vec4 a_color;
varying vec4 v_fragmentColor;
varying vec2 v_texCoord;
varying vec2 v_alphaCoord;
void main()
{
    gl_Position = CC_PMatrix * a_position;
    v_fragmentColor = a_color;
    v_texCoord = a_texCoord;
}
);
```
片段着色器 ccShader_PositionTexture_Ect1Alpha.frag:
```
const char* ccPositionTexture_Ect1Alpha_frag = STRINGIFY(

\n#ifdef GL_ES\n
precision lowp float;
\n#endif\n
varying vec4 v_fragmentColor;
varying vec2 v_texCoord;
varying vec2 v_alphaCoord;
void main()
{
    vec4 v4Colour = texture2D(CC_Texture0, v_texCoord);
    v4Colour.a = texture2D(CC_Texture0, vec2(0.0, 0.5) + v_texCoord).r;
    //v4Colour.rgb *= v4Colour.a;//Premultiply with Alpha channel
    gl_FragColor = v_fragmentColor * v4Colour;
}
);
```
 具体的shader语法可参考：https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/#_3

**✔️ 图片遮罩不生效**
![图片遮罩](/images/cocos-creator-support-etc1-alpha/mask.png)

CC上预览是可以，但由于mask上设置的SpriteFrame经过纹理压缩后，在app上就展示不出mask效果。需要找到mask设置SpriteFrame对应的c++层逻辑，对应CCClippingNode.cpp如下方法：
```
void ClippingNode::visit(Renderer *renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    ……省略部分代码

    auto alphaThreshold = this->getAlphaThreshold();
    if (alphaThreshold < 1)
    {
#if CC_CLIPPING_NODE_OPENGLES
        // since glAlphaTest do not exists in OES, use a shader that writes
        // pixel only if greater than an alpha threshold
        //
        GLProgram *program = GLProgramCache::getInstance()->getGLProgram(GLProgram::SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST_NO_MV);
        if(_stencil != nullptr){
            if (auto scale9sp = dynamic_cast<creator::Scale9SpriteV2*>(_stencil)){
                cocos2d::SpriteFrame* spriteFrame = scale9sp->getSpriteFrame();
                if(spriteFrame && spriteFrame->getTexture() && spriteFrame->getTexture()->getPixelFormat() == Texture2D::PixelFormat::ETC){
                    program = GLProgramCache::getInstance()->getGLProgram(GLProgram::SHADER_NAME_POSITION_TEXTURE_ETC_ALPHA_TEST_NO_MV);
                }
            }
        }
        GLint alphaValueLocation = glGetUniformLocation(program->getProgram(), GLProgram::UNIFORM_NAME_ALPHA_TEST_VALUE);
        // set our alphaThreshold
        program->use();
        program->setUniformLocationWith1f(alphaValueLocation, alphaThreshold);
        // we need to recursively apply this shader to all the nodes in the stencil node
        // FIXME: we should have a way to apply shader to all nodes without having to do this
        setProgram(_stencil, program);
#endif

    }

    ……省略部分代码
}
```
找到alphaThreshold小于1的地方，要应用GLProgram之前先判断PixelFormat格式如果是etc则重新设置shader，SHADER_NAME_POSITION_TEXTURE_ETC_ALPHA_TEST_NO_MV是我重新设置的shader，区别于SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST_NO_MV只是关键的一行代码：
```
const char* ccPositionEtcTextureColorAlphaTest_frag = STRINGIFY(

\n#ifdef GL_ES\n
precision lowp float;
\n#endif\n

varying vec4 v_fragmentColor;
varying vec2 v_texCoord;
uniform float CC_alpha_value;

void main()
{
    vec4 texColor = texture2D(CC_Texture0, v_texCoord);
    texColor.a = texture2D(CC_Texture0, vec2(0.0, 0.5) + v_texCoord).r;//关键的一行：设置Alpha层叠加到原图上

\n// mimic: glAlphaFunc(GL_GREATER)
\n// pass if ( incoming_pixel >= CC_alpha_value ) => fail if incoming_pixel < CC_alpha_value\n

    if ( texColor.a <= CC_alpha_value )
        discard;

    gl_FragColor = texColor * v_fragmentColor;
}
);
```

**✔️ 自定义Shader不生效**
 ABCmouse项目中的bookplayer使用了自定义Shader，加了纹理压缩后，发现翻书shader效果失效了，而翻书用的图片是网络图片(没有经过纹理压缩)，理论上跟etc纹理没关系。

后面定位到是CCScale9Sprite.cpp下setSpriteFrame当spriteFrame为空时重新设置了默认shader（SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP）导致，而bookplayer刚好就是有设置sprite.spriteFrame = null;的情况，解决方法去掉设置默认shader即可
```
bool Scale9SpriteV2::setSpriteFrame(cocos2d::SpriteFrame* spriteFrame)
{
    if(this->_spriteFrame == nullptr){
        if (spriteFrame && spriteFrame->getTexture()) {
            Texture2D::PixelFormat pixelFormat = spriteFrame->getTexture()->getPixelFormat();
            if (pixelFormat == Texture2D::PixelFormat::ETC) {
                this->setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_TEXTURE_ETC1ALPHA));
            } else{//自己添加的下面这行导致shader被覆盖，去掉即可
                this->setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP));
            }
        }
    }
   ......忽略其他代码
    return true;
}
```
那么还可以思考的是，要是etc纹理做自定义shader应该如何处理呢？项目中暂时没有这种暂时没支持，编码思路是判断是否有自定义纹理，有则使用自定义，没有则用etc shader

**✔️ 经过ETC纹理压缩后，cocos js获取Texture的height变成实际显示高度的2倍**
解决方法是在cc.Texture2D新增获取真正height的方法，如下：
```
cc.Texture2D.prototype.getWrapPixelHeight = function(){//Android本地图片使用etc+alpha纹理压缩，height、getPixelHeight()、pixelHeight是显示height的两倍，请使用这个wrap方法获取修复
        if(this.getPixelFormat() == 14){//etc+alpha 高度减半处理
            return this.height * 0.5;
        }
        return this.height;
    };
```

### 5、纹理压缩白名单
由于etc是有损压缩，对于一些设计师图片有质量要求（特别是alpha透明度有要求）的需要添加到白名单里，不进行压缩。

目前白名单主要有：AutoAtlas和骨骼动画、压缩后alpha有问题的图片

### 6、优化后的内存效果
> Graphics：图形缓冲区队列向屏幕显示像素（包括 GL surfaces, GL textures等等）所使用的内存

使用Memory Profiler分析内存占用情况，在同一场景gc后稳定内存，然后分别加载png图片和etc图片，对比内存如下：
![png内存占用](/images/cocos-creator-support-etc1-alpha/png_memory.png)
![etc内存占用](/images/cocos-creator-support-etc1-alpha/etc_memory.png)

| 类型 | 加载图片前 | 加载图片后 | 占用内存 |
| :------:| :------: | :------: | :------:|
| png | 176.3M | 182M | 5.7M |
| etc | 195.5M | 197.3M | 1.8M |

### 附加参考：
>CC论坛：cocos实现对ETC2的支持：https://forum.cocos.com/t/cocos-etc2/49061
CC论坛：Creator使用压缩纹理：https://forum.cocos.com/t/creator/47206
cocos2d中添加自己的shader教程：http://www.cocoachina.com/bbs/read.php?tid=220630
OpenGL着色器语言(GLSL)：https://learnopengl-cn.github.io/01%20Getting%20started/05%20Shaders/
关于cocos2d-x对etc1图片支持的分析：https://blog.csdn.net/langresser_king/article/details/9339313
Mali-Texture-Compression工具：https://developer.arm.com/tools-and-software/graphics-and-gaming/graphics-development-tools/mali-texture-compression-tool/downloads

以上如有错误疏漏，烦请指正