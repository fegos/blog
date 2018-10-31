---
title: Nodejs之web服务开发
date: 2018-05-15
tags: nodejs
author: swx
---

### 内容大纲
+ 原生http
+ connect
+ koa

### 原生http

#### http模块的组成
node里面内置了http的模块，http模块中四个很重要的类会经常用到，分别是`http.Server`,`http.ClientRequest`,`http.ServerResponse`,`http.IncomingMessage`，下图反应了他们之间的关系。

![image](https://note.youdao.com/yws/api/personal/file/WEB038b0dbdf13606320ea6a08cde1d3bb5?method=download&shareKey=7888dd66e68cafbae317d29777063ecd)

#### 简单的http服务器
一个简单的http服务端实现是这样:
```
//demo1.js
const http = require('http');
let num = 0;
http.createServer((request,response)=>{
    console.log('new request ! ',++num);
    response.end('Hello World');
}).listen(8080);
console.log('now listen on 8080');
```
`createServer`方法会返回一个`http.Server`的实例，创建的时候可以传入一个`requestListener`，该函数会自动绑定到 `request`事件上，当客户端向服务器端发起请求的时候，就会触发`request`事件，`request`事件的回调函数有两个参数，第一个参数是`http.IncomingMessage`，第二个参数是`http.ServerResponse`，`http.IncomingMessage`代表`client`端的信息，比如httpVersion，headers，method等。`http.ServerResponse`包含服务端一些列数据处理方法，比如设置头信息，往sokcet中写入数据等。

#### 文件的传输

node中提供了`Stream类`，结合`http.SeverResponse`类，可以很方便的将文件流接入到socket流里面。如果服务端不设置编码类型，默认为chunked，这是由node异步机制决定的，下面是传输图片的例子。

```
//server.js
const http = require('http');
let num = 0;
http.createServer((req,res)=>{
    console.log('new request ! ',++num);
    res.writeHead(200,{'Content-Type':'image/jpeg'});
    require('fs').createReadStream('desert.jpeg').pipe(res);
}).listen(8080);
console.log('now listen on 8080');
```
#### 主动发起请求
Node中，可以通过`http.request`或者`http.get`主动发起请求，这两个函数均返回一个`http.ClientRequest`的实例。发起请求的时候，可以传入一个回调函数，该回调函数会绑定在`response`事件上，当请求相应被接收到的时候触发该事件，回调函数的参数为`http.IncomingMessage`的实例。

```
//client.js
http.get('http://127.0.0.1:8080',(res)=>{
	let num = 0;
    res.on('data',(data)=>{
		console.log('******chunk data******\r\n');
		console.log(`chunk: ${++num}  len: ${data.length}\n`);
    });
    res.on('end',()=>{
		console.log('******responese end******\r\n');
    });
});
```

#### 获取client数据
编写服务器的时候，针对客户端不同的请求路径，需要展示不同的页面，这个过程称为路由，请求路径通过`http.IncomingMessage`实例的url可以拿到，`http.IncomingMessage`实现了可读流的接口，通过流的`data`事件和`end`事件，能够获取来自client端提交过来的数据。

```
const http = require('http');
const qs=require('querystring');

let num = 0;
http.createServer((req,res)=>{
    console.log('new request ! ',++num);
    res.writeHead(200,{'Content-Type':'text/html; charset=utf-8'});
    if(req.url === '/register' && req.method === 'POST') {
        let content = '';
        req.on('data',(chunk)=>{
            content += chunk;
        });
        req.on('end',()=>{
            let param = qs.parse(content);
            param = JSON.stringify(param);
            res.end(`<p>Content-Type: ${req.headers['content-type']}</p><p>Data: ${param}</p>`,'utf-8');
        });
    } else if('/' === req.url){
        res.end([
                '<form method="POST" action="/register">',
                '<h1>表单</h1>',
                '<fieldset>',
                '<label>个人信息</label>',
                '<p>姓名: <input type="text" name="name"/> </p>',
                '<p><button>提交</button></p>',
                '</form>'
        ].join(''));
    } else {
        res.writeHead(404)
        res.end('Not Found');
    }
}).listen(8080);
console.log('now listen on 8080');
```
服务端在writeHead的时候需要指定charset，不然客户端会显示为乱码，res.write()和res.end()的时候默认编码是utf-8。

#### 辅助工具
调试服务器的时候，可能会需要经常改动服务端代码，停掉服务器，然后重启服务器，这一个过程会非常麻烦，这时候可以使用第三方的工具`nodemon`，`nodemon`会监视文件的变化，然后自动重启服务器。
```
npm i nodemon --save-dev
```
nodemon的配置
```
//package.json

  "nodemonConfig": {
    "ignore": ["test/*", "docs/*", "*.jpeg"],
    "delay": "2500"
  }
```

#### 静态网站
```
const http = require('http');
const fs = require('fs');

function notfound(res) {
    res.writeHead(404);
    res.end('Not Found');
}

function show(path, type, res) {
    fs.stat(path, (err, stat) => {
        if (err || !stat.isFile()) {
            notfound(res);
        } else {
            res.writeHead(200, {
                'Content-Type': type
            });
            fs.createReadStream(path).pipe(res);
        }
    });
}
const STATIC_PATH = '/public'
const server = http.createServer((req, res) => {
    console.log('url: ', req.url);
    if (req.url.startsWith('/image')) {
        let suffixIndex = req.url.lastIndexOf('.');
        if (suffixIndex > 0) {
            let suffix = req.url.substring(suffixIndex + 1)
            show(__dirname + STATIC_PATH + req.url, 'image/' + suffix, res);
        } else {
            notfound(res);
        }
    } else if (req.url === '/') {
        show(__dirname + STATIC_PATH + '/index.htm','text/html', res);
    } else {
        notfound(res);
    }
});
server.listen(8080);
server.on('error',(err)=>{
    console.log(err);
});
```
Error: listen EACCES 0.0.0.0:80

原因监听 1024 以下端口 需要sudo权限，否则报   listen EACCES 错误

### connect

#### http服务器
```
const http = require('http');
const connect = require('connect');
const app = connect();
app.use((req,res,next)=>{
    res.end('Hello from connect\r\n');
});
app.listen(8080);
```
#### 静态资源服务器

```
const http = require('http');
const connect = require('connect');
const app = connect();
const serveStatic = require('serve-static');
app.use(serveStatic('public',{'index':['index.html','index.htm']}));
app.listen(8080);

```

#### connect中间件

```
const http = require('http');
const connect = require('connect');
const responseTime = require('response-time');
const app = connect();

app.use(responseTime());
app.use((req,res,next)=>{
    console.log('111');
    next();
    console.log('222');
});
app.use('/api',(req,res,next)=>{
    console.log('333');
    res.write('before next\n');
    next();
    res.end('hello api');
    console.log('444');
});
app.use((req,res,next)=>{
    console.log('555');
    next();
    console.log('666');
});
app.use((req,res,next)=>{
    res.write('common info\n');
    res.end('end');
});
app.listen(8080);
console.log('server start at port: ', 8080);
```

#### connect中的错误处理
```
const http = require('http');
const connect = require('connect');
const app = connect();

app.use((req,res,next)=>{
    setTimeout(()=>{
        console.log('async');
        throw(new Error('Async Error Demo'));
    },1000);
    throw(new Error('Error Demo'));
    next();
});

app.use((req,res,next)=>{
    console.log('middware');
    next();
});

app.use((err,req,res,next)=>{
    console.log("err1:",err);
    res.end('err1');
    // next(err);
});
app.use((err,req,res,next)=>{
    console.log("err2:",err);
    res.statusCode = 500;
    res.end('Internal Error');
});
app.listen(8080);
console.log('server start at port: ', 8080);
```
原则:
+ 通过特殊的中间件进行处理，参数为4个，第一个为错误信息
+ 出错后，寻找最近的一个错误中间件进行处理
+ 可以通过next(err)传递到下一个错误中间件
+ 对于异步抛出的错误，需要在异步中调用next(err)来抛出错误。

#### callback hell
```
app.use((req,res,next)=>{
    console.log('111');
    proxy.fetchUser((err, data)=>{
        if(!err){
            console.log('user fetched!');
            res.write(JSON.stringify(data));
            res.write('\n');
            proxy.fetchInfo((err,data)=>{
                if(!err){
                    console.log('info fetched');
                    res.write(JSON.stringify(data));
                    res.write('\n');
                }
                next();
            });
        } else {
            //error 
            next();
        }
    });
    console.log('222');
});
```
express是对connect的封装，驱动方式一样，内置了很多组件，比如router，logger等，还有很多第三方组件方便开发者使用。express还支持使用模板渲染网页。express的使用可以参考https://github.com/expressjs/express。

#### 通过模板渲染页面

```
//index.ejs

<h1>express demo</h1>
<p>输入你的名字</p>
<form action="/register" method="POST">
    <input type="text" name=name>
    <button>提交</button>
</form>

//normal.ejs

<h1>欢迎</h1>
<b><%= name %></b>
```

#### express服务器
```
const express = require('express');
const app = express();
const bodyParser = require('body-parser')
app.set('view engine', 'ejs');
app.set('views', __dirname + '/views');
app.set('view options', {
    layout: false
})
app.use(bodyParser.urlencoded({
    extended: true
}));

app.get('/', (req, res) => {
    res.render('index');
});

app.post('/register', (req, res, next) => {
    res.render('normal', {
        name: req.body.name
    });
})

app.get('/api/check', (req, res) => {
    res.json({
        retcode: 200,
        retdesc: 'success'
    });
});
app.use((err, req, res, next) => {
    console.log(err);
});
app.listen(8080);
```

### koa
koa是和connect类似的框架，koa使用async、await消除了深层的callback，可以用同步的方式去写异步的代码，koa的理念和express不同，koa只提供必要的组件，核心框架很简洁，没有内置其他的组件，我们在实现某个功能的时候，可以根据自己的需要去选择喜欢的第三方组件，koa支持的第三方组件可以上[wiki](https://github.com/koajs/koa/wiki)上查询。

#### koa的例子

```
const koa = require('koa');
const staticServer = require('koa-static');
const logger = require('koa-logger');
const koaRouter = require('koa-router');
const bodyParser = require('koa-bodyparser');

const app = new koa();

app.use(async(ctx,next)=>{
    console.log('koa---1');
    await next();
    console.log('koa---2');
})

app.use(bodyParser());

app.use(logger());

const router = new koaRouter();
const apiRoute = require('./routes/api');
router.use('/api',apiRoute.routes());
app.use(router.routes()).use(router.allowedMethods());
//跳转
router.get('/redirect',async(ctx,next)=>{
    ctx.response.redirect('/');
    ctx.body = 'Main Page';
});
app.use(staticServer(__dirname + '/public'));

app.listen(8080);
```
对ctx.body赋值就是将数据返回给客户端，可以通过ctx.redirect()来实现302跳转，设置ctx.status可以对http的status code进行设置，如果没有对ctx.body进行赋值，则ctx.status默认为404，如果已经对ctx.body赋值，则ctx.status值为200。

#### koa中的错误处理
得益于koa的框架，捕获服务器的错误变得非常方便，只需要加入一个`try catch`的代码块将`next()`函数包裹起来就可以了。

```
const koa = require('koa');
const app = new koa();
app.use(async (ctx, next) => {
    try {
        await next();
    } catch (err) {
        console.log('error handler', err);
    }
});

app.use(async (ctx, next) => {
    console.log('koa---1');
    await next();
    console.log('koa---2');
})

app.use(async (ctx, next) => {
    console.log('koa---3');
    // throw new Error('error3');
    await next();
    console.log('koa---4');
    // throw new Error('error4');
})

app.use(async (ctx, next) => {
    console.log('koa---5');
    await next();
    console.log('koa---6');
})
app.use(async (ctx,next) =>{
   ctx.body = 'ok' ;
});
app.listen(8080);
```
在error3和error4的位置抛出异常都能够被错误处理中间件捕获，和connect类似，当错误抛出的时候，koa不在执行其他的中间件，直接跳到错误处理中间件处理错误。

通过`try catch`捕获的错误不会触发app的error事件，如果需要在`app.on('error')`中继续处理该错误，可以通过`ctx.app.emit('error',err)`来释放该error事件。

### 服务器的一般目录结构
```
├── logs
├── package.json
├── public
│   ├── index.html
│   └── news.html
├── server
│   ├── app.js
│   ├── config
│   ├── controller
│   ├── routes
│   └── service
└── views
    ├── index.ejs
    └── welcome.ejs
```
logs为输入日志的目录，public为静态文件目录，线上环境下一般放在cdn上，然后通过nginx进行转发，view为模板目录，config为服务器配置目录，一般服务器都会分好几个环境，比如dev，test，online环境。service为各种服务的抽象，一般用来做业务逻辑的封装，减轻controller层的任务，保证业务逻辑的独立性，提供给不同controller调用。controller层负责解析用户的数据，并请求相应的service获取数据，拼接成用户所需要的数据返回给用户。routes里面对路由进行设置，不同的请求路径调用不同的controller。

### 课后作业
#### 目标
实现一个版本管理后台

#### 需求
+ 提供版本录入功能
    + 包含当前版本号，系统(Android/IOS/RN)，环境(dev/online)，是否强制更新，灰度级别(1%,10%,30%,50%,100%)
+ 修改灰度级别
+ 上线，下架功能
+ 获取最新版本信息(传入系统环境参数)
+ 额外
    + 白名单(保证每次都会灰度)
    + 白名单的管理(增删改查)
