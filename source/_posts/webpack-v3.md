---
title: Webpack3.0
date: 2017-11-14
tags: webpack
author: henry
---

## 主要特性
* 依赖解析, 打包合并
* 支持异步模块, 按需加载
* 预编译代码, 通过loaders处理es6, jsx, less, sass等格式的模块
* 多任务处理, 通过plugins提供代码混淆、压缩、样式提取等功能
* 支持 ES6 & commonJS 模块
* 支持 sourcemap

## 主要流程
webpack构建时, 从入口文件开始, 循环的解析依赖, 并将项目所需要的依赖, 打包成一个或多个文件输出

生成compiler -> 处理入口模块 -> 解析模块依赖 -> 调用loader加载模块 -> 打包文件, 拆分异步chunks -> 调用插件处理文件内容 -> 输出模块文件

## 核心概念
* 入口文件 (Entry) 构建项目和解析依赖关系的入口
* 输出文件 (Output) 打包后文件的输出格式和位置
* 加载器 (Loaders) 针对不同格式模块的预处理器, 可以替换或转义代码内容
* 插件 (Plugins) 代码打包输出前的最后处理, 如压缩和混淆等操作

### Entry
**单一入口, entry: string | Array <string>**
```
const config = {
  entry: './path/to/my/entry/file.js'
}
```
**多个入口,  entry: {[entryChunkName: string]: string | Array<string>}**
```
const config = {
  entry: {
    vendor: './path/to/my/entry/file1.js',
    index: ['./path/to/my/entry/file2.js', './path/to/my/entry/app.js']
  }
}
```

### Output
输出配置至少需要以下两个配置
* filename, 输出文件的名称(格式)
* path, 输出文件夹的绝对路径

```
const config = {
  output: {
    filename: 'bundle.js',
    path: '/home/proj/public/assets'
  }
}
```
额外的属性
* chunkFilename, 拆分bundle文件的名称(格式)
* publicPath, 静态资源的线上路径

```
const config = {
  output: {
    path: '/home/proj/build',
    publicPath: '/cdn/'
    filename: '[name].[chunkhash:8].js',
    chunkFilename: 'chunk.[name].[chunkhash:8].js',
  }
}
```
**常用的名称格式有以下几种**

| 标记 | 描述 |
| --- | --- |
| [hash] | 模块id的hash值 |
| [chunkhash] | chunk内容的hash值 |
| [name] | 模块的名称 |
| [id] | 模块的id |

### Loader
必要的属性
* test, 正在表达式, 用于匹配文件名
* loader/use, 处理文件内容的加载器

额外的属性
* include, 指定文件的目录范围
* exclude, 指定排除的目录
* options/query, 加载器的参数配置


```
module: {
  rules: [{
    test: /\.jsx?$/,
    loader: 'babel-loader',
    exclude: [
      path.resolve(SRC_PATH, '../node_modules')
    ]
  }, {
    test: /\.css?$/,
    use: ['style-loader', 'css-loader', 'postcss-loader']
  }]
}
```
### Plugin
提供定制化的webpack编译模式, 下面是一些常用插件
```
plugins: [
  // build optimization plugins
  new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    filename: 'vendor-[hash].min.js',
  }),
  new webpack.optimize.UglifyJsPlugin({
    compress: {
      warnings: false,
      drop_console: false,
    }
  }),
  new ExtractTextPlugin({
    filename: 'index.min.css'
  }),
  new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),
  // compile time plugins
  new webpack.DefinePlugin({
    'process.env.NODE_ENV': '"production"',
  }),
  // webpack-dev-server enhancement plugins
  new webpack.HotModuleReplacementPlugin()
]
```
## 升级 3.0
此次从1.0升级到3.0, 带来了许多新的功能点

* 支持ES6模块, 如import, export等语法(v2)
* 针对ES6模块, 提供动态加载方法`import()`(v2), 并通过注释定义模块信息(v3)
* 基于ES6模块的静态结构, 提供Tree shaking功能, 移除未被使用的模块输出(v2)
* 作用域提升/合并, 通过启用插件`webpack.optimize.ModuleConcatenationPlugin`, 减少模块打包后产生的闭包数量, 优化代码执行速度和打包体积(v3)
* 集成常用loader, 如less, sass, CoffeeScript, TypeScript等

## 迁移至新版本
[webpack官方文档](https://doc.webpack-china.org/guides/migrating/#resolve-root-resolve-fallback-resolve-modulesdirectories)
* resolve.modulesDirectories 改为 resolve.modules, 取消resolve.root
* module.loaders 改为 module.rules, 并取消加载器自动添加'-loader后缀'
* json-loader 不再需要手动添加, 已自动集成
* loader 默认相对于 context 进行解析, 解决了引用 context 上下文目录之外的模块时，loader 所导致的模块重复载入的问题
* UglifyJsPlugin 的默认配置 sourceMap & warnings 改为 false
* loaders默认不再使用压缩模式, 可以通过插件`webpack.LoaderOptionsPlugin({minimize:true})`开启
* 移除 webpack.optimize.DedupePlugin, 默认加载 webpack.optimize.OccurenceOrderPlugin
* ExtractTextWebpackPlugin有较大改动, 具体内容请参考文档
* require.ensure 以及 AMD require 将采用异步式调用, 而不是当 chunk 已经加载完成的时候同步调用它们的回调函数(callback)
* 自定义属性配置loader无效, 只能通过options配置
* debug模式, 需要通过loader的options配置
* require.ensure & import() 两种方式实现动态加载模块, 可以混合使用ES2015、AMD 和 CommonJS模块, 但是每个模块内只能由一种模块格式
* 让webpack处理ES6模块, 以启用tree shaking, 需要通过.babelrc或者babel-loader设置`modules:false`
* 支持配置文件中返回Promise对象
* 支持对loader更多方式的匹配
