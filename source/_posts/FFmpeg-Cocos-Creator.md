---
title: 基于 FFmpeg 的 Cocos Creator 视频播放器
date: 2020-12-29 17:03:37
tags: Cocos
categories: 客户端
author: "黄成华(galiohuang)"
---

> 只为解决一个核心问题，追求更好体验。

## 背景

腾讯开心鼠项目使用的游戏引擎是 Cocos Creator，由于引擎提供的视频组件实现方式问题导致视频组件和游戏界面分了层，从而导致了以下若干问题:

1. 不可以在视频组件上添加其他渲染组件；
2. 不可以使用遮罩组件来限定视频形状；
3. 退出场景时存在视频组件残影；
4. 等等...

核心问题就是分层问题，对于开心鼠项目带来的最大弊端就是：一套设计，Android，iOS，Web 三端需要各自实现，开发和维护成本高，又因为平台差异化，还存在视觉不一致和表现不一致问题。

## 解决方案

![](/images/FFmpeg-Cocos-Creator/1609213303_85_w2856_h1188.png)

因为开心鼠项目需要兼容 Android，iOS 和 Web 三端，Android 和 iOS 一起视为移动端，所以解决方案有以下两点：

1. 移动端可使用 FFmpeg 库解码视频流，然后使用 OpenGL 来渲染视频，和使用 Andorid, iOS 两端各自的音频接口来播放音频；
2. 网页端可以直接使用 `video` 元素来解码音视频，然后使用 WebGL 来渲染视频，和使用 `video` 元素来播放音频。

## 任务细分

![](/images/FFmpeg-Cocos-Creator/ntB1Ew.png)

## 任务详情

### 移动端 ffplay 播放音视频

FFmpeg 官方源码，可以编译出三个可执行程序，分别是 ffmpeg, ffplay, ffprobe ，三者作用分别是：

1. ffmpeg 用于音视频视频格式转换，视频裁剪等；
2. ffplay 用于播放音视频，需要依赖 SDL；
3. ffprobe 用于分析音视频流媒体数据。

其中 ffplay 程序满足了播放音视频的需求，理论上，只要把 SDL 视频展示和音频播放接口替换成移动端接口，就能完成 Cocos Creator 的音视频播放功能，但在实际 ffplay 改造过程中，还是会遇到很多小问题，例如：在移动端使用 swscale 进行纹理缩放和像素格式转换效率低下，不支持 Android asset 文件读取问题等等，下文会逐一解决。经过一系列改造后，Cocos Creator 可用的 AVPlayer 诞生了。以下为 AVPlayer 播放音视频流程分析：

![](/images/FFmpeg-Cocos-Creator/1606737160_43_w4030_h2180.png)

概括：

1. 调用 stream_open 函数，初始化状态信息和数据队列，并创建 read_thread 和 refresh_thread；
2. read_thread 主要职责为打开流媒体，创建解码线程（音频，视频，字幕），读取原始数据；
3. 解码线程分别解码原始数据，得到视频图片序列，音频样本序列，字幕字符串序列；
4. 在创建音频解码器过程中，同时打开了音频设备，在播放过程中，会不断消耗生成的音频样本；
5. refresh_thread 主要职责为不断消耗视频图片序列和字幕字符串序列。

ffplay 改造后的 AVPlayer UML如下：

![](/images/FFmpeg-Cocos-Creator/Wn9Lbz.png)

声明：因为本人少接触 c 和 c++ ，所以在 ffplay 改造过程中，SDL 线程改造和字幕分析参考了 bilibili 的 ijkplayer 源码。

### JSB 绑定视频组件接口

