---
title: 前端构建秘籍
date: "2019-05-29 14:46:35"
tags: webpack
categories: 前端
author: "刘华(edenhliu)"
---

## 前言

随着前端构架工具的不断发展，提供了很多提高我们的开发体验和开发效率的能力，同时构建已经成为前端技术栈中常见的技术。

webpack 也是众多构建工具中崭露头角一员，早期的 webpack 配置复杂难懂，随着其发展，相关配置也不断简化，性能也不断提高，但是对于深入使用的开发人员，通常它的默认配置并不适用于业务开发，需要针对自己业务调整适配。

你对 webpack 了解多少？如何针对业务集成最佳配置？如何优化开发体验？如何开足马力，实现极速的 webpack 的构建性能 🚀？又会有哪些坑 💣？本文带你解答这些问题 🔭。

*本文涉及到的所有代码片段的完整代码请参考[a8k仓库](https://github.com/hxfdarling/a8k)*
![-w1078](/images/frontend-packing-tips/15466012648067/15469465809799.jpg)

## 一、webpack 关键配置项

> 对构建有所了解的，可直接略过本节

此处不会深入介绍相关配置，更多的详细说明与配置参见官方文档，稍作介绍关键配置项铺垫后面内容。

### entry

webpack 查找依赖的入口文件配置，入口文件可以有多个。

**单页面应用入口配置**
通常做法配置：vendor.js 第三方依赖库，polyfill.js 特性填充库，index.js 单页面应用入口文件

```js
// 导出配置
module.exports = {
  entry: {
    vendor: './src/vendor.js',
    polyfill: './src/polyfill.js',
    index: './src/index.js',
  },
};
```

**多页面应用入口配置**

和单页面应用类似，但不同页面会不同有入口文件，这种情况高效的做法就不是直接写死在 entry 里面了，而是通过生成 webpack.config 时，扫描指定目录确定每个页面的入口文件以及所有的页面。

下面举个例子

> 假定你的页面都放置在 src/pages 目录下面，并且你的每个页面单独一个目录，并且其中有 index.html 和 index.jsx

```js
const path = require('path');
const fs = require('fs');
// 处理公共entry
const commonEntry = ['./src/vendor.js', './src/polyfill.js'];
// 页面目录
const PAGES_DIR = './src/pages/';
const entry = {};
// 遍历页面目录
const getPages = () => {
  return fs.readdirSync(PAGES_DIR).filter(item => {
    let filepath = path.join(PAGES_DIR, item, 'index.js');
    if (!fs.existsSync(filepath)) {
      filepath = `${filepath}x`; // jsx
    }
    if (!fs.existsSync(filepath)) {
      return false;
    }
    return true;
  });
};
getPages(options).forEach(file => {
  const name = path.basename(file);
  // 加入页面需要的公共入口
  entry[name] = [...commonEntry, `${PAGES_DIR}/${file}/index`];
});
// 导出配置
module.exports = {
  entry,
};
```

**入口 boundle 如何插入对应的 html 中？**

我们通常需要这个插件`HtmlWebpackPlugin`自动处理，具体代码如下：

```js
const plugins = [];
if (mode === 'single') {
  // 单页面只需要一次HtmlWebpackPlugin
  plugins.push(
    new HtmlWebpackPlugin({
      minify: false,
      filename: 'index.html',
      template: './src/index.html',
    })
  );
}
if (mode === 'multi') {
  // 多页面遍历目录，使用目录下面的html文件
  // 不同页面的配置不同，每个页面都单独配置一个html
  // 所有页面的公共部分可以抽离后，通过模版引擎编译处理
  // 具体的方式后面部分loader中提到
  const files = getPages(options);
  files.forEach(file => {
    const name = path.basename(file);
    file = `${PAGES_DIR}/${file}/index.html`;
    // 添加runtime脚本，和页面入口脚本
    const chunks = [`runtime~${name}`, name];
    plugins.push(
      new HtmlWebpackPlugin({
        minify: false,
        filename: `${name}.html`,
        template: file,
        chunks,
      })
    );
  });
}
// 导出配置
module.exports = {
  plugins,
};
```

### output

该项配置输出的 bundle 的相关信息，比较常用的配置如下：

```js
{
  output:{
   // name是你配置的entry中key名称，或者优化后chunk的名称
   // hash是表示bundle文件名添加文件内容hash值，以便于实现浏览器持久化缓存支持
   filename: '[name].[hash].js',
   // 在script标签上添加crossOrigin,以便于支持跨域脚本的错误堆栈捕获
   crossOriginLoading:'anonymous',
   //静态资源路径，指的是输出到html中的资源路径前缀
   publicPath:'https://7.ur.cn/fudao/pc/',
   path: './dist/',//文件输出路径
  }
}
```

### resolve

该项配置主要用于解析模块依赖的自定义项, 比较常规的配置项如下，modules用于加速绝对路径查找效率，alias可以用户自定义模块查找路径。

```js
resolve: {
    modules: [
        path.resolve(__dirname, 'src'), 
        path.resolve(__dirname,'node_modules'),
    ],
    alias: {
      components: path.resolve(__dirname, '/src/components'),
    },
}
```

**扩展**
如果你使用了绝对路径后，可能就发现vscode智能代码导航就失效了，别慌！请在想目录下面配置`jsconfig.json`文件解决这个问题，配置和上面对应:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "src/*": ["./src/*"],
      "components/*": ["./src/components/*"],
      "assets/*": ["./src/assets/*"],
      "pages/*": ["./src/pages/*"]
    }
  },
  "include": ["./src/**/*"]
}
```
这样，你就可以愉快的使用vscode的智能代码提示和导航了！

### module

该项主要配置就是rules了，rules中配置对于不同资源的处理器，是其核心之一，这里简单添加一个示例代码
```js
module: {
  // 这些库都是不依赖其它库的库 不需要解析他们可以加快编译速度
  // 通常可以将那些大型的库且已经编译好的库排除，减少webpack对其解析耗时
  noParse: /node_modules\/(moment|chart\.js)/,
  rules: [
    {
      test: /\.jsx?$/,
      use: resolve('babel-loader'),
      // 需要被这个loader处理的资源
      include: [
        path.resolve(projectDir, 'src'),
        path.resolve(projectDir, 'node_modules/@tencent'),
      ].filter(Boolean),
      // 忽略哪些压缩的文件
      exclude: [/(.|_)min\.js$/],
    }
  ]
```

### optimization

该顶配项中最重要最常用的是:`splitChunks`,`minimizer`

`minimizer` 可以自己配置输出的文件压缩插件，js压缩我们可以使用webpack集成的uglifyjs，也可以使用Terser，Terser支持es6代码的压缩,同时支持多进程压缩；css压缩我们可以使用`optimize-css-assets-webpack-plugin`压缩，它使用cssnano作为处理引擎，帮助我们去除重复样式.

`splitChunks`是webpack4.x推出的重磅功能，优化的公共chunk提取策略，更高效的提取公共模块，在后面性能优化中会详细说明其使用方法。

### plugin

plugin 可以介入整个构建过程任何阶段。例如：报告构建耗时、修改输出代码支持主域重试、添加构建进度报告、代码压缩、资源替换等很多能力都在这里实现。

plugin不展开讨论，因为插件太多了。对于项目需要自己实现插件的，需要注意一点，当你使用插件对输出结果处理时，应当在文件输出到磁盘之前处理，我们以前的构建中主域重试插件就踩了这个坈，导致最终构建的代码出现错误，原因是该插件直接修改磁盘上面的文件，两次构建同时启动，结束时两次构建的插件都修改了磁盘上同一个文件，最终导致bug，并且导致我们需要强行清理发布环境代码才恢复正常发布。

## 二、开发体验优化

舒适的开发体验，有助于提高我们的开发效率，优化开发体验也至关重要

### 组件热刷新、CSS热刷新

自从webpack推出热刷新后，前端开发者在开环境下体验大幅提高。
没有热刷新能力，我们修改一个组件后

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 等待构建
op1=>operation: 刷新页面
op2=>operation: 点击跳转到对应组件
op3=>operation: 等待数据刷新
op4=>operation: 验证修改

st->op(right)->op1(right)->op2(right)->op3->op4->e
```

加入热构建后：

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 等待构建
op1=>operation: 自动下发变化组件
op2=>operation: 自动重新渲染组件(状态保持)
op4=>operation: 验证修改

st->op(right)->op1(right)->op2->op4->e
```

主要看一下我们业务基于React技术栈，如何在构建中接入热刷新。
1. 无论什么技术栈，都需要在dev模式下加上`webpack.HotModuleReplacementPlugin`插件
2. 在所有entry中插入`require.resolve('../utils/webpackHotDevClient')`,`webpackHotDevClient`这份代码是由react官方的create-react-app提供的
3. 在`webpack-dev-server`模块的启动参数中添加`hot:true`
4. 在你需要热加载的js文件中添加以下代码(这段代码在构建生产包会自动删除):

    ```js
    if (process.env.NODE_ENV==='development' && module.hot) {
        module.hot.accept()
    }
    ```
    
*注：也可以使用[react-hot-loader](https://github.com/gaearon/react-hot-loader)来实现，具体参考官方文档*

### SSR热调试

辅导的H5/PC项目都有部分页面支持直出，以前直出调试方式是如下流程所示：

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 修改代码
op1=>inputoutput: 构建静态资源(数十秒)
op2=>inputoutput: 重新启动server
op3=>operation: 刷新页面
op4=>operation: 验证修改
cond=>condition: 验证通过?
st->op->op1->op2->op3->op4(right)->cond(yes)->e
cond(no)->op
```
这种调试流程太长，每一次修改都需要重新构建静态资源，并重启node服务，非常耗时，其次直出模式下，非直出的页面将无法正常访问，整个流程无法走通。

因此, 提出了新的解决方案, 采用 `webpack watch+nodemon` 结合的模式实现对SSR热调试的支持。node 服务需要的html/js通过webpack插件动态输出，当nodemon检测到变化后将自动重启，html文件中的静态资源全部替换为dev模式下的资源，并保持socket连接自动更新页面。

实现热调试后，调试流程大幅缩短，和普通非直出模式调试体验保持一致。在`a8k`中通过`k dev -s`命令即可开启ssr调试模式。下面是SSR热调试的流程图：

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 修改代码
op1=>inputoutput: 等待热构建(数秒)
op2=>operation: 自动刷新
op3=>operation: 验证修改
cond=>condition: 验证通过?
st->op->op1->op2->op3(right)->cond(yes)->e
cond(no)->op
```


### style调试体验

**问题：**

给`style-loader` 开启sourceMap后, sourceMap是内联在style文件中的，需要通过link导入，这种方式是通过JavaScript生成blob后丢个link标签解析。之后我们可以在dev工具中直接看到每个样式所在的源文件位置，方便快速的调试样式。但也同样引起一个问题FOUC（页面加载后闪烁），可参见这个[ssue](https://github.com/webpack-contrib/style-loader/issues/107)

**解决方法：**

添加`singleton: true`参数可解决这个问题，但是sourceMap就不能定位到源文件了，而是合并后的文件中的位置，二者不可兼得。所以在`a8k`工具中提供了可选项，默认开启`singleton:true`,通过`k dev -c`可开启cssSourceMap映射

## 三、性能优化

### node_modules缓存

> 辅导大多数项目node_modules依赖数量都非常惊人，辅导PC项目剔除构建相关依赖后，依赖包都1883个，依赖包的安装耗时也就大幅增加，因此减少依赖包安装耗时,对构建整体提升非常重要，方法那就是缓存。

**JB系统编译**
每次编译都会启动一个新的目录，这导致项目依赖的众多node_modules无法缓存，每次编译重新安装耗时非常长，针对JB的编译，我开发了@tencent/im-build模块自动缓存项目依赖的node_modules，大幅提升了编译性能。

**OCI编译系统**
OCI中不需要额外的插件支持，该系统本身已经可以通过配置实现部分目录缓存，二次利用的能力，使用方法如下：

1. 在项目根目录添加`.orange-cache.cache`文件，并添加你需要缓存的目录
    
    ```
    /node_modules
    /fudao_qq_com_pc_imt
    ```

2. 修改`.orange-ci.yml`配置，添加缓存配置文件路径

    ```yml
      push:
        - cacheFrom: .orange-ci.cache
        #其它配置省略
    ```

#### 优化效果

**优化前**
![](/images/frontend-packing-tips/15336395035110/15341352482636.jpg)

**优化后**
![](/images/frontend-packing-tips/15336395035110/15341352771515.jpg)

### 构建中间结果缓存

中间结果缓存优化同样能大幅提升构建性能，对模块的编译本身就是CPU密集型任务。通常来说每次构建并非所有模块都需要被重新处理，可以只考虑处理那些文件内容有变化的模块，那么文件内容没有变化的模块就可以从缓存中获取，通常通过文件内容hash值作为缓存文件的名称，这就是“热构建”。

在webpack中，能够被缓存的内容有：loader处理结果、plugin处理结果、输出文件结果。下面详细说明不同资源不同阶段的缓存方式。

#### 1. babel-loader缓存,通过cacheDirectory开启缓存

```js
test: /\.jsx?$/,
use: [
  {
    loader: resolve('babel-loader'),
    options: {
      babelrc: false,
      // cacheDirectory 缓存babel编译结果加快重新编译速度
      cacheDirectory: path.resolve(options.cache, 'babel-loader'),
      presets: [[require('babel-preset-imt'), { isSSR }]],
    },
  },
],
```

#### 2. eslint-loader缓存,通过cache选项指定缓存路径

```js
test: /\.(js|mjs|jsx)$/,
enforce: 'pre',
use: [
    {
      options: {
        cache: path.resolve(options.cache, 'eslint-loader'),
      },
      loader: require.resolve('eslint-loader'),
    },
],
```
*eslint-loader通常只需要在开发模式下开启，方便及时的提醒开发者，存在eslint错误，及时修复*

#### 3. css/scss缓存

css-loader/sass-loader/postcss-loader本身并没有提供缓存机制，这里需要用到cache-loader辅助我们实现对css/scss的构建结果缓存，具体使用方式如下：

```js
{
  loader: resolve('cache-loader'),
  options: { cacheDirectory: path.join(cache, 'cache-loader-css') },
},
{
  loader: resolve('css-loader'),
  options: {
    importLoaders: 2,
    sourceMap,
  },
},
...由于篇幅原因，这里不展示其它更多loader
```
*只需要将该loader添加到这个loader的最头部即可，该loader不仅可以对于css缓存*

#### 4. 输出代码压缩缓存，JS压缩引擎多进程处理

JS代码压缩我们采用了`TerserPlugin`插件，具体配置如下：

```js
{
    // 设置缓存目录
    cache: path.resolve(cache, 'terser-webpack-plugin'),
    parallel: true,// 开启多进程压缩
    sourceMap,
    terserOptions: {
      compress: {
        // 删除所有的 `console` 语句
        drop_console: true,
      },
    },
}
```
#### 5. CI系统固定缓存目录

上面在不同的plugin和loader上面配置了cache目录，对于CI系统来说你需要将cache目录路径固定，以便于重复使用缓存内容，使用方式：JB就配置`/tmp/xxx`目录,OCI系统可配置在项目目录。

*⚠️注意：由于使用了缓存，当你修改你的编译配置后，需要立即清理缓存结果，最好的做法是在构建工具中自动检测相关配置是否有变化，自动清理缓存*

### 其它优化手段

#### 1. 指定绝对路径模块查找路径，加速模块查找

```js
resolve: {
  //加快搜索速度
  modules: [
    'node_modules', 
    path.resolve(projectDir, 'src'), 
    path.resolve(projectDir, 'node_modules')
  ],
},
```

#### 2. 过滤不需要做任何处理的库

```js
module: {
  // 这些库都是不依赖其它库的库 不需要解析他们可以加快编译速度
  noParse: /node_modules\/(moment|chart\.js)/,
}
```

#### 3. 缩小babel处理范围,避免处理已经压缩的代码

```js
// 指处理指定目录的文件
include: [
    path.resolve(projectDir, 'src'),
    path.resolve(projectDir, 'node_modules/@tencent'),
].filter(Boolean),
// 忽略哪些压缩的文件
exclude: [/(.|_)min\.js$/],
```

#### 4. lodash库按需倒入优化，减少无用代码

我们在使用lodash库是，通常只会用到其中非常少的function，但是像下面这段代码，将会导致lodash全部被打入最终的bundle中。
```js
import _ from 'lodash'
_.difference(1, 2)
```
这种情况幸好有插件可以帮我们优化，通过lodashPlugin即可自动处理lodash的按需引用

使用方法如下：
```js
const LodashPlugin = require('lodash-webpack-plugin');
plugins:[
    // 支持lodash包 按需引用
    new LodashPlugin(),
]
```
加入这个plugin后，上面的代码自动处理为如下代码：
```js
import difference from 'lodash/difference';
difference([1, 2], [1, 3]);
```

*注意：导入代码方式必须使用import，不能使用require*

#### 5. 针对服务端渲染代码，我们可以剔除node_modules，从而大幅减少服务端代码生成耗时

通过`webpack-node-externals`插件实现这一点，具体使用方法如下：
```js
const nodeExternals = require('webpack-node-externals');
module.export={
// 省略其它配置
 externals: [
  nodeExternals({
    // 注意如果存在src下面其他目录的绝对引用，都需要添加到这里
    whitelist: [
    /^components/, /^assets/, /^pages/, /^@tencent/, /\.(scss|css)$/
    ],
  }),
 ],
// 省略其它配置
}
```

#### 6. webpack4.x的鼎力之作之splitChunks

在webpack4之前，我们处理公共模块的方式都是使用`CommonsChunkPlugin`，然后该插件的让开发这配置繁琐，并且公共代码的抽离，不够彻底和细致，因此新的`splitChunks`改进了这些能力。使用的正确姿势如下：

```js
splitChunks: {
      chunks: 'all',
      minSize: 10000, // 提高缓存利用率，这需要在http2/spdy
      maxSize: 0,//没有限制
      minChunks: 3,// 共享最少的chunk数，使用次数超过这个值才会被提取
      maxAsyncRequests: 5,//最多的异步chunk数
      maxInitialRequests: 5,// 最多的同步chunks数
      automaticNameDelimiter: '~',// 多页面共用chunk命名分隔符
      name: true,
      cacheGroups: {// 声明的公共chunk
        vendor: {
         // 过滤需要打入的模块
          test: module => {
            if (module.resource) {
              const include = [/[\\/]node_modules[\\/]/].every(reg => {
                return reg.test(module.resource);
              });
              const exclude = [/[\\/]node_modules[\\/](react|redux|antd)/].some(reg => {
                return reg.test(module.resource);
              });
              return include && !exclude;
            }
            return false;
          },
          name: 'vendor',
          priority: 50,// 确定模块打入的优先级
          reuseExistingChunk: true,// 使用复用已经存在的模块
        },
        react: {
          test({ resource }) {
            return /[\\/]node_modules[\\/](react|redux)/.test(resource);
          },
          name: 'react',
          priority: 20,
          reuseExistingChunk: true,
        },
        antd: {
          test: /[\\/]node_modules[\\/]antd/,
          name: 'antd',
          priority: 15,
          reuseExistingChunk: true,
        },
      },
    },
```

简要解释上面这段配置

1. 将node_modules共用部分打入vendor.js bundle中；
2. 将react全家桶打入react.js bundle中；
3. 如果项目依赖了antd，那么将antd打入单独的bundle中；
4. 最后剩下的业务模块超过3次引用的公共模块，将自动提取公共块


### 优化效果

做了这么多优化，下面是基于模块超过2.5k的辅导h5项目，构建耗时对比，感受一下效果

优化前：热构建需要40s
![-w642](/images/frontend-packing-tips/15435516132696/15435701717876.jpg)

优化后：只需要20s
![-w601](/images/frontend-packing-tips/15435516132696/15435516153134.jpg)


## 四、收敛配置集成最佳实践

构建的配置和优化的工作并不小，将最佳实践收敛和集成为独立的模块，在不同项目中复用，可以大幅减少构建维护工作，以及后续升级优化工作难度。

IMWeb团队的项目目前也独立维护一套基于React技术栈的构建最佳实践工具`a8k`，在所有的项目中不会在看到复杂多样的webpack配置，以及各种花样的前置、后置脚本。各项目仅需要简单的关键配置即可快速接入该构建工具，享受其带来的开发体验提升，和构建性能提升。

## 五、其他经验

### 关于node-sass

用过node-sass的童鞋应该遇到过，安装node-sass遇到各种编译错误、二进制文件下载错误、甚至文件写入权限错误等等😟。也有各种骚操作解决这个问题，但终归不能一劳永逸。

于是就出现想通过postcss插件去兼容sass语法，虽然通过插件能够兼容部分语法，但是想要在已经有一定量的业务代码中，替换node-sass的风险是非常高的，本人亲自测试各种坑💣

当然也有其他途径解决这个问题，不仅让你使用完整的sass语法，同时也免去各种安装node-sass的问题，官方的sass-loader其实已经提供了dart-sass解析模块的支持具体参见[文档](https://github.com/webpack-contrib/sass-loader#examples)，可能有人担心dart-sass的js模块性能不高，本人亲测在我们项目中2000+的模块中，dart-sass的编译性能并没有明显下降的感觉，同时我们使用使用了缓存能力，通常只变异哪些变化的资源。

具体的配置入下:

```js
{
  loader: resolve('sass-loader'),
  options: {
    // 安装dart-sass模块：npm i -D sass
    implementation: require('sass'),
    includePaths: [
      // 支持绝对路径查找
      path.resolve(projectDir, 'src'),
    ],
    sourceMap,
  },
},
```

node-sass 变量使用问题
我在H5中发现很多这种语法的代码，但是实际上没有生效，构建后，并没有替换为变量的值。

![-w615](/images/frontend-packing-tips/15427990043309/15427990093549.jpg)

编译后：
![-w937](/images/frontend-packing-tips/15427990043309/15427991423306.jpg)

解决方法如下：
![-w838](/images/frontend-packing-tips/15427990043309/15427990336740.jpg)


### 关于 postcss

个人觉得postcss是css预处理器的未来，现在的postcss对于css就像babel对于JavaScript。postcss通过插件支持未来的css特性，于此同时你还可以自定义插件实现想要的特性。但其他的less、sass这种预处理器，就难以介入它的处理过程，只能按照它既定的规则处理。因此对于全新的项目建议直接使用`postcss+postcss-preset-env` 使用最新的css语法特性，同时以便于在未来浏览器全面支持相关特性后，快速接入支持。

💣如果你使用了`css-loader`的import能力，同时有使用了`post-css-import`插件的import能力，两个插件会存在冲突，不建议同时使用！

如果使用了`postcss-custom-properties`,需要注意在8.x版本中存在一个bug,无法解析如下语法:
```scss
:root{
  --green: var(--customGreen, #08cb6a);
   // 8.x无法正确处理该语法
  --primary: var(--customPrimary, var(--green));
}
.test {
  background: color(var(--primary) shade(5%));
  // 上面面这句将会被转换为如下代码，最终导致浏览器无法解析该语法
  background: var(--green);
  background: var(--primary);
  // 我们期望转换为
  background: #08cb6a;
}
```
解决方法：禁用 postcss-preset-env 中的custom-properties,安装6.x版本的custom-properties，单独添加该插件。

### 关于缓存

如果在开发模式下面启用了`eslint-loader`对`jsx?`文件校验，并且启动了其缓存能力，当修改eslint校验规则，你需要清理缓存文件并且重新启动构建，否则规则修改不会生效！如果使用`a8k`工具构建，可以使用`k clean`命令自动处理处理。

### 主域重试

篇幅太长不详细介绍了，有兴趣的可以在这里看到相关源代码[webpack-retry-load-plugin](https://github.com/hxfdarling/webpack-retry-load-plugin), 后续输入相关文章介绍如何实现CSS/JS同步异步代码重试
