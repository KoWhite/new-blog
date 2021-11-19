---
title: webpack构建速度及体积优化
date: 2021-11-19 11:54:42
tags: webpack
categories: 前端
---

## 分析（初级）-使用webpack内置的stats

{% blockquote %}
stats: 构建的统计信息

缺点：颗粒度太粗，看不出问题在哪
{% endblockquote %}

### stats 使用

`package.json`中使用`stats`

```javaScript
"scripts": {
    "build:stats": "webpack --env production --json > stats.json",
    ...
}
```

## 分析（速度）-使用 [speed-measure-webpack-plugin](https://github.com/stephencookdev/speed-measure-webpack-plugin)

{% blockquote %}

1. 分析整个打包总耗时

2. 每个插件和loader的耗时情况

{% endblockquote %}

### 插件使用

1. 安装

    ```javaScript
    npm i --save-dev speed-measure-webpack-plugin
    ```

    or

    ```javaScript
    yarn add speed-measure-webpack-plugin --save-dev
    ```

2. 接入

    在webpack配置文件中引入`speed-measure-webpack-plugin`;

    ```javaScript
    const speedMeasureWebpackPlugin = require('speed-measure-webpack-plugin');
    ...
    const smp = new speedMeasureWebpackPlugin();
    ...
    module.exports = smp.wrap({
        ...
    });
    ```

    像上面的例子一样，使用插件中的`wrap`方法将内容包裹起来，在打包完成之后就能看到模块间的时间。

