---
title: Webpack4入门配置指南2
date: 2019-11-25 22:32:48
tags: [前端,webpack]
---

# 压缩JavaScript代码
一般来说，对于需要上线的项目，都要进行代码压缩。

对于Webpack4，已经集成了一个压缩插件`uglifyjs-webpack-plugin`，我们只需要在配置文件引入和使用就行了。

```javascript
const uglify = require('uglifyjs-webpack-plugin');
```

然后在`plugins`中使用：

```javascript
plugins:[
    new uglify()
],
```

执行`npm run dev`，然后`dist`打开目录下的出口文件就会看见已经被压缩过的代码。


# 打包HTML
我们把HTML写好后，要引入打包后的js文件和css文件。为了更方便引入，所以用webpack帮助我们打包html文件，并且自动引入js文件。

打包HTML需要`html-webpack-plugin`插件，先来安装：

```
cnpm install html-webpack-plugin --save-dev
```

然后修改配置文件：

```javascript
const path = require('path');
const uglify = require('uglifyjs-webpack-plugin');
const htmlPlugin = require('html-webpack-plugin');

module.exports = {

    mode: 'development',
    entry: {
        main: './src/app.js',
    },
    output: {
        path: path.resolve(__dirname,'../dist'),
        filename: 'bundle.js'
    },
    module:{
        rules: [
            {
                test: /\.js$/,
                exclude: /(node_modules)/,
                use: { 
                    loader: 'babel-loader'
                }
            },
            {
                test: /\.css$/,
                use: [
                        { loader: 'style-loader'},
                        { loader: 'css-loader'}
                    ]   
            },
            {
                test: /\.scss$/,
                use: [
                    { loader: 'style-loader'},
                    { loader: 'css-loader'},
                    { loader: 'sass-loader'}
                ]
            }
        ]
    },
    plugins:[
        new uglify(),
        new htmlPlugin({
            // 压缩
            minify: {
                removeAttributeQuotes: true // 清除属性名的引号
            },
            hash: true, // 利用hash可以避免开发中的js缓存
            template: './src/index.html' // 需要打包的html文件
        })
    ],
    devServer:{}
}
```

然后执行`npm run dev`打包，OK。