此节不适合 Web 端，关于 JSB 相关知识，可查阅文档：[JSB 2.0 绑定教程](https://docs.cocos.com/creator/manual/zh/advanced-topics/JSB2.0-learning.html)

概括 JSB 功能：通过 ScriptEngine 暴露的接口绑定 JS 对象和其他语言对象，让 JS 对象控制其他语言对象。

因为播放器逻辑使用 C 和 C++ 编码，所以需要绑定 JS 和 C++ 对象。上文中的 AVPlayer 只负责解码和播放流程，播放器还需要处理入参处理，视频渲染和音频播放等工作，因此封装了一个类：Video，其 UML 如下：

![](/images/FFmpeg-Cocos-Creator/aHbvFN.png)

Video.cpp 绑定的 JS 对象声明如下：

```cpp
bool js_register_video_Video(se::Object *obj) {
    auto cls = se::Class::create("Video", obj, nullptr, _SE(js_gfx_Video_constructor));
    cls->defineFunction("init", _SE(js_gfx_Video_init));
    cls->defineFunction("prepare", _SE(js_gfx_Video_prepare));
    cls->defineFunction("play", _SE(js_gfx_Video_play));
    cls->defineFunction("resume", _SE(js_gfx_Video_resume));
    cls->defineFunction("pause", _SE(js_gfx_Video_pause));
    cls->defineFunction("currentTime", _SE(js_gfx_Video_currentTime));
    cls->defineFunction("addEventListener", _SE(js_gfx_Video_addEventListener));
    cls->defineFunction("stop", _SE(js_gfx_Video_stop));
    cls->defineFunction("clear", _SE(js_gfx_Video_clear));
    cls->defineFunction("setURL", _SE(js_gfx_Video_setURL));
    cls->defineFunction("duration", _SE(js_gfx_Video_duration));
    cls->defineFunction("seek", _SE(js_gfx_Video_seek));
    cls->defineFunction("destroy", _SE(js_cocos2d_Video_destroy));
    cls->defineFinalizeFunction(_SE(js_cocos2d_Video_finalize));
    cls->install();
    JSBClassType::registerClass<cocos2d::renderer::Video>(cls);

    __jsb_cocos2d_renderer_Video_proto = cls->getProto();
    __jsb_cocos2d_renderer_Video_class = cls;

    se::ScriptEngine::getInstance()->clearException();
    return true;
}

bool register_all_video_experiment(se::Object *obj) {
    se::Value nsVal;
    if (!obj->getProperty("gfx", &nsVal)) {
        se::HandleObject jsobj(se::Object::createPlainObject());
        nsVal.setObject(jsobj);
        obj->setProperty("gfx", nsVal);
    }
    se::Object *ns = nsVal.toObject();

    js_register_video_Video(ns);
    return true;
}
```

 概括：以上声明，表示可在 JS 代码中，使用以下方法

```javascript
let video = new gfx.Video();                                // 构造函数

video.init(cc.renderer.device, {                            // 初始化参数
    images: [],                                                         
    width: videoWidth,                                                 
    height: videoHeight,                                                
    wrapS: gfx.WRAP_CLAMP,                                              
    wrapT: gfx.WRAP_CLAMP,
});

video.setURL(url);                                          // 设置资源路径
video.prepare();                                            // 调用准备函数
video.play();                                               // 播放
video.pause();                                              // 暂停
video.resume();                                             // 恢复
video.stop();                                               // 停止
video.clear();                                              // 清理
video.destroy();                                            // 销毁
video.seek(position);                                       // 跳转

let duration = video.duration();                            // 获取视频时长
let currentTime = video.currentTime();                      // 获取当前播放位置

video.addEventListener('loaded', () => {});                 // 监听 Meta 加载完成事件
video.addEventListener('ready', () => {});                  // 监听准备完毕事件
video.addEventListener('completed', () => {});              // 监听播放完成事件
video.addEventListener('error', () => {});                  // 监听播放失败事件
```

### 视频展示，纹理渲染

实现视频展示功能，需要先了解纹理渲染流程，由于 Cocos Creator 在移动端使用的是 OpenGL API，在 Web 端使用的 WebGL API，OpenGL API 和 WebGL API 大致相同，因此可以到 OpenGL 网站学习下纹理渲染流程。初学者，推荐到 [LearnOpenGL CN](https://learnopengl-cn.github.io/) 学习。接下来使用 [LearnOpenGL CN](https://learnopengl-cn.github.io/) 纹理章节讲解以下纹理渲染流程。

顶点着色器：

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;

out vec2 TexCoord;

void main()
{
    gl_Position = vec4(aPos, 1.0);
    TexCoord = vec2(aTexCoord.x, aTexCoord.y);
}
```

 片段着色器：

```glsl
#version 330 core
out vec4 FragColor;

in vec2 TexCoord;

uniform sampler2D tex;

void main()
{
    FragColor = texture(tex, TexCoord);
}
```

 纹理渲染程序：

```c
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <stb_image.h>

#include <learnopengl/shader_s.h>

#include <iostream>

// 窗口大小
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;

int main()
{
    // 初始化窗口
    // --------
    GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
    ...

    // 编译和链接着色器程序
    // ----------------
    Shader ourShader("4.1.texture.vs", "4.1.texture.fs"); 

    // 设置顶点属性参数
    // -------------
    float vertices[] = {
        // 位置                // 纹理坐标
         0.5f,  0.5f, 0.0f,   1.0f, 0.0f, // top right
         0.5f, -0.5f, 0.0f,   1.0f, 1.0f, // bottom right
        -0.5f, -0.5f, 0.0f,   0.0f, 1.0f, // bottom left
        -0.5f,  0.5f, 0.0f,   0.0f, 0.0f  // top left 
    };

    // 设置索引数据，此程序画的图形基元是三角形，图片为矩形，所以由两个三角形组成
    unsigned int indices[] = {  
        0, 1, 3, // first triangle
        1, 2, 3  // second triangle
    };

    // 声明和创建 VBO 顶点缓冲对象，VAO 顶点数组对象，索引缓冲对象
    // C 语言并非面向对象编程，这里使用无符号整形来代表对象
    unsigned int VBO, VAO, EBO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);

    // 绑定顶点对象数组，可记录接下来设置的缓冲对象数据，方便在渲染循环中使用
    glBindVertexArray(VAO);

    // 绑定顶点缓冲对象，用于传递顶点属性参数
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

    // 绑定索引缓冲对象，glDrawElements 会按照索引顺序画图形基元
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

    // 链接顶点属性：位置， 参数：索引，大小，类型，标准化，步进，偏移
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);

    // 链接顶点属性：纹理坐标，参数：索引，大小，类型，标准化，步进，偏移
    glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)(3 * sizeof(float)));
    glEnableVertexAttribArray(1);


    // 生成纹理对象
    // -------------------------
    unsigned int texture;
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);

    // 设置环绕参数
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    
    // 设置纹理过滤
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    // 加载图片
    int width, height, nrChannels;
    unsigned char *data = stbi_load(FileSystem::getPath("resources/textures/container.jpg").c_str(), &width, &height, &nrChannels, 0);
    if (data)
    {
        // 传递纹理数据，参数：目标，级别，内部格式，宽，高，边框，格式，数据类型，像素数组
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
        glGenerateMipmap(GL_TEXTURE_2D);
    }
    else
    {
        std::cout << "Failed to load texture" << std::endl;
    }
    stbi_image_free(data);


    // 渲染循环
    // -------
    while (!glfwWindowShouldClose(window))
    {
        // 清理
        // ---
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        // 绑定纹理
        glBindTexture(GL_TEXTURE_2D, texture);

        // 应用 shader 程序
        ourShader.use();

        // 绑定顶点数组对象
        glBindVertexArray(VAO);

        // 绘制三角形基元
        glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

        // glfw 交换缓冲
        glfwSwapBuffers(window);
    }

    // 清理对象
    // ------------------------------------------------------------------------
    glDeleteVertexArrays(1, &VAO);
    glDeleteBuffers(1, &VBO);
    glDeleteBuffers(1, &EBO);

    // 结束
    // ------------------------------------------------------------------
    glfwTerminate();
    return 0;
}
```

 简单的纹理渲染流程：

1. 编译和链接着色器程序；
2. 设置顶点数据，包括位置和纹理坐标属性（值得注意的是：位置坐标系和纹理坐标系不同，下文介绍）；
3. 设置索引数据，索引是用来绘制图形基元时参照；
4. 创建顶点缓冲对象，索引缓冲对象，顶点数组对象，并绑定传值；
5. 链接顶点属性；
6. 创建和绑定纹理对象，加载图片，传递纹理像素值；
7. 让程序进入渲染循环，在循环中绑定顶点数组对象，不断绘制图形基元。

第2点描述的位置坐标系和纹理系不同，具体不同如下图：

![](/images/FFmpeg-Cocos-Creator/xKR8ij.png)

- 位置坐标系原点（0，0）在中心位置，x，y 取值范围是 -1 到 1；
- 纹理坐标系原点（0，0）在左上角位置，x，y取值范围是 0 到 1；

在 Cocos Creator 2.0 版本后，自定义渲染组件，分为三步：

1. 自定义材质（材质负责着色器程序）；
2. 自定义 Assembler （Assembler 负责传递顶点属性）；
3. 设置材质动态参数，如设置纹理，变换平移旋转缩放矩阵等。

第 1 步：着色器程序需要写在 effect 文件中，而 effect 被 material 使用，每个渲染组件，需要挂载 material 属性。由于视频展示，可以理解为图片帧动画渲染，因此可以直接使用 Cocos Creator 提供的 CCSprite 所用的 builtin-2d-sprite 材质。

第 2 步：有了材质后，只需要关心位置坐标和纹理坐标传递，即要自定义 Assembler，可参考官方文档 [自定义 Assembler](https://docs.cocos.com/creator/manual/zh/advanced-topics/custom-render.html#自定义-assembler)。为了效率，直接使用官方源码 https://github.com/cocos-creator/engine/blob/master/cocos2d/core/renderer/assembler-2d.js 改造，值得注意的是，原生端和 Web 端世界坐标计算方式（ updateWorldVerts ）不一样，否则会出现展示位置错乱问题。直接贴代码：

```javascript
export default class CCVideoAssembler extends cc.Assembler {
    constructor () {
        super();
        this.floatsPerVert = 5;
        this.verticesCount = 4;
        this.indicesCount = 6;
        this.uvOffset = 2;
        this.colorOffset = 4;
        this.uv = [0, 1, 1, 1, 0, 0, 1, 0]; // left bottom, right bottom, left top, right top
        this._renderData = new cc.RenderData();
        this._renderData.init(this);
        
        this.initData();
        this.initLocal();
    }

