---
title: Tinyhttpd源码分析
date: 2020-09-04 00:46:00
tags: [开源]
categories: 编程
toc: false
---

<img src="https://s1.ax1x.com/2020/09/04/wigODs.jpg" alt="LAN、WLAN、WAN" style="zoom:40%;" />

学习记录，分析Http协议及TinyHttpd的学习过程。

TinyHttpd是一个简单的单文件http服务端程序，能够展示http服务器与客户端之间的信令交互流程。

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

#### 二、TinyHttpd源码分析

TinyHttpd是一个简单的http服务器实现，只有一个C文件和一个头文件。除了main函数外，共有12个函数，以下是函数声明。

```C
void  accept_request(int);       									/*接收接入请求*/
void  bad_request(int);												/*返回400 Bad Request*/
void  cat(int, FILE *);												/*读取本地文件，返回给客户端*/
void  cannot_execute(int);											/*管道或fork出现错误，返回500 Internal Server Error*/
void  error_die(const char *);										/*打印错误，并退出程序*/
void  execute_cgi(int, const char *, const char *, const char *);   /*执行CGI脚本*/
int   get_line(int, char *, int);									/*从socket输出缓冲中读取一行*/
void  headers(int, const char *);									/*返回200 OK*/
void  not_found(int);												/*返回404 NOT FOUND*/
void  serve_file(int, const char *);   								/*为客户端返回文件*/
int   startup(u_short *);    										/*启动服务*/
void  unimplemented(int);											/*请求方法错误，返回501 Method Not Implemented*/
```

几个主要的函数如下，这几个函数构成了服务器的主要功能：

```C
void  accept_request(int);       									/*接收接入请求*/
void  execute_cgi(int, const char *, const char *, const char *);   /*执行CGI脚本*/
void  serve_file(int, const char *);   								/*为客户端返回文件*/
int   startup(u_short *);    										/*启动服务*/
```

代码阅读流程。

`main()`->`startup()`->`accept_request()`->`serve_file()`->`execute_cgi()`

以下进行拆分讲解，部分代码中应用到的小知识点会附在每部分的最后。

#### 1、main()

main()函数的功能主要是在指定的port端口上启动服务(或者随机分配端口)，之后进入死循环，阻塞在accept()。直到有用户请求接入，此时接收请求之后会为该用户创建一个accept_request()线程处理这次请求。以下是代码。

```C
int main(void)
{
    int server_sock = -1;
    u_short port = 0;
    int client_sock = -1;
    struct sockaddr_in client_name;
    int client_name_len = sizeof(client_name);
    pthread_t newthread;

    /*在指定port上监听，port=0时随机分配端口，并将端口号赋值给port*/
    server_sock = startup(&port);
    printf("httpd running on port %d\n", port);
    
    while (1)
    {
        /*accept接入请求*/
        client_sock = accept(server_sock, (struct sockaddr *)&client_name, &client_name_len);
        if (client_sock == -1)
            error_die("accept");
        /*为每个接入的用户创建一个线程处理请求*/
        if (pthread_create(&newthread, NULL, accept_request, client_sock) != 0)
            perror("pthread_create");
    }
    close(server_sock);
    return (0);

}
```

这部分没啥好说的，标准的服务端监听流程。

#### 2、startup()

再回过头看下main函数中的`startup()`，主要功能是创建套接字，并绑定指定端口，并且开始`listen()`。

```C
int startup(u_short *port)
{
    int httpd = 0;
    struct sockaddr_in name;

    httpd = socket(PF_INET, SOCK_STREAM, 0); /* PF_INET等价于AF_INET */
    if (httpd == -1)
        error_die("socket");
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;
    name.sin_port = htons(*port);
    name.sin_addr.s_addr = htonl(INADDR_ANY); /* 监听本机所有ip的接入请求 */
    if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0)
        error_die("bind");
    if (*port == 0) /* 小知识点：bind的时候未指定端口会随机分配一个 */
    {
        int namelen = sizeof(name);
        if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1) /* getsockname可以获取指定句柄的addr和len信息 */
            error_die("getsockname");
        *port = ntohs(name.sin_port);/* 将端口号写回port */
    }
    if (listen(httpd, 5) < 0) /* 开始监听 */
        error_die("listen");
    return (httpd);

}
```

