---
title: Webpack4入门配置指南1
date: 2019-11-25 22:32:18
tags: [前端,webpack]
---
# 1. 安装Webpack
`Webpack`官方并不建议全局安装，因此局部安装就好了，主要有以下三个模块：
1. `webpack` 主模块
2. `webpack-cli` 脚手架
3. `webpack-dev-server` 本地开发环境

由于懒，就一次性装三个了：
```
cnpm install webpack webpack-cli webpack-dev-server --save-dev
```

安装完之后，在根目录下先建立三个文件夹`dist`、`src`、`config`，它们分别存放`出口文件`、`源文件`、`webpack配置文件`。

然后在`config`文件夹新建一个文件`webpack.dev.js`，用于写配置信息，接下来都是在这个文件中作文章。

因此，先把`webpack.dev.js`文件的主结构写好：

```javascript
module.exports = {
 
    // 模式
    // 'production' 生产环境会对JS进行压缩
    // 'development' 开发环境
    mode: 'development',
    // 入口文件
    entry: {},
    // 打包后的出口文件
    output: {},
    // 模块，例如读CSS模块、图片压缩模块等
    module:{},
    // 插件，后面压缩、分离要用到
    plugins:[],
    // 本地环境
    devServer:{}
}
```

然后就开始了！

# 2. 打包JS
假如我们在`src`目录建立两个文件，一个为`add.js`，一个为`app.js`，大致如下：
```javascript
// add.js
function add(a,b) {
    return a + b;
}

export default add;

// app.js
// 引入add.js文件的add函数
import add from './add';

console.log('这是main.js');
console.log('5 + 3 = ' + add(5,3));
```

因为浏览器是不支持这样引入的，所以，我们才要用`webpack`进行打包，把它写到同一个文件上去，才能让浏览器使用。

然后，更改配置文件`webpack.dev.js`：
```javascript
const path = require('path');
module.exports = {

    mode: 'development',
    entry: {
        main: './src/app.js',
    },
    output: {
        path: path.resolve(__dirname,'../dist'),
        filename: 'bundle.js'
    },
    module:{},
    plugins:[],
    devServer:{}
```

通过上面的配置描述，我们把`/src/app.js`作为入口文件打包成`dist`目录下的`bundle.js`文件。

然后执行命令：
```
webpack --config=config/webpack.dev.js
```

然后就会输出一堆：
```
Hash: b64ba297710d7b23b743
Version: webpack 4.20.2
Time: 2193ms
Built at: 2018-10-16 20:12:33
    Asset     Size  Chunks             Chunk Names
bundle.js  502 KiB   main1  [emitted]  main1
Entrypoint main1 = bundle.js
[./node_modules/_webpack@4.20.2@webpack/buildin/global.js] (webpack)/buildin/global.js 509 bytes {main1} [built]
[./src/add.js] 146 bytes {main1} [built]
[./src/app.js] 312 bytes {main1} [built]
    + 326 hidden modules
```

这时候说明打包成功，我们会发现`dist`目录下多了个`bundle.js`，我们建立一个`html`把它引入，就能看到执行结果了。

**如果有多个入口文件怎么办？**那就这样修改配置文件：
```javascript
const path = require('path');
module.exports = {

    mode: 'development',
    entry: {
        main1: './src/app.js',
        main2: './src/app2.js',
    },
    output: {
        path: path.resolve(__dirname,'../dist'),
        filename: '[name].bundle.js'
    },
    module:{},
    plugins:[],
    devServer:{}
}
```

在`entry`中编写多个属性，属性名映射到`output`中`[name]`。例如，上面的`main1`和`main2`通过打包分别会在`dist`目录生成`main1.bundle.js`和`main2.bundle.js`。千万要记住属性名不能重复，否则后定义的会覆盖先定义的。

同样地，执行命令:
```
webpack --config=config/webpack.dev.js
```

打包成功就不说了，但是每次打包都执行这么一长串命令有点麻烦，因此，我们打开`package.json`文件，它现在是这样的：
```json
{
  "name": "study_webpack_2",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "webpack": "^4.20.2",
    "webpack-cli": "^3.1.2",
    "webpack-dev-server": "^3.1.9"
  },
  "dependencies": {}
}
```

我们修改`scripts`属性，添加一个：
```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "webpack --config=config/webpack.dev.js"
  },
```

然后需要打包时，只需要执行一小段命令：
```
npm run dev
```

# 兼容ES6
我们可以通过使用`babel`来把ES6的语法转换成ES5的，使之在低版本浏览器能运行。