    get verticesFloats () {
        return this.verticesCount * this.floatsPerVert;
    }

    initData () {
        this._renderData.createQuadData(0, this.verticesFloats, this.indicesCount);
    }
    
    initLocal () {
        this._local = [];
        this._local.length = 4;
    }

    updateColor (comp, color) {
        let uintVerts = this._renderData.uintVDatas[0];
        if (!uintVerts) return;
        color = color || comp.node.color._val;
        let floatsPerVert = this.floatsPerVert;
        let colorOffset = this.colorOffset;
        for (let i = colorOffset, l = uintVerts.length; i < l; i += floatsPerVert) {
            uintVerts[i] = color;
        }
    }

    getBuffer () {
        return cc.renderer._handle._meshBuffer;
    }

    updateWorldVerts (comp) {
        let local = this._local;
        let verts = this._renderData.vDatas[0];

        if(CC_JSB){
            let vl = local[0],
            vr = local[2],
            vb = local[1],
            vt = local[3];
            // left bottom
            verts[0] = vl;
            verts[1] = vb;
            // right bottom
            verts[5] = vr;
            verts[6] = vb;
            // left top
            verts[10] = vl;
            verts[11] = vt;
            // right top
            verts[15] = vr;
            verts[16] = vt;
        }else{
            let matrix = comp.node._worldMatrix;
            let matrixm = matrix.m,
                a = matrixm[0], b = matrixm[1], c = matrixm[4], d = matrixm[5],
                tx = matrixm[12], ty = matrixm[13];

            let vl = local[0], vr = local[2],
                vb = local[1], vt = local[3];
            
            let justTranslate = a === 1 && b === 0 && c === 0 && d === 1;

            if (justTranslate) {
                // left bottom
                verts[0] = vl + tx;
                verts[1] = vb + ty;
                // right bottom
                verts[5] = vr + tx;
                verts[6] = vb + ty;
                // left top
                verts[10] = vl + tx;
                verts[11] = vt + ty;
                // right top
                verts[15] = vr + tx;
                verts[16] = vt + ty;
            } else {
                let al = a * vl, ar = a * vr,
                bl = b * vl, br = b * vr,
                cb = c * vb, ct = c * vt,
                db = d * vb, dt = d * vt;

                // left bottom
                verts[0] = al + cb + tx;
                verts[1] = bl + db + ty;
                // right bottom
                verts[5] = ar + cb + tx;
                verts[6] = br + db + ty;
                // left top
                verts[10] = al + ct + tx;
                verts[11] = bl + dt + ty;
                // right top
                verts[15] = ar + ct + tx;
                verts[16] = br + dt + ty;
            }
        }
    }

