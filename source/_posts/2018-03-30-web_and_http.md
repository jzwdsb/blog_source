---
layout: post
category: 计算机网络
date: 2018-03-30
title: 应用层协议笔记（２）
description: web 和 http
---

## 概况

　　Web 的应用层协议是**超文本传输协议**(HyperText Transfer Protocol, HTTP).<br>
　　HTTP 由两个程序实现：一个客户程序和一个服务器程序．客户程序和服务器程序运行在不同的端系统中，通过交换 HTTP 报文进行会话.通常 Web 浏览器实现了 HTTP 的客户端，Web 服务器实现了 HTTP 的服务器端.

　　HTTP 使用 TCP 作为它的支撑运输协议．HTTP 客户端首先发起一个与服务器的 TCP 连接，一旦链接建立，该浏览器和服务器进程就可以通过套接字接口访问 TCP.<br>
　　客户向他的套接字接口发送 HTTP 请求报文和从他的套接字接口接收 HTTP 响应报文，类似的，服务器从他的套接字接口接收 HTTP 请求报文并发送响应报文.

　　服务器想客户发送被请求的文件，而不存储任何关于客户的状态信息．所以说 HTTP 是一个**无状态协议**(Stateless protocol)．

## 非持续连接和持续连接

　　非持续连接指的是每个请求/响应对经一个单独的 TCP 连接发送．<br>
　　持续连接指的是所有的请求及其相应经相同的 TCP 连接发送，HTTP 默认是使用持续连接的.

## HTTP 报文格式
　
　　HTTP 报文包括以下部分

* 起始行(request line)

    报文的第一行就是起始行，在请求报文中用来说明要做什么，在响应报文中说明出现了什么情况
* 首部字段(header line)

    起始行后面有零个或多个首部字段，每个首部字段都包含一个名字和一个值，为了便于解析，两者之间用冒号(:) 分割．<br>
    首部之间用 CRLF 分割，每个首部字段之后都有一个 CRLF 表示结束．首部的结束是一个 CRLF 结束，也就是说当遇到连续两个 CRLF 表示首部已结束.
* 主体(entity body)

    空行之后就是可选的报文主题，其中包含了所有类型的数据．请求主体中包括要发送给 Web 服务器的数据；响应主体中装载了要返回给客户端的数据．起始行和首部都是文本形式且都是格式化的，而主体不同，主体中可以包含任意的二进制数据(比如图片，视频，音轨，软件程序).主体中也可以包含文本.


### HTTP 请求报文

　　以下是一个典型的 HTTP 请求报文

```
GET /somedir/page.html HTTP/1.1
Host: www.someschool.edu
Connection: close
User-agent: Mozilla/5.0
Accept-language: fr
```

　　HTTP 请求报文格式如下图

![HTTP_request_header](/downloads/request_header.png)

### HTTP 响应报文

　　以下是一个典型的 HTTP 响应报文

```
HTTP/1.1 200 OK
Connection: close
Date: Tue, 09 Aug 2011 15:44:04 GMT
Server: Apache/2.2.3 (CentOS)
Last-Modified: Tue, 09 Aug 2011 15:11:03 GMT
Content-Length: 6821
Content-Type: text/html
```

　　HTTP 响应报文格式如下图

![HTTP_response_header](/downloads/response_header.jpeg)

　　起始行的状态码和对应的短语指示了请求的结果，一些常见的状态码和相关的短语包括:
* 200 OK: 请求成功，信息在返回的相应报文中
* 301 Moved Permanently: 请求的对象已经被永久转移了，新的 URL 定义在响应报文中的 Location: 首部行中．客户软件自动获取新的 URL
* 400 Bad Requset: 一个通用差错代码，指示该请求不能被服务器理解．
* 404 Not Found: 被请求的文档不在服务器上.
* 505 HTTP Version Not Supported: 服务器不支持请求报文的使用的 HTTP 协议版本


## Cookie

　　HTTP 服务器是无状态的，然而一个 Web 服务器通常希望能够识别用户．为此，HTTP 使用了 cookies.<br>
　　cookies 有 4 个组件:
1. HTTP 响应报文中的一个 cookies 首部行
2. HTTP 请求报文中的一个 cookies 首部行
3. 在用户端系统中保留有一个 cookies 文件，并由用户的浏览器进行管理.
4. 位于 Web 服务器的一个后端数据库

## Web 缓存

　　Web 缓存器(Web cache)也叫代理服务器(proxy server)，它是能够代表初始 Web 服务器来满足 HTTP 请求的网络实体.<br>
　　当配置用户的浏览器对某个对象的请求首先定向到该 Web 服务器时，将会发生如下情况

1. 浏览器建立一个到 Web 缓存器的 TCP 连接，并向 Web 缓存器中的对象发送一个 HTTP 请求.
2. Web 缓存器进行检查，查看本地是否存储了该对象副本．如果有，Web 缓存器就向客户浏览器用 HTTP 响应报文返回该对象.
3. 如果 Web 缓存器中没有该对象，它就打开一个与该对象的初始服务器的 TCP 连接．Web 缓存器则在这个连接上发送一个对该对象的 HTTP 请求．在收到该请求后，初始服务器向 Web 缓存器发送具有该对象的 HTTP 响应.
4. 当 Web 缓存器接收到该对象时，它在本地存储空间存储一份副本，并向客户浏览器用 HTTP 响应报文发送该副本.

## 条件 GET 方法

　　如果请求报文使用 GET 方法，并且请求报文中包含一个 `If-Modified-Since` 首部行，那么这个 HTTP 请求报文就是一个条件 GET 方法.