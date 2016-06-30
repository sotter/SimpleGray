---
layout: post
title: HTTP协议分析
category: default
---


####HTTP协议解析

本文通过两个示例来对HTTP进行解析学习，在服务端启动了一个简单的测试服务器，对于收到的所有请求返回```{"code":2}``` 的字符串，通过浏览器和curl的方式分别模拟了GET和POST请求；
    
####***1. GET请求分析***

***1.1 操作方式***
在浏览器中地址栏直接输入：```http://10.11.3.50:8081/load_balancer?client_type=1&client_version=300```

***1.2 数据抓包***

REQUEST:

```
GET /load_balancer?client_type=1&client_version=300 HTTP/1.1
Host: 10.11.3.50:8081
Connection: keep-alive
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.86 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8
```

***1.3 详细说明***

http请求由三部分组成，分别是：请求行、消息报头、请求正文三部分组成，在实际的抓包中发请求行与消息报头之间是只有一个\r\n 消息报头和消息体之间有两个\r\n。

***1.3.1 请求行： GET /load_balancer?client_type=1&client_version=300 HTTP/1.1***

格式：`Method Request-URI HTTP-Version CRLF`

对应关系：

|字段|对应值|备注|
|---|---|---|
|Method|GET|
|Request-URI|/load_balancer?client_type=1&client_version=300|
|HTTP-Version|HTTP/1.1|
|结束符|\r\n|

**引申：除了GET方法还有以下几个方法， 主要用到的是前6种**

|Method|说明|
|---|---|
|GET     |请求获取Request-URI所标识的资源 |
|POST    |在Request-URI所标识的资源后附加新的数据|
|HEAD    |请求获取由Request-URI所标识的资源的响应消息报头|
|PUT     |请求服务器存储一个资源，并用Request-URI作为其标识|
|DELETE  |请求服务器删除Request-URI所标识的资源|
|TRACE   |请求服务器回送收到的请求信息，主要用于测试或诊断|
|CONNECT |保留将来使用|
|OPTIONS |请求查询服务器的性能，或者查询与资源相关的选项和需求|

***1.3.2 消息头***

`Host: 10.11.3.50:8081` -> 表示要连接的服务器地址, 不多解释；

`Connection: keep-alive` ->使用持久连接，看下面的维基解释：

维基解释：

	这样做，连接就不会中断，而是保持连接。当客户端发送另一个请求时，它会使用同一个连接。这一直继续到客户端或服务器端认为会话已经结束，其中一方中断连接。

	在 HTTP 1.1 中 所有的连接默认都是持续连接，除非特殊声明不支持。[1] HTTP 持久连接不使用独立的 keepalive 信息，而是仅仅允许多个请求使用单个连接。然而， Apache 2.0 httpd 的默认连接过期时间[2] 是仅仅15秒[3] ，对于 Apache 2.2 只有5秒。[4] 短的过期时间的优点是能够快速的传输多个web页组件，而不会绑定多个服务器进程或线程太长时间。[5]


> Cache-Control: max-age=0``` 缓存策略，为了提升性能和减少服务器重复请求的压力， max-age表示缓存失效时间；告知服务端是没有响应的；

> Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8```

表示浏览器想要接受的文件格式，如果是文本的html格式，application的xhtml+xml格式的，image webp格式的；

`Upgrade-Insecure-Requests: 1` 浏览器可以自动升级请求；

```User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.86 Safari/537.36``` 浏览器把自己的信息告知服务端，服务端可以根据不同的浏览器类型和版本号做适配；

```Accept-Encoding: gzip, deflate, sdch``` 浏览器支持的编码格式；

```Accept-Language: zh-CN,zh;q=0.8```  浏览器可支持的语言类型，如果这个字段没有设置，表示各种语言都可以接受；

####2. 响应数据分析

RESPONSE：wireshark 抓包

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=utf-8
    Date: Tue, 03 May 2016 04:27:59 GMT
    Connection: keep-alive
    Content-Length: 11
    
    {"code":2}

通过TCPDUMP的方式可以更原生态的字节流：

	0x0000:  4500 00d1 05d1 4000 4006 177c 0a0b 0332  E.....@.@..|...2
	0x0010:  0a0b 0593 1f91 fa4a f72a 7989 4fed 2b48  .......J.*y.O.+H
	0x0020:  8018 4d78 1d9e 0000 0101 080a a942 6508  ..Mx.........Be.
	0x0030:  2fbd 87c6 4854 5450 2f31 2e31 2032 3030  /...HTTP/1.1.200
	0x0040:  204f 4b0d 0a43 6f6e 7465 6e74 2d54 7970  .OK..Content-Typ
	0x0050:  653a 2061 7070 6c69 6361 7469 6f6e 2f6a  e:.application/j
	0x0060:  736f 6e3b 6368 6172 7365 743d 7574 662d  son;charset=utf-
	0x0070:  380d 0a44 6174 653a 2054 7565 2c20 3033  8..Date:.Tue,.03
	0x0080:  204d 6179 2032 3031 3620 3034 3a33 323a  .May.2016.04:32:
	0x0090:  3435 2047 4d54 0d0a 436f 6e6e 6563 7469  45.GMT..Connecti
	0x00a0:  6f6e 3a20 6b65 6570 2d61 6c69 7665 0d0a  on:.keep-alive..
	0x00b0:  436f 6e74 656e 742d 4c65 6e67 7468 3a20  Content-Length:.
	0x00c0:  3131 0d0a 0d0a 7b22 636f 6465 223a 327d  11....{"code":2}
	
Respone主要由3部分组成：状态行，包头，包体组成； 通过抓包来看，状态行和header之间只有一个换行，包头和包体之间有两个换行；

***2.1 状态行***
HTTP/1.1 200 OK   由协议版本、状态码、message三部分组成；

引申：状态码
|状态码|含义|示例|
|---|---|---|
|1XX |提示信息 - 表示请求已被成功接收，继续处理 | |
|2XX |成功 - 表示请求已被成功接收，理解，接受 ||
|3XX |重定向 - 要完成请求必须进行更进一步的处理 | |
|4XX |客户端错误 - 请求有语法错误或请求无法实现 | |
|5XX |服务器端错误 - 服务器未能实现合法的请求| |

详细含义可查看维基百科：
https://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81

***2.2 Header***

Content-Type: 内容类型，在本实例中是json类型；
其他不做详解了；

####3. POST请求分析

***3.1 操作方式***

在命令行执行如下命令：

curl http://10.11.3.50:8080/load_balancer -d '{"userId":79631315,"clientIp":"117.136.0.23","clientType":"ios"}'

***3.2 数据抓包***

抓到的请求包如下：

```
POST /load_balancer HTTP/1.1
Host: 10.11.3.50:8080
User-Agent: curl/7.43.0
Accept: */*
Content-Length: 64
Content-Type: application/x-www-form-urlencoded

{"userId":79631315,"clientIp":"117.136.0.23","clientType":"ios"}
```

***3.3 说明***
具体的GET和POST的请求资料一大堆，不做详细解释；这里仅仅从http协议传输层看下区别：

二者的命令行是不一样的，GET携带了参数，而post没有；

```GET /load_balancer?client_type=1&client_version=300 HTTP/1.1```

``` POST /load_balancer HTTP/1.1 ```

POST消息体有内容，GET方式没有；