这时候，就需要装几个包来支持了：
1. `babel-loader` 负责es6语法转化
2. `babel-preset-env` 包含es6、7等版本的语法转化规则
3. `babel-polyfill` es6内置方法和函数转化垫片
4. `babel-plugin-transform-runtime`
5. `babel-runtime`

其中，1\2\4 安装在开发环境的依赖中，3\5 安装在生产环境
```
cnpm install babel-loader@7 babel-preset-env babel-plugin-transform-runtime --save-dev
```

```
cnpm install babel-polyfill babel-runtime --save
```

需要注意的是，`babel-loader`建议指定版本号为`@7`，因为我在装`8.x.x`版本时就发生了一个未知错误，看了网上很多例子说安装`babel-core`还是不行，所以降到了`7.x.x`，发现居然可以了，无语。

同样地，修改配置文件：
```javascript
const path = require('path');
module.exports = {

    mode: 'development',
    entry: {
        main1: './src/app.js',
        main2: './src/app2.js',
    },
    output: {
        path: path.resolve(__dirname,'../dist'),
        filename: '[name].bundle.js'
    },
    module:{
        rules: [
            {
                test: /\.js$/,
                exclude: /(node_modules)/,
                use: { 
                    loader: 'babel-loader'
                }
            }
        ]
    },
    plugins:[],
    devServer:{}
}
```

同时我们要避免把`node_modules`里面的文件给转换了。因此要添加`exclude: /(node_modules)/`。

然后，必须在入口文件引入`babel-polifill`，它会将ES6的新语法糖用ES5实现，所以我们在入口文件定义：
```javascript
// app.js
import "babel-polyfill";

// 引入add.js文件的add函数
import add from './add';

console.log('这是app.js');
console.log('5 + 3 = ' + add(5,3));

// app2.js
import "babel-polyfill";

// 引入add.js文件的add函数
import add from './add';

console.log('这是app2.js');
console.log('8 + 8 = ' + add(8,8));
```

这样，即使我们把add.js的函数改成ES6的箭头函数，打包输出的JS代码依然可以在老版本浏览器使用。

然后执行打包命令，就不多说了。

# 打包CSS
打包CSS文件时，需要用到`css-loader`和`style-loader`。

安装：
```
cnpm install css-loader style-loader --save-dev
```

同样，修改配置文件：
```javascript
const path = require('path');
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
                        {
                          loader: 'style-loader'
                        },
                        {
                          loader: 'css-loader'
                        }
                ]
            }
        ]
    },
    plugins:[],
    devServer:{}
}
```

这里要判断以`.css`结尾的文件，然后通过两个加载器进行CSS的加载。

假如写一个简单的HTML：
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>Page Title</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script src="bundle.js"></script>
</head>
<body>
    <h1>Hello world</h1>
</body>
</html>
```

简单的`main.css`：
```css
body {
    background: green;
    color: #fff;
}
```

入口文件`app.js`：
```javascript
// 引入css
import './main.css';
```

然后执行`npm run dev`打包，就会发现有效果了。

# 如果是SASS/SCSS
如果是预处理语言SASS，就需要安装`node-sass`和`sass-loader`来支持。
```
cnpm install node-sass sass-loader --save-dev
```

然后修改配置文件，跟之前操作CSS的很像：
```javascript
const path = require('path');
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
    plugins:[],
    devServer:{}
}
```

然后简单写以下scss文件：
```scss
$myred: red;

body {
    background: $myred;
    
    h1 {
        text-decoration-line: underline;
        color: aqua;
    }
}
```

在入口文件`app.js`引入
```javascript
import './main.scss';
```

然后执行命令打包，完成。

# CSS/SASS样式分离
前面我们的CSS代码在打包时都是与JS代码打包在一起的，现在我们尝试把它们分开。

这时候需要用到一个插件：`extract-text-webpack-plugin`，对于Webpack4，要这样安装：
```
cnpm install extract-text-webpack-plugin@next --save-dev
```

后面是有`@next`的，必须注意了！！

然后引入
```javascript
const extractTextPlugin = require('extract-text-webpack-plugin');
```

修改配置文件
```javascript
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
            use: extractTextPlugin.extract({
                fallback: "style-loader",
                use: "css-loader"
            }),
        },
        {
            test: /\.scss$/,
            use: extractTextPlugin.extract({
                fallback: "style-loader",
                use: [
                    { loader: 'css-loader' },
                    { loader: 'sass-loader' }
                ]
            }),
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
    }),
    new extractTextPlugin('css/main.css') // CSS分离后的路径
],
```

然后打包，就OJBK了。