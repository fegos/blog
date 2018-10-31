---
title: Nodejs之web服务基础
date: 2018-05-15
tags: nodejs
author: swx
---

### 规划
+ 服务器开发基础
+ 搭建一个简单的服务器
+ koa框架运行原理
+ 服务器线上运行保障

### 协议模型
#### 分层

理论上的网络协议分为7层（OSI），目前实际上是用的是tcp/ip 5层协议。

各层的职责：

物理层：确保数据在物理媒介上进行传输，主要规定了网络的一些电气特性，作用是负责传送0和1的电信号。

链路层：定义数据帧，确定主机的物理地址（MAC地址），以太网协议。

网络层：负责传输过程中路由的选择，找到目标的网络地址，实现主机到主机的传输，IP协议。

传输层：负责端到端的传输，对于多个使用网络的进程通过端口进行区分，TCP/UDP协议。

应用层：定义数据的格式，并按照对应的格式封包解包，HTTP协议。

![image](https://note.youdao.com/yws/api/personal/file/WEB4bf9b80aeb81ea1a4cdf8bc7f48fb146?method=download&shareKey=f84c786ef44bb3d96ad5ec2511d37bbf)

用户通过http发起请求的时候，应用层，传输层，网络层，链路层都会根据相关协议添加对应的首部，最终在数据链路层形成以太网数据包。然后通过物理层进行传输，到达对方主机后，再通过一层层解包还原传过来的数据包。

#### tcp连接的建立

tcp连接的建立要经历三次握手，以确保建立可靠连接，如下图所示：

![image](https://note.youdao.com/yws/api/personal/file/WEB3ecdd4743c6e92b3a6384254db7af6af?method=download&shareKey=8266dbec154736773df1f2409081580b)

(1) 第一次握手，Client发送SYN包到Server，告诉Server，本次消息的序列号为J

(2) 第二次握手，Server回复SYN + ACK包给Client，期待下次Client发送消息的序列号位J+1，并告诉Client，本次消息的序列号为K

(3) 第三次握手，Client回复ACK给Server，期待下次Server发送消息的序列号位K+1

#### tcp vs udp
协议|连接性|双工性|可靠性|有序性|有界性|拥塞控制|传输速度|头部大小
---|---|---|---|---|---|---|---|---
TCP|面向连接|全双工|可靠(重传机制)|有序(通过SYN排序)|无, 有粘包情况|有|慢|20~60字节
UDP|无连接|n:m|不可靠(丢包后数据丢失)|无序|有消息边界, 无粘包|无|快|8字节

常见应用场景

<table>
  <tr><th>传输层协议</th><th>应用</th><th>应用层协议</th></tr>
  <tr><td rowspan="5">TCP</td><td>电子邮件</td><td>SMTP</td></tr>
  <tr><td>终端连接</td><td>TELNET</td></tr>
  <tr><td>终端连接</td><td>SSH</td></tr>
  <tr><td>万维网</td><td>HTTP</td></tr>
  <tr><td>文件传输</td><td>FTP</td></tr>
  <tr><td rowspan="8">UDP</td><td>域名解析</td><td>DNS</td></tr>
  <tr><td>简单文件传输</td><td>TFTP</td></tr>
  <tr><td>网络时间校对</td><td>NTP</td></tr>
  <tr><td>网络文件系统</td><td>NFS</td></tr>
  <tr><td>路由选择</td><td>RIP</td></tr>
  <tr><td>IP电话</td><td>-</td></tr>
  <tr><td>流式多媒体通信</td><td>-</td></tr>
</table>

简单的说, UDP 速度快, 开销低, 不用封包/拆包允许丢一部分数据, 监控统计/日志数据上报/流媒体通信等场景都可以用 UDP. 目前 Node.js 的项目中使用 UDP 比较流行的是 [StatsD](https://github.com/etsy/statsd) 监控服务.

#### http协议

默认使用80端口

1991年的0.9版 -> 1996年的1.0版 -> 1997年的1.1版(目前主流版本)

支持的方法：

GET（0.9）

POST
HEAD（1.0新增）

PUT
PATCH
OPTIONS
DELETE（1.1新增）

+ 请求格式：
```
GET / HTTP/1.1
Host: localhost:8009
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7
```
第一行为请求方法和协议版本

Connection表示维持连接，不必要重新建立TCP连接。

User-Agent为浏览器的信息

Accept代表客户端可以接受的数据格式

Accept-Encoding代表客户端接受的压缩方式

+ 响应格式：

```
HTTP/1.1 200 OK
Accept-Ranges: bytes
content-type: text/html; charset=utf-8
content-length: 51807
keep-alive: timeout=5
Date: Mon, 09 Apr 2018 12:29:46 GMT
Connection: keep-alive

!DOCTYPE html> <html> <head> <meta charset=utf-8>
```

回应格式是“头信息 + 空行 + 数据”，其中第一行是协议版本，状态码，状态描述

content-type是告诉客户端，数据的格式是什么样的，以及数据所采用的编码。对应请求是的Accept

content-length为当前传输数据的字节数。由于一个TCP连接可以传送多个回应，所以需要区分数据包属于哪一个回应，这就是content-length的作用。

content-encoding代表当前采用的压缩方式。

+ x-www-form-urlencoded与multipart/form-data的区别

"application/x-www-form-urlencoded"，他是默认的MIME内容编码类型，一般可以用于所有的情况。但是他在传输比较大的二进制或者文本数据时效率极低。这种情况应该使用"multipart/form-data"。如上传文件或者二进制数据和非ASCII数据。

关于"application/x-www-form-urlencoded"和"multipart/form-data"的消息的区别可以看下面的例子：
这是一个表单，有2个表单域：name和email
```
 -------------------------------------
| field     value                                           |
| name:  ryan ou                                       |
| email:  ryan@rhythmtechnology.com      |
--------------------------------------
```
 
在 application/x-www-form-urlencoded 消息中:
name=ryan+ou&email=ryan@rhythmtechnology.com
(不同的field会用"&"符号连接;空格被替换成"+";field和value间用"="联系,等等)
 
 
再看multipart/form-data 消息中:
```
......
-----------------------------7cd1d6371ec
Content-Disposition: form-data; name="name"
 
ryan ou
-----------------------------7cd1d6371ec
Content-Disposition: form-data; name="email"
 
ryan@rhythmtechnology.com
-----------------------------7cd1d6371ec
Content-Disposition: form-data; name="Logo"; filename="D:\My Documents\My Pictures\Logo.jpg"
Content-Type: image/jpeg
 ......
 ```
(每个field被分成小部分，而且包含一个value是"form-data"的"Content-Disposition"的头部；一个"name"属性对应field的ID,等等)

当action为get时候，浏览器用x-www-form-urlencoded的编码方式把form数据转换成一个字串（name1=value1&name2=value2...），然后把这个字串append到url后面，用?分割，加载这个新的url。

当action为post时候，浏览器把form数据封装到http body中，然后发送到server。

如果没有 type=file 的控件，用默认的 application/x-www-form-urlencoded 就可以了。

但是如果有 type=file 的话，就要用到 multipart/form-data 了。浏览器会把整个表单以控件为单位分割，并为每个部分加上Content-Disposition(form-data或者file)、Content-Type(默认为text/plain)、name(控件name)等信息，并加上分割符(boundary)。

#### 一次http请求过程

打开地址栏，输入www.163.com，这意味着浏览器要想163发起一个网络请求。

发送数据包的时候，必须要知道对方的ip地址，这里我们只知道域名，所以要通过dns协议对域名进行解析，OS会向DNS服务器查询，然后DNS服务器返回163的IP地址为61.149.22.99

得到目标主机的IP后，需要通过子网掩码来判断目标主机是不是在同一网段，通过判断，目标主机不在同一网段，需要通过网关进行转发。

请求的HTTP内容如下：

```
GET  HTTP/1.1
Host: www.163.com
```

上面HTTP的数据会封装在TCP包里面，TCP设置好远程端口80，和本地端口（随机分配）

然后TCP数据包封装在IP数据包中，IP包中填入双方的IP地址，自己的IP地址为10.235.18.95，远程的IP地址为通过dns查询得到的61.149.22.99。

最后TCP数据要封装在以太网数据包中，以太网数据包需要设置双方的MAC地址，发送方为本机的网卡MAC地址，接收方为网关10.235.0.1的MAC地址（通过ARP协议得到）

经过多个网关的转发，163服务收到以太网的数据包，按照封包的逆顺序进行拆包，得到HTTP的数据，然后封装好HTTP的响应数据，发送给我们，浏览器收到服务器返回的数据后，直接展示出来，这样就完成了一次网络通信。

### Node相关

+ Node的前世今生

Node.js 诞生于 2009 年，由 Joyent 的员工 Ryan Dahl 开发而成，2015年Node基金会的成立，并发布了4.0版本。Node.js基金会的创始成员包括 Google、Joyent、IBM、Paypal、微软、Fidelity 和 Linux基金会，创始成员将共同掌管过去由 Joyent 一家企业掌控的 Node.js 开源项目。此后，Node.js基金会稳定的发布5、6、7、8、9等版本，截止到目前(2018.4.10)最新版本已经是9.11.1，长期支持版本是8.11.1。

+ Node是什么

>Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine. Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient. Node.js' package ecosystem, npm, is the largest ecosystem of open source libraries in the world.


Node.js 不是一门语言也不是框架，它只是基于 Google V8 引擎的 JavaScript 运行时环境。Node.js使用事件驱动、非阻塞I/O模型。Node.js的包管理器npm是全球最大的开源库生态系统。

Node.js结合 Libuv 扩展了 JavaScript 功能，使之支持 io、fs 等只有语言才有的特性，使得 JavaScript 能够同时具有 DOM 操作(浏览器)和 I/O、文件读写、操作数据库(服务器端)等能力，是目前最简单的全栈式语言。

+ Node中的事件

```
const EventEmitter = require('events');
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('触发了一个事件！');
});
myEmitter.on('another event', (data) => {
    console.log('触发了另一个事件！', data);
  });
myEmitter.emit('event');

setTimeout(()=>{
    myEmitter.emit('another event','msg');
},2000);
```

+ Node的框架

Node.js可分为四大部分：Node Standard Library（Native modules），Node Binding（builtin modules），V8，libuv

![image](https://yjhjstz.gitbooks.io/deep-into-node/chapter1/a9e67142615f49863438cc0086b594e48984d1c9.jpeg)

Node Standard Library用JS编写，供我们应用程序进行调用（Http模块），平时经常接触到的就是这部分库了。

Node Binding使用C\+\+编写，是连接JS和C\+\+的桥梁，封装了libuv和V8的细节，为上层标准库提供接口。

V8提供JavaScript的运行环境，执行我们的JS代码，可以认为是Node的发动机。

libuv重写了JS中的事件驱动，提供异步I/O模型，提供跨平台的支持。

### 总结
+ 互联网协议的构成
+ TCP连接建立过程
+ HTTP协议
+ 一次完整的HTTP请求
+ Node介绍

### 参考
http://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html
https://www.cnblogs.com/onepixel/p/7092302.html
