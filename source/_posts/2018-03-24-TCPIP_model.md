---
layout: post
category: 计算机网络
date: 2018-03-24
title: TCP/IP 四层模型
description: 层次划分及各层功能
---

　　TCP/IP 协议族最终采纳的是 ARPANET 参考模型，它的结构是 OSI 模型更简单，一般认为因特网协议栈分为四层．<br>
　　通常人们认为 OSI 模型上面最上面三层(应用层，会话层，表示层)在 TCP/IP 中是一个应用层，TCP/IP 有一个相对较弱的会话层，由 TCP 和 RTP 下的打开和关闭连接组成，并且为在 TCP 和 UDP 下的各个应用提供不同的端口号.<br>
　　OSI 模型下层还不具备能够占据真正层的位置的能力，在传输层和网络层之间还需要另外一个层(网络互连层)．

　　以下列出互联网协议套组，工作在不同层的各个协议.

1. 应用层

    BGP, DHCP, DNS, FTP, HTTP, IMAP, LDAP, MGCP, NNTP, NTP, POP, ONC/RPC, RTP, RTSP, RIP, SIP, SMTP, SNMP, SSH, Telnet, TLS/SSL, XMPP(BGP, RIP 这样的路由协议，尽管它们工作在 TCP UDP 上，但是在 TCP/IP 协议族中认为他们工作在网络层)
2. 传输层

    TCP, UDP, DCCP, SCTP, RSVP
3. 网络层

    IPv4, IPv6, ICMP, ICMPv6, ECN, IGMP, OSPF, OSPF, IPsec
4. 连接层

    ARP, NDP, Tunnels(L2TP), PPP, MAC(Etherent, DSL, ISDN, FDDI)

　　TCP/IP 参考模型分层如下，排序由上至下

## 应用层

　　该层包括所有和应用程序协同工作，利用基础网络交换应用程序专用的数据的协议.<br>
　　这个层上运行这一些特定的程序，它们提供用户应用需要的服务，与其对应的协议有 HTTP(超文本传输协议), FTP(文件传输协议), SMTP(简单邮件传输协议), SSH(安全远程登录协议), DNS(域名服务) 等．一旦从应用程序来的数据被编码成一个标准的应用层协议，它将被传送到 IP 栈的下一层.
　　应用层工作的应用通常使用传输层中的 TCP 或 UDP 服务，并且服务器应用常与一个公开的端口号绑定．该端口号由互联网号码分配局正式分配，但是新协议的开发者可以选择自己的端口号.<br>
　　每一个应用层协议一般会使用两个传输层协议之一: 面向连接的 TCP 和无连接的包传输的 UDP，常用的应用层协议有

　　运行在 TCP 上的协议
* HTTP(Hypertext Transfer Protocol, 超文本传输协议)，主要用于普通的浏览网页
* HTTPS(Hypertext Transfer Protocol over Secure Socket Layer, or HTTP over SSl, 安全超文本传输协议)，HTTP 协议的安全版本
* FTP(File Transfer Protocol, 文件传输协议), 用于文件传输
* POP3(Post Office Protocol, 邮局协议), 收邮件用
* SMTP(Simple Mail Transfer Protocol, 简单邮件传输协议), 用于发送电子邮件
* TELNET(Teletype over the NetWork, 网络电传), 通过一个终端登录到网路
* SSH(Secure Shell, 用于替代安全性差的TELNET), 用于加密安全登录

　　运行在 UDP 协议上的协议
* BOOTP(Boot protocol, 启动协议), 应用于无盘设备
* NTP(Network Time Protocol, 网络时间协议), 用于网络同步
* DHCP(Dynamic Host Configuration Protocol, 动态主机配置协议), 动态配置 IP 地址

　　其他
* DNS(Domain Name Service, 域名服务), 用于完成地址查找，邮件转发等工作(运行在 TCP 和 UDP 之上)
* ECHO(Echo Protocol, 回绕协议), 用于查错及测量应答时间(运行在 TCP 和 UDP 之上)
* SNMP(Simple Network Management Protocol, 简单网络管理协议)，用于网络信息的收集和网络管理
* ARP(Address Resolution Protocol, 地址解析协议), 用于动态解析以太网硬件的地址

## 传输层

　　传输层的协议，能够解决诸如端到端的可靠性和保证数据按照正确的顺序到达这样的问题．在 TCP/IP 协议族中，传输协议也包括所给数据应该送给哪个应用程序．<br>
　　TCP 是一个可靠的，面向连接的传输机制，它提供一种可靠的字节流保证数据完整，无损并且按顺序到达．<br>
　　UDP 是一个无连接的数据报协议，它并不检查数据报是否已经到达目的地，并且不保证数据报到达的顺序.

## 网络互联层

　　这一层在 TCP/IP 中称为**网络互联层**，在 OSI 参考模型中对应于网络层.<br>
　　如同在 OSI 参考模型中定义的，网络层解决在一个单一网络上传输数据包的问题，之后发展为将数据从源网络传输至目的网络，这牵扯到在网络组成的网上选择路径将数据包传输．在因特网协议族中，IP 完成数据从源发送至目的的基本人物．

## 网络接口层

　　网络接口层实际上并不是因特网协议族中的一部分，但是它是数据包从一个设备的网络层传输到另外一个设备的网络层的方法．这个过程能够在网卡的软件驱动中控制，也可以在专用芯片中控制．这将完成如添加包头准备发送，通过实体媒介实际发送这样的数据链路功能．<br>
　　