这里有个小技巧，`bind()`一个`sockaddr`的时候，可以不指定端口，这时系统会自动分配一个未使用的端口号。如果想要知道系统为你分配了哪个端口号，可以通过`getsockname()`函数来获取指定socket上的`sockaddr`和`socklen`。

#### 3、accept_request()

`accept_request()`是一个比较重要的函数，它是在每个用户接入后，主进程为每个用户分配的处理线程入口函数。

在这个函数中，将会对用户的请求类型进行判断，如果不是GET或POST请求，则会拒绝服务。通过请求的方法和URL来判断用户是动态请求还是静态请求，设置`cgi`标志位。提取用户请求URL中的路径信息，为用户返回请求的文件。最后，通过`cgi`标志位决判断请求类型，如果是静态请求，则直接返回请求URL中路径对应的文件，如果是动态请求，则执行CGI脚本。

```C
void accept_request(int client)
{
    char buf[1024];
    int  numchars;
    char method[255];
    char url[255];
    char path[512];
    size_t i, j;
    struct stat st;
    int cgi = 0; /* cgi=1表示需要执行cgi脚本 */
    char *query_string = NULL;

    /*读取第一行数据*/
    numchars = get_line(client, buf, sizeof(buf));
    i = 0;
    j = 0;
    while (!ISspace(buf[j]) && (i < sizeof(method) - 1))
    {   /*获取method*/
        method[i] = buf[j];
        i++;j++;
    }
    method[i] = '\0';
    
    /*不是GET和POST请求则拒绝服务*/
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        unimplemented(client);
        return;
    }
    /* POST请求需要执行CGI脚本 */
    if (strcasecmp(method, "POST") == 0)
        cgi = 1;
    
    i = 0;
    while (ISspace(buf[j]) && (j < sizeof(buf)))
        j++;
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
    {   /*获取URL*/
        url[i] = buf[j];
        i++;j++;
    }
    url[i] = '\0';
    
    /*GET请求中有query_string字段时执行CGI脚本，否则为静态请求*/
    if (strcasecmp(method, "GET") == 0)
    {
        query_string = url; /*从URL中提取query_string*/
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?')/*URL中包含‘？’为动态请求*/
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }
    
    sprintf(path, "htdocs%s", url); /*拼接文件路径*/
    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    if (stat(path, &st) == -1)/*文件或者路径不存在*/
    {
        while ((numchars > 0) && strcmp("\n", buf)) /* 将头部信息读取并且丢弃 */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client); /*返回404*/
    }
    else
    {
        if ((st.st_mode & S_IFMT) == S_IFDIR) /*PATH为路径*/
            strcat(path, "/index.html");/*默认访问index.html*/
        if ((st.st_mode & S_IXUSR) ||(st.st_mode & S_IXGRP) ||(st.st_mode & S_IXOTH)) /*对该路径指向的文件具有执行权限*/
            cgi = 1;
        if (!cgi)
            serve_file(client, path); /*静态请求*/
        else
            execute_cgi(client, path, method, query_string); /*动态请求*/
    }
    
    close(client);

}
```

对HTTP服务器的请求可以分为`动态请求`和`静态请求`。**静态请求**的典型例子就是html，即服务器可以直接返回的就是静态请求。

而**动态请求**的则不是独立存在于服务器上的一个网页文件，只有当用户发起请求时才能返回一个完整的网页，网页的部分内容存储在例如数据库中或是别的服务器上，根据用户的不同请求可以返回不同的内容。

> 一般在URL当中包含`？`的都是动态链接。

这里用到的`stat()`是一个获取指定路径文件状态的函数。它的函数原型为：

```C
int stat(const char * path, struct stat *st);
```

`path`可以是一个文件或者路径，`st`是一个stat结构体的指针。`struct stat`用于存放`stat()`执行的结果，与`path`相关的所有文件信息均存放在该结构体中。该结构体的成员如下：