    fillBuffers (comp, renderer) {
        if (renderer.worldMatDirty) {
            this.updateWorldVerts(comp);
        }

        let renderData = this._renderData;
        let vData = renderData.vDatas[0];
        let iData = renderData.iDatas[0];

        let buffer = this.getBuffer(renderer);
        let offsetInfo = buffer.request(this.verticesCount, this.indicesCount);

        // fill vertices
        let vertexOffset = offsetInfo.byteOffset >> 2,
            vbuf = buffer._vData;

        if (vData.length + vertexOffset > vbuf.length) {
            vbuf.set(vData.subarray(0, vbuf.length - vertexOffset), vertexOffset);
        } else {
            vbuf.set(vData, vertexOffset);
        }

        // fill indices
        let ibuf = buffer._iData,
            indiceOffset = offsetInfo.indiceOffset,
            vertexId = offsetInfo.vertexOffset;
        for (let i = 0, l = iData.length; i < l; i++) {
            ibuf[indiceOffset++] = vertexId + iData[i];
        }
    }

    updateRenderData (comp) {
        if (comp._vertsDirty) {
            this.updateUVs(comp);
            this.updateVerts(comp);
            comp._vertsDirty = false;
        }
    }

    updateUVs (comp) {
        let uv = this.uv;
        let uvOffset = this.uvOffset;
        let floatsPerVert = this.floatsPerVert;
        let verts = this._renderData.vDatas[0];
        for (let i = 0; i < 4; i++) {
            let srcOffset = i * 2;
            let dstOffset = floatsPerVert * i + uvOffset;
            verts[dstOffset] = uv[srcOffset];
            verts[dstOffset + 1] = uv[srcOffset + 1];
        }
    }