<a data-fancybox title="速度分析" href="https://img-blog.csdnimg.cn/2020042710203448.png">![速度分析](https://img-blog.csdnimg.cn/2020042710203448.png)</a>

## 分析（体积）-使用 [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)

{% blockquote %}

1. 依赖的第三方模块文件大小

2. 业务里面的组件代码大小

{% endblockquote %}

1. 安装

    ```javaScript
    npm i webpack-bundle-analyzer -D
    ```

    or

    ```javaScript
    yarn add webpack-bundle-analyzer -D
    ```

2. 接入

    在webpack配置文件中引入`webpack-bundle-analyzer`;

    ```javaScript
    const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
    ...
    module.exports = smp.wrap({
        ...
        plugins: [
            ...
            new BundleAnalyzerPlugin ()
        ]
    });
    ```

在构建完成之后可以访问本地8888端口（默认可修改），查看当前项目文件的体积关系

## 优化（速度）-使用 高版本的 webpack 和 Node.js

## 优化（速度）-多进程/多实例构建

{% blockquote %}

1. thread-loader

2. parallel-webpack

3. HappyPack

{% endblockquote %}

### 使用 [HappyPack](https://github.com/amireh/happypack) 解析资源

{% blockquote %}
每次 webpack 解析一个模块，HappyPack 会将它及它的依赖分配给 worker 线程中

{% endblockquote %}

### 使用 [thread-loader](https://github.com/webpack-contrib/thread-loader) 解析资源

{% blockquote %}
每次 webpack 解析一个模块，thread-loader 会将它及它的依赖分配给worker线程中
{% endblockquote %}

## 优化（体积）-多进程/多实例：并行压缩

1. 使用 [parallel-uglify-plugin](https://github.com/gdborton/webpack-parallel-uglify-plugin) 插件

2. [uglifyjs-webpack-plugin](https://github.com/webpack-contrib/uglifyjs-webpack-plugin) 开启 `parallel` 参数 （webpack 4 版本之前）

3. [terser-webpack-plugin](https://github.com/webpack-contrib/terser-webpack-plugin) 开启 `parallel` 参数 （webpack 4 版本之后）

## 进一步分包

### 设置 Externals (初级)

{% blockquote %}
将 react、react-dom 基础包通过 cdn 引入，不打入 bundle 中

方法: 使用 html-webpack-externals-plugin
{% endblockquote %}

### 预编译资源模块 (高级)

{% blockquote %}

思路：将 react、react-dom、redux、react-redux 基础包和业务基础包打包成一个文件

方法：使用 DLLPlugin 进行分包， DIIReferencePlugin 对 manifest.json 引用

{% endblockquote %}

1. 创建`webpack.dll.js`文件

    ```javaScript
    const path = require('path');
    const webpack = require('webpack');

    module.exports = {
        entry: {
            // 基础包
            library: [
                'react',
                'react-dom',
            ]
            // 如果还有其他类可以在这里加一个键和对象
        },
        output: {
            filename: '[name]_[chunkhash].dll.js',
            path: path.join(__dirname, 'build/library'),
            library: '[name]'
        },
        plugins: [
            new webpack.DllPlugin({
                name: '[name]_[hash]',
                path: path.join(__dirname, 'build/library/[name].json')
            })
        ]
    }
    ```

    文件用于分包，将一些基础包进行分离

2. 引入分离包

    首先可以在`package.json`中的`scripts`添加一项

    ```javaScript
        "scripts": {
            "dll": "webpack --config webpack.dll.js"
        }
    ```

    然后在打包配置文件中增加：

    ```javaScript
        const webpack = require('webpack');
        ...
        module.exports = {
            ...
            plugins: [
                ...
                new webpack.DllReferencePlugin({
                    manifest: require('./build/library/library.json')
                })
            ]
        };
    ```

## 利用缓存提高二次构建速度

{% blockquote %}

1. babel-loader 开启缓存

2. terser-webpack-plugin 开启缓存

    ```javaScript
        optimization: {
            minimizer: [
                new TerserPlugin({
                    parallel: true,
                    cache: true
                })
            ],
        }
    ```

3. 使用 cache-loader 或者 hard-source-webpack-plugin

{% endblockquote %}

## 缩小构建目标

### 减少文件搜索范围

1. 优化 resolve.modules 配置 （减少模块搜索层级）

2. 优化 resolve.mainFields 配置

3. 优化 resolve.extensions 配置

4. 合理使用 alias

## 使用 Tree Shaking 擦除无用的JavaScript和CSS

### 无用的CSS如何删除掉

1. PurifyCSS: 遍历代码，识别已经用到的 CSS class

    如何在 webpack 中使用 PurifyCSS

    {% blockquote %}
        使用 `purgecss-webpack-plugin` 和 `mini-css-extract-plugin` 配合使用
    {% endblockquote %}

    (1) 安装

    ```javaScript
        npm i purgecss-webpack-plugin -D
    ```

    or

    ```javaScript
        yarn add purgecss-webpack-plugin -D
    ```

    在webpack配置文件中使用：

    ```javaScript
    const PurgecssPlugin = require('purgecss-webpack-plugin');
    ...
    const PATHS = {
        src: path.join(__dirname, 'src')
    };
    ...
    plugins: [
        new PurgecssPlugin ({
            // 这里的glob类似我们之前那种多页面打包的方式获取对应文件
            paths: glob.sync(`${PATHS.src}/**/*`, { nodir: true }),
        })
    ]
    ```

    [purgecss-webpack-plugin](https://github.com/FullHuman/purgecss/tree/master/packages/purgecss-webpack-plugin)

2. uncss: HTML 需要通过 jsdom 加载，所有的样式通过 PostCSS 解析，通过 document.querySelector 来识别在 html 文件里面不存在的选择器

## 使用webpack进行图片压缩

{% blockquote %}
要求：基于Node库的 `imagemin` 或者 `tinypng` API

使用：配置 [image-webpack-loader](https://github.com/tcoopman/image-webpack-loader)
{% endblockquote %}

Imagemin 的优点分析
{% blockquote %}
1. 有很多定制选项

2. 可以引入更多第三方优化插件，例如pngquart

3. 可以处理多种图片格式
{% endblockquote %}

## 使用Polyfill