```C
struct stat {
    dev_t st_dev; 				//文件的设备编号
    ino_t st_ino; 				//文件的i-node
    mode_t st_mode;				//文件的类型和存取的权限
    nlink_t st_nlink; 			//连到该文件的硬连接数目, 刚建立的文件值为1.
    uid_t st_uid; 				//文件所有者的用户识别码
    gid_t st_gid; 				//文件所有者的组识别码
    dev_t st_rdev; 				//若此文件为装置设备文件, 则为其设备编号
    off_t st_size; 				//文件大小, 以字节计算
    unsigned long st_blksize; 	//文件系统的I/O 缓冲区大小.
    unsigned long st_blocks;	//占用文件区块的个数, 每一区块大小为512 个字节.
    time_t st_atime; 			//文件最近一次被存取或被执行的时间
    time_t st_mtime; 			//文件最后一次被修改的时间, 一般只有在用mknod、 utime 和write 时才会改变
    time_t st_ctime; 			//最近一次被更改的时间, 此参数会在文件所有者、组、 权限被更改时更新
};
```

这里用到的`st_mode`用于描述文件的类型和相应的权限信息。`st_mode`其实本质就是一个`unsigned int`，最低的9位(0-8)是权限，9-11是id，12-15是类型。`S_IFMT`是掩码，`st_mode & S_IFMT`即可得到文件类型。`st_mode`的具体各位信息如下:

```
S_IFMT     0170000   bitmask for the file type bitfields
S_IFSOCK   0140000   socket
S_IFLNK    0120000   symbolic link
S_IFREG    0100000   regular file
S_IFBLK    0060000   block device
S_IFDIR    0040000   directory
S_IFCHR    0020000   character device
S_IFIFO    0010000   fifo
S_ISUID    0004000   set UID bit
S_ISGID    0002000   set GID bit (see below)
S_ISVTX    0001000   sticky bit (see below)
S_IRWXU    00700     mask for file owner permissions
S_IRUSR    00400     owner has read permission
S_IWUSR    00200     owner has write permission
S_IXUSR    00100     owner has execute permission
S_IRWXG    00070     mask for group permissions
S_IRGRP    00040     group has read permission
S_IWGRP    00020     group has write permission
S_IXGRP    00010     group has execute permission
S_IRWXO    00007     mask for permissions for others (not in group)
S_IROTH    00004     others have read permission
S_IWOTH    00002     others have write permisson
S_IXOTH    00001     others have execute permission
```

#### 4、serve_file()

`serve_file()`的功能主要是调用`cat()`函数将客户端请求的静态文件从磁盘中读取出来，返回给客户端。

```C
void serve_file(int client, const char *filename)
{
    FILE *resource = NULL;
    int numchars = 1;
    char buf[1024];

    buf[0] = 'A';
    buf[1] = '\0';
    while ((numchars > 0) && strcmp("\n", buf)) /* 丢弃头部信息 */
        numchars = get_line(client, buf, sizeof(buf));
    
    resource = fopen(filename, "r"); /*以读的方式打开文件*/
    if (resource == NULL)
        not_found(client);
    else
    {
        headers(client, filename);/*返回200 OK*/
        cat(client, resource);/*为服务器返回静态文件*/
    }
    fclose(resource);

}
```

这部分也没啥好说的。

#### 5、execute_cgi()

`execute_cgi()`函数的主要功能是创建一个子进程及一对读写管道，子进程负责执行CGI脚本，父进程负责写入请求并读取结果返回给客户端。