    updateVerts (comp) {
        let node = comp.node,
            cw = node.width, ch = node.height,
            appx = node.anchorX * cw, appy = node.anchorY * ch,
            l, b, r, t;
        l = -appx;
        b = -appy;
        r = cw - appx;
        t = ch - appy;

        let local = this._local;
        local[0] = l;
        local[1] = b;
        local[2] = r;
        local[3] = t;
        this.updateWorldVerts(comp);
    }
}
```

 第 3 步，设置材质动态参数，在视频播放器中，需要动态修改的就是纹理数据了，在移动端，ffplay 改造后的 AVPlayer 在播放过程，通过 ITextureRenderer.render(uint8_t) 接口调用到 void Video::setImage(const uint8_t *data) 方法，实际在不断更新纹理数据，代码如下：

```cpp
void Video::setImage(const uint8_t *data) {
    GL_CHECK(glActiveTexture(GL_TEXTURE0));
    GL_CHECK(glBindTexture(GL_TEXTURE_2D, _glID));
    GL_CHECK(
            glTexImage2D(GL_TEXTURE_2D, 0, _glInternalFormat, _width, _height, 0, _glFormat,
                         _glType, data));
    _device->restoreTexture(0);
}
在 Web 端，则是在 CCVideo 渲染组件的每一帧去传递 video 元素，代码如下：
let gl = cc.renderer.device._gl;
this.update = dt => {
    if(this._currentState == VideoState.PLAYING){
        gl.bindTexture(gl.TEXTURE_2D, this.texture._glID);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, this.video);
    }
};
```

至此，视频展示小节完毕。

### 音频播放

在改造音频播放过程之前，查阅了 **ijkplayer** 的音频播放方案，作为现状分析。

**ijkplayer** 在 Android 端有两套方案：AudioTrack 和 OpenSL ES。

1. AudioTrack 属于那种同步写数据的方式，属于 “推” 方案，google 最开始推行的方式，估计比较稳定，由于 AudioTrack 是 Java 接口，C++ 调用需要反射，理论上，对效率有点影响。
2. OpenSL ES 可以做到 “拉” 方案，但 google 在官方文档说过对 OpenSL ES 接口没做太多的兼容，也许不太可靠。

**ijkplayer** 在 iOS 端也是两套方案：AudioUint 和 AudioQueue，由于本人对 iOS 开发不熟，不知道二者区别，因此不做展开。

在 Cocos Creator 音频播放改造中，在 Android 端选择了 google 最新推行的响应延迟极低的 [Google Oboe](https://github.com/google/oboe) 方案，Oboe 是 AAudio 和 OpenSL ES 封装集合，功能更强大，接口更人性化。在 iOS 端选择了 AudioQueue ，要问原因的话，就是 iOS AudioQueue 的接口和 Android Oboe 提供的接口更像...

音频播放模型，属于生产者消费者模型，音频设备在开启状态下，会不断拉取音频解码器生成的音频样本。

音频播放的接口并不复杂，主要用于替换 ffplay 程序中的 SDL 音频相关接口，具体接口代码如下：

```cpp
#ifndef I_AUDIO_DEVICE_H
#define I_AUDIO_DEVICE_H

