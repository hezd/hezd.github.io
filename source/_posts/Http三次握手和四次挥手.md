---
title: Http三次握手和四次挥手
date: 2023-11-22 17:47:58
tags:
- 三次握手
- 四次挥手
categories:
- 网络
---

### Http三次握手和四次挥手

了解Http的三次握手和四次挥手有助于对Http请求过程有更深刻的认识，同时也可以解决面试相关问题😁



### 三次握手

第一次握手：客户端想服务器发起请求连接的SYN报文，发送完毕后进入SYN_SENT状态

第二次握手：服务器收到客户端请求后，向客户端发送SYN ACK报文，进入SYN_RCVD状态

第三次握手：客户端收到服务器SYN ACK报文后，向服务器发送ACK报文，当服务器收到客户端ACK报文后客户端和服务器都会进入ESTABLISHED状态，TCP连接成功

三次握手示意图

![](https://github.com/hezd/Image-Folders/blob/master/Blog/http_three_shake.png?raw=true)

### 四次挥手

当需要关闭http连接时需要经历四次挥手

第一次挥手：客户端向服务器发送FIN报文，进入FIN_WAIT状态

第二次挥手：服务器收到客户端FIN报文后向客户端发送ACK报文，进入CLOSE_WAIT状态

第三次挥手：当服务器没有数据可发送准备好关闭连接后向客户端发送FIN报文，进入LAST_ACK状态

第四次挥手：客户端收到服务端FIN报文后向服务端发送ACK报文进入TIME_WAIT状态，服务器收到客户端ACK报文后关闭连接，客户端在等待2MSL（最大报文生存时间）时间后也关闭连接

四次挥手示意图

![](https://github.com/hezd/Image-Folders/blob/master/Blog/http_four_wave.png?raw=true)

到这里三次挥手和四次握手已经讲解完了
