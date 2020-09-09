---
title: Tinyhttpd源码分析
date: 2020-09-04 00:46:00
tags: [开源]
categories: 编程
toc: false
---

<img src="https://s1.ax1x.com/2020/09/04/wigODs.jpg" alt="LAN、WLAN、WAN" style="zoom:40%;" />

学习记录，分析Http协议及TinyHttpd的学习过程。

正在学习中~

<!--more-->

#### 1、HTTP协议

>  在开始学习TinyHttpd源码之前，有必要先学习一下http协议，以便对代码有更深入的理解，不然可能会事倍功半。

HTTP协议是一个简单的请求-响应协议，它通常运行在TCP/IP协议之上。它指定了客户端可能发送给服务器什么样的消息以及得到什么样的响应。**请求和响应消息的头**以ASCII码形式给出；而**消息内容**则具有一个类似MIME或XML的格式。

HTTP是基于`C/S模式`，且面向连接的。典型的HTTP事务处理有如下的过程：

>（1）客户与服务器建立连接；
>（2）客户向服务器提出请求；
>（3）服务器接受请求，并根据请求返回相应的文件作为应答
>（4）客户与服务器关闭连接。

在**HTTP1.0**中，客户与服务器之间的HTTP连接是一种一次性连接，它限制每次连接只处理一个请求，当服务器返回本次请求的应答后便立即关闭连接，下次请求再重新建立连接。这种一次性连接主要考虑到web服务器面向的是Internet中成干上万个用户，且只能提供有限个连接，故服务器不会让一个连接处于等待状态，及时地释放连接可以提高服务器的执行效率。而在**HTTP1.1**以后，默认的连接都是keep-alive的，即一次连接可以处理多个请求。

HTTP是一种**无状态协议**，即服务器不保留与客户进行交互时的任何状态。这就大大减轻了服务器负担，从而保持较快的响应速度。

HTTP是一种**面向对象**的协议。允许传送任意类型的数据对象。它通过`数据类型`和`长度`来标识所传送的数据内容和大小，并允许对数据进行压缩传送。

从技术上讲，HTTP服务端在一个特定的TCP端口（默认为80）上打开一个套接字，并且持续监听，等待客户端请求连接。当客户端与服务端建立连接之后，客户端即可通过该连接发送一个包含**请求方法(method)**的数据包，服务端会对该请求做出响应，并通过这条长链路返回给客户端。

HTTP规范定义了**9种**请求方法，每种请求方法规定了客户和服务器之间不同的信息交换方式。

| method  | 描述                                                         |
| ------- | ------------------------------------------------------------ |
| GET     | 请求一个指定资源的表示形式. 使用GET的请求应该只被用于获取数据. |
| HEAD    | 请求一个与GET请求的响应相同的响应，但没有响应体.             |
| POST    | 用于将实体提交到指定的资源，通常导致在服务器上的状态变化或副作用. |
| PUT     | 请求有效载荷替换目标资源的所有当前表示。                     |
| DELETE  | 删除指定的资源。                                             |
| CONNECT | 建立一个到由目标资源标识的服务器的隧道。                     |
| OPTIONS | 用于描述目标资源的通信选项。                                 |
| TRACE   | 沿着到目标资源的路径执行一个消息环回测试。                   |
| PATCH   | 用于对资源应用部分修改。                                     |

常用的请求方法是`GET`和`POST`。服务器将根据客户请求完成相应操作，并以RSP数据包的形式返回给客户，最后关闭连接。

之前我们说过，HTTP是承载于TCP/IP之上的。HTTP请求的内容加上HTTP的头部信息即组成了HTTP的请求报文，也就是TCP报文的数据区部分。

<img src="https://s1.ax1x.com/2020/09/10/w8fhqO.png" alt="HTTP请求报文" style="zoom:40%;" />

同样，下面这张图展示了HTTP响应消息的报文结构。

<img src="https://s1.ax1x.com/2020/09/10/w8f7id.png" alt="HTTP响应报文" style="zoom:40%;" />

举个栗子：

```
###post请求报文格式
POST 　/index.php　HTTP/1.1 
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 5.1; rv:10.0.2) Gecko/20100101 Firefox/10.0.2　　
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: zh-cn,zh;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Referer: http://localhost/
Content-Length：25
Content-Type：application/x-www-form-urlencoded

username=aa&password=1234　　
```

```
###响应报文格式
HTTP/1.1 200 OK　　
Date: Sun, 17 Mar 2013 08:12:54 GMT　　
Server: Apache/2.2.8 (Win32) PHP/5.2.5
X-Powered-By: PHP/5.2.5
Set-Cookie: PHPSESSID=c0huq7pdkmm5gg6osoe3mgjmm3; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Content-Length: 4393
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=utf-8

<html>　　
<head>
<title>xxx<title>
</head>
<body>
Hello world!
</body>
</html>
```



HTTP的报文大致就是这么个样子。如`TinyHttpd`这样的http服务端，它们的功能可以很简要的概况为以下几点：

> 1、监听指定TCP端口，接受用户请求，与用户建立连接。(高并发场景)
>
> 2、接收用户的请求数据包并进行解析。
>
> 3、根据解析得到的结果，判断用户的请求类型，并给出相应的反馈。(主要业务逻辑)

重点的部分其实一个是在于如何能够在高并发的场景下接入大量的用户请求，另一个是对用户的不同请求做出相应的反馈。例如用户请求显示一个网页，这时就需要去读取网页相应的内容发送给客户，或者用户请求写入某个数据，这时就需要读取用户写入的数据，并存入文件或者数据库当中。具体的业务逻辑一般是在上面的第3点当中实现的，像TinyHttpd这样的示例程序并没有复杂的逻辑在里面，主要还是展示了如何对用户的请求进行处理的过程。

以上就是对HTTP协议及其服务端功能的简要介绍，了解了这些之后，就可以开始学习TinyHttpd的源码了。



参考文献：

[1] [HTTP协议理解及服务端与客户端的设计实现](https://blog.csdn.net/yanzhenjie1003/article/details/93098495)(引用了两张关于HTTP报文结构的图片)