#include "AudioSpec.h"

class IAudioDevice {

public:

    virtual ~IAudioDevice() {};

    virtual bool open(AudioSpec *wantedSpec) = 0;  // 开启音频设备，AudioSpec 结构体包含拉取回调

    virtual void close() = 0;                      // 关闭音频输出

    virtual void pause() = 0;                      // 暂停音频输出

    virtual void resume() = 0;                     // 恢复音频输出

    AudioSpec spec;
};

#endif //I_AUDIO_DEVICE_H
```

### 优化与扩展

#### 边下边播

边下边播可以说是音视频播放器必备的功能，不但可以节省用户流量，而且可以提高二次打开速度。最常见的边下边播实现方式是在客户端建立代理服务器，只需要对播放器传入的资源路径加以修改，从而达到播放功能和下载功能解耦。不过理论上，建立代理服务器会增加移动设备的内存和电量消耗。

接下来介绍另外一种更简单易用的方案：利用 FFmpeg 提供的协议组合来实现边下边播

在查阅 [FFmpeg 官方协议](https://ffmpeg.org/ffmpeg-protocols.html) 文档时，发现某些协议支持组合使用，如下：

`cache:http://host/resource`

这里在 http 协议前面添加了 cache 协议，即可以使用官方提供的播放过程中缓存观看过的一段，以便跳转使用，由于 cache 协议生成的文件路径问题，导致移动端不适用，此功能也达不到边下边播功能。

但从中可以得到结论：在其他协议前面加入自己协议，就能像钩子一样 hook 住其他协议接口，于是整理一个边下边播的 avcache 协议：

```c
const URLProtocol av_cache_protocol = {
        .name                = "avcache",
        .url_open2           = av_cache_open,
        .url_read            = av_cache_read,
        .url_seek            = av_cache_seek,
        .url_close           = av_cache_close,
        .priv_data_size      = sizeof(AVContext),
        .priv_data_class     = &av_cache_context_class,
};
```

原理就是：在 av_cache_read 方法中，调用其他协议的 read 方法，得到数据后，写入文件并存储下载信息，并把数据返回给播放器。

#### libyuv 替换 swscale

YUV（[wikipedia](https://zh.wikipedia.org/wiki/YUV)），是一种颜色编码方法。为了节省带宽，大多数 YUV 格式平均使用的每像素位数都少于24位，因此一般视频都是用 YUV 颜色编码。YUV 由分为两种格式，分别是紧缩格式和平面格式。其中平面格式将 Y、U、V 的三个分量分别存放在不同的矩阵中。

根据上文，如果让片段着色器直接支持 YUV 纹理渲染，不同格式下，片段着色器所需要的 sampler2D 纹理采样器数量也不同，因此管理起来相当不便。最简单的方式，就是把 YUV 颜色编码转成 RGB24 颜色编码，因此需要用到 FFmpeg 提供的 swscale。

但在使用 swscale （已开启 FFmpeg 编译选项 neon 优化）进行颜色编码转换后，就可以发现 swscale 在移动端效率低下，使用小米 Mix 3 设备，1280x720 分辨率的视频，像素格式从 AV_PIX_FMT_YUV420P 转成 AV_PIX_FMT_RGB24，缩放按照二次线性采样，平均耗时高达 16 毫秒，而且导致 CPU 占用率相当高。数据截图待补：

经过 google 一番搜索，找到了 google 的 libyuv 替代方案

开源项目：https://chromium.googlesource.com/libyuv/libyuv/

官方优化说明：

- Optimized for SSSE3/AVX2 on x86/x64；
- Optimized for Neon on Arm；
- Optimized for MSA on Mips。

使用 libyuv 进行像素格式转换后，使用小米 Mix 3 设备，1280x720 分辨率的视频，像素格式从 AV_PIX_FMT_YUV420P 转成 AV_PIX_FMT_RGB24，缩放按照二次线性采样，平均耗时 8 毫秒，相对 swscale 降低了一半。数据截图待补：

#### Android asset 协议

由于 Cocos Creator 本地音视频资源在 Android 端会打包到 asset 目录下，在 asset 目录下的资源需要使用 AssetManager 打开，因此需要支持 Android asset 协议，具体协议声明如下：

```c
const URLProtocol asset_protocol = {
        .name                = "asset",
        .url_open2           = asset_open,
        .url_read            = asset_read,
        .url_seek            = asset_seek,
        .url_close           = asset_close,
        .priv_data_size      = sizeof(AssetContext),
        .priv_data_class     = &asset_context_class,
};
```

## 成果展示


{% raw %}

<div style="position: relative; width: 100%; height: 0; padding-bottom: 75%;">
    <iframe src="//player.bilibili.com/player.html?aid=288261417&bvid=BV1kf4y1e7Ew&cid=272365385&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" style="position: absolute; width: 100%; height: 100%; left: 0; top: 0;">
    </iframe>
</div>

{% endraw %}


## 参考文档

1. FFmpeg: https://ffmpeg.org/
2. Cocos Creator 自定义 Assembler: https://docs.cocos.com/creator/manual/zh/advanced-topics/custom-render.html#
3. Cocos Creator JSB 绑定：https://docs.cocos.com/creator/manual/zh/advanced-topics/JSB2.0-learning.html
4. Android Oboe: https://github.com/google/oboe/blob/master/docs/FullGuide.md
5. Google libyuv: https://chromium.googlesource.com/libyuv/libyuv/+/HEAD/docs/getting_started.md
6. LearnOpenGL CN: https://learnopengl-cn.github.io/
7. WebGL: https://developer.mozilla.org/zh-CN/docs/Web/API/WebGL_API/Tutorial/Animating_textures_in_WebGL
8. ijkplayer: https://github.com/bilibili/ijkplayer
9. 等等...