```C
void execute_cgi(int client, const char *path, const char *method, const char *query_string)
{
    char buf[1024];
    int cgi_output[2];
    int cgi_input[2];
    pid_t pid;
    int status;
    int i;
    char c;
    int numchars = 1;
    int content_length = -1;

    buf[0] = 'A'; //?
    buf[1] = '\0';
    if (strcasecmp(method, "GET") == 0)             /*忽略大小写比较字符串*/
        while ((numchars > 0) && strcmp("\n", buf)) /* get_line读取，1、socket缓冲区无内容，2、读取到空行。功能：读取头部信息，并且丢弃 */
            numchars = get_line(client, buf, sizeof(buf));
    else /* POST */
    {
        numchars = get_line(client, buf, sizeof(buf));
        while ((numchars > 0) && strcmp("\n", buf))
        {
            buf[15] = '\0';
            if (strcasecmp(buf, "Content-Length:") == 0) /*读取Content-Length行（请求包头部分，可能有若干行，但这里只读取该行）*/
                content_length = atoi(&(buf[16]));
            numchars = get_line(client, buf, sizeof(buf));
        }
        if (content_length == -1)
        {
            bad_request(client); /*返回 400 Bad Request*/
            return;
        }
    }
    
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    send(client, buf, strlen(buf), 0); /*返回200 OK*/
    
    if (pipe(cgi_output) < 0)
    {
        cannot_execute(client);
        return;
    }
    if (pipe(cgi_input) < 0) /*创建一对pipe*/
    {
        cannot_execute(client);
        return;
    }
    
    /*注意：fork()可以创建一个与父进程几乎完全一样的子进程，但是只会拷贝调用fork的线程，其他的线程不会拷贝。*/
    if ((pid = fork()) < 0) /*创建一个子进程*/
    {
        cannot_execute(client);
        return;
    }
    
    if (pid == 0) /* child: CGI script */
    {             /*pid==0标志此进程为子进程，在子进程中执行CGI脚本*/
        /* 环境变量 */
        char meth_env[255];
        char query_env[255];
        char length_env[255];
    
        /* dup和dup2函数的功能时重定向文件描述符 */
        /* dup2(oldfd,newfd)可以将newfd重定向到oldfd */
        dup2(cgi_output[1], 1); /* 将标准输出重定向到输出管道上 */
        dup2(cgi_input[0], 0);  /* 将标准输入重定向到输入管道上 */
        /* 结合父进程代码可知，cgi_input管道：父进程->子进程，cgi_output管道：子进程->父进程 */
        /* 即input和output是相对于父进程而言的 */
        close(cgi_output[0]); /*子进程关闭输出管道的输出端*/
        close(cgi_input[1]);  /*子进程关闭输入管道的写入端*/
        sprintf(meth_env, "REQUEST_METHOD=%s", method);/*设置环境变量*/
        putenv(meth_env);
        if (strcasecmp(method, "GET") == 0)
        {
            sprintf(query_env, "QUERY_STRING=%s", query_string);
            putenv(query_env);
        }
        else
        { /* POST */
            sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
            putenv(length_env);
        }
        execl(path, path, NULL);/*调用execl()函数执行cgi脚本*/
        exit(0);
    }
    else
    { /* parent */
        close(cgi_output[1]); /*父进程关闭输出管道的写入端*/
        close(cgi_input[0]);  /*父进程关闭输入管道的输出端*/
        if (strcasecmp(method, "POST") == 0)
            for (i = 0; i < content_length; i++)
            { /*POST    将从客户端接收的字符写入cgi_input管道中*/
                recv(client, &c, 1, 0);
                write(cgi_input[1], &c, 1);
            }
        /* 从cgi_output管道中读取的数据发送给客户端 */
        while (read(cgi_output[0], &c, 1) > 0)
            send(client, &c, 1, 0);
    
        close(cgi_output[0]);
        close(cgi_input[1]);
        waitpid(pid, &status, 0);
    }

}
```

PIPE的创建和关闭，如何组成一对输入输出管道

`execute_cgi()`函数首先创建了一对管道`cig_input`和`cig_output`，之后调用`fork()`创建了一个子进程。需要指出的是，主进程当中只创建了`accept_request()`这一个线程，如果是在多线程的环境中，只有调用`fork()`的那个线程才会存活下来，其他的线程都会消失。因此在多线程环境当中需要注意`fork()`的使用。当然，这里使用`fork()`并没有啥影响，因为也有且只有一个线程存在。

调用`fork()`后可以通过返回值来区分父子进程，父进程中`fork()`的返回值为子进程的PID，而子进程的`fork()`返回值为0。因此，我们可以很好的区分父子进程，在子进程中将`cig_input`的输入端关闭，`cig_output`的输出端关闭，在父进程中将`cig_input`的输出端关闭，`cig_output`的输入端关闭，以此组成一对管道。管道的具体概念可以参见另一篇讲解UNIX进程间通信的blog。

> `cig_input`对于父进程来说是输入管道，`cig_output`是输出管道。

同时，在子进程当中，调用`dup2()`函数将`stdin`合`stdout`的文件描述符重定向到对应的管道当中，这样即可直接对管道进行读写操作。

至于CGI脚本的内容并没有细究，毕竟也不是做这个方向的。

参考文献：

[1] [HTTP协议理解及服务端与客户端的设计实现](https://blog.csdn.net/yanzhenjie1003/article/details/93098495)



