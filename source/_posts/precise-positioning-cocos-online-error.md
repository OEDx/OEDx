---
title: 精确定位 Cocos 线上报错
date: 2019-01-14 17:01:35
tags: Cocos
categories: 客户端
author: "[张韩(khanzhang)](http://khanzhang.cn)"
---

![图片源自 Zoommy](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/2019-01-14-242430%20-1-.jpg)
在 Cocos-JS 的项目中，编译时往往采用 加密 + 压缩 的方式。最终 CocosCreator 编译出的代码文件都是无法直接阅读的，代码文件有 jsb_polyfill.jsc、project.jsc、setting.jsc。强行打开如下图：
![乱码](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/2019-01-14-024003.png)
应用上线后，如有线上 JS 报错，很多情况也是无法直接定位到问题的，错误中指明的只有在 jsb_polyfill.js 文件的多少行有问题，而无法还原到实际的代码文件。比如：
![线上报错](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/2019-01-14-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-01-14%20%E4%B8%8A%E5%8D%8810.44.06.png)
对于此类问题，可以通过 Cocos-JS 加解密工具来帮助我们更加精确的定位问题，其代码地址为 [cocos-jsc-endecryptor](https://github.com/CSIGer/cocos-jsc-endecryptor)。

## 使用 Cocos-JS 加解密工具来定位问题代码行

> 这里以线下 Cocos 编译的 project.jsc 为例

1. 把上述工具的代码拉到本地
2. 终端执行如下命令即可将加密编译出的代码文件反编译出来，以 project.jsc 文件为例，你需要替换括号中内容为你实际的路径和密钥：

    ```shell
    {PATH}/cocos-jsc-endecryptor/edc.py decrypt -k {COCOS_PROJECT_ENCRYT_KEY} -p {COCOS_PROJECT_PATH}/build/jsb-default/src/project.jsc
    ```

    你也可以在 VSCode 中配置一项任务，之后直接在 VSCode 中运行该任务即可编译出 project.jsc 来定位问题：

    ```json
    {
        "label": "decryABCmouse",
        "group": {
            "kind": "build",
            "isDefault": true
        },
        "type": "shell",
        "command": "{PATH}/cocos-jsc-endecryptor/edc.py decrypt -k {COCOS_PROJECT_ENCRYT_KEY} -p {COCOS_PROJECT_PATH}/build/jsb-default/src/project.jsc ; open {COCOS_PROJECT_PATH}/decryptOutput"
    }
    ```

    > 注：VS Code 中配置任务可以在 `Cmd + Shift + P` 呼出命令面板后，输入 `tasks configure task`，然后根据提示新增任务，之后可以在命令面板输入 `tasks run task`，运行刚刚创建的命令。

3. 打开 project.jsc 解密后的文件 decry.js，使用 Ctrl + G 快捷键呼出快速跳转代码行窗口，输入线上报错中的行数，如上图报错中的 16661。即可快速跳转到出错行对应的代码：
    ![跳转至对应行](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/2019-01-14-031132.png)
    然后即可看到具体的代码，而非乱码。之后即可根据这里更加具体的代码来定位问题：
    ![具体代码行](https://blog-pic-1251295613.cos.ap-guangzhou.myqcloud.com/2019-01-14-031647.png)

> 注：这里以线下 Cocos 编译的 project.jsc 为例，你也可以反编译对应的 jsb_polyfill.jsc 文件。如果是编译线上的文件，可以在对应的 apk 中找到。
