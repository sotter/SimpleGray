---
layout: post
title: Websocket
category: default
---


## Websocket 漫谈
	
> 传统的请求-响应模式的 Web 开发在处理此类业务场景时，通常采用实时通讯方案，常见的是：

>    轮询，原理简单易懂，就是客户端通过一定的时间间隔以频繁请求的方式向服务器发送请求，来保持客户端和服务器端的数据同步。问题很明显，当客户端以固定频率向服务器端发送请求时，服务器端的数据可能并没有更新，带来很多无谓请求，浪费带宽，效率低下。
>    基于 Flash，AdobeFlash 通过自己的 Socket 实现完成数据交换，再利用 Flash 暴露出相应的接口为 JavaScript 调用，从而达到实时传输目的。此方式比轮询要高效，且因为 Flash 安装率高，应用场景比较广泛，但在移动互联网终端上 Flash 的支持并不好。IOS 系统中没有 Flash 的存在，在 Android 中虽然有 Flash 的支持，但实际的使用效果差强人意，且对移动设备的硬件配置要求较高。2012 年 Adobe 官方宣布不再支持 Android4.1+系统，宣告了 Flash 在移动终端上的死亡。

从上文可以看出，传统 Web 模式在处理高并发及实时性需求的时候，会遇到难以逾越的瓶颈，我们需要一种高效节能的双向通信机制来保证数据的实时传输。在此背景下，基于 HTML5 规范的、有 Web TCP 之称的 WebSocket 应运而生。

## Websocket通信协议

### 1. client发起握手请求

```
	GET / HTTP/1.1
	Upgrade: websocket
	Connection: Upgrade
	Host: example.com
	Origin: null
	Sec-WebSocket-Key: sN9cRrP/n9NdMgdcy2VJFQ==
	Sec-WebSocket-Version: 13
```

`Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ== ` 一般来说，server是对client进行验证的，实际上client对server端进行一次验证，验证server是否支持websocket。如果server端不支持websocket，client乱发一气websocket协议的数据导致server crash了那就不好玩了，同时也可能存在某些安全问题。

### 2. server回复握手响应

```
	HTTP/1.1 101 Switching Protocols
	Upgrade: websocket
	Connection: Upgrade
	Sec-WebSocket-Accept: fFBooB7FAkLlXgRSz0BT3v4hq5s=
	Sec-WebSocket-Origin: null
	Sec-WebSocket-Location: ws://example.com/
```

上文中说道client对server进行验证的问题，如果server支持websocket，就会根据websocket的协议算出需要返回客户端的Sec-WebSocket-Accept， client收到后发现这个Sec-WebSocket-Accept是按照ws协议计算出来，就认为server支持ws，握手成功；

计算规则也很简单：Sec-WebSocket-Key 加上 字符串“258EAFA5-E914-47DA-95CA-C5AB0DC85B11”， 然后对新字符串进行SHA-1加密，再进行BASE-64编码就得到了Sec-WebSocket-Accept的值；


### 3. 帧结构

(不管是从客户端到服务端还是相反) 每个数据帧的格式都是：

```
0                  1                  2                  3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|    (7)      |            (16/64)            |
|N|V|V|V|       |S|             |  (if payload len==126/127)    |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|    Extended payload length continued, if payload len == 127   |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                    Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                    Payload Data continued ...                 |
+---------------------------------------------------------------+
```

Server端在完成握手后，后面对于这个socket上面发来的数据就按照ws帧的方式解析了，而不是原来的HTTP协议，双方就可以开始美好的全双工通信之旅了。

在测试时对websocket进行抓包，虽然没有进行SSL加密，但是简单的加密的；如果MASK被置为1，Masking-key被设置为密钥，与原文进行异或编码，所以直接抓包是看不到明文的；

参考  

1. [WebSocket 实战](http://www.ibm.com/developerworks/cn/java/j-lo-WebSocket/index.html)


