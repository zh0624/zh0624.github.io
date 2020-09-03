---
title: I/O多路复用之epoll
date: 2020-08-10 23:45:00
tags: [同步,操作系统]
categories: 编程
toc: true
---

<img src="https://s1.ax1x.com/2020/08/11/aq5Z0f.jpg" alt="epoll" style="zoom:40%;" />

I/O多路复用是通过一种机制，让进程可以同时监听多个文件描述符，当某个文件描述符符合某种状态时，能够通知到相应的进程采取对应的操作。多路复用机制发展至现在，常用的有select、poll、epoll三种方式，它们的发展历程是一个不断优化的过程。本文首先简要介绍多路复用机制的概念，再简单描述下它的发展例程，会着重介绍epoll的机制及部分实现。文中内容提炼了不少前人梳理的结果，并加以自己的理解进行阐述。

<!--more-->

#### 1、阻塞、非阻塞、同步、异步

阻塞、非阻塞、同步、异步，这几个概念一开始就曾让我困惑了很久。阻塞和同步不是一个意思嘛？非阻塞不就是异步嘛？为啥还搞这些乱七八糟花里胡哨的。为了弄懂这些名词的区别，我开始在网上查阅资料，在知乎上看到一个解释，感觉十分清晰，这里分享一下。

同步与异步，关注的是消息通信的机制，**重点在于通信机制的不同**。

同步指的是，在消息交互的过程当中，我给你打了一个电话，在你给我回复之前，我要一直把电话开着，等你给我回复，直到你给我回消息之后，我才会挂断电话。用专业术语来说就是，**在发出一个调用时，在没有得到结果之前，该调用就不返回。但是一旦调用返回，就得到返回值了。**结合例子，做出解释就是，在发出一个调用(我给你打电话)时，在没有得到结果(你给我回复我想要的消息)之前，该调用就不返回(我不会挂断电话)。但是一旦调用返回(我挂掉电话)，就得到返回值了(意味着我得到了我想要的消息)。

异步指的是，在消息交互的过程当中，我给你发一个信息，我就先去干别的事情了，等你给我回复信息之后，我再来看你回的消息。用专业术语来说就是，**当一个异步过程调用发出后，调用者不会立刻得到结果。而是在调用发出后，被调用者通过状态、通知来通知调用者，或通过回调函数处理这个调用。**在结合例子解释一下，就是说，当一个异步过程调用(发信息)发出后，调用者(我)不会立刻得到结果。而是在调用发出后，被调用者(你)通过状态、通知(给我回消息)来通知调用者，或通过回调函数处理这个调用。

注意到了没有，区别在哪里，就是在于这个**机制**的不同，**同步类比于电话，异步类比于短信**，两种不同的消息通信机制。我给你打电话，如果你没给我回复时，我就挂了电话，那么我就无法获得想要的信息了。而我给你短信，在我发出去之后，我就可以不管了，等你给我回复了，我再过来看你回复的消息。

阻塞与非阻塞，关注的是在等待调用结果时的一个状态，**重点是在等待期间的状态的不同**。

阻塞指的是，当我在等你给我回复之前，我啥也不干，就在那干等着，直到你给我回复之后，我才去做别的事情。用专业术语来说就是，**阻塞调用是指调用结果返回之前，当前线程会被挂起，调用线程只有在得到结果之后才会返回。**

而非阻塞指的是，在我等你给我回复之前，我可以做一些别的事，不用在那里干等着。用专业术语来解释就是，**非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。**

现在你来思考一下，能否区分出阻塞与同步，非阻塞与异步？

如果还有一点点晕，再仔细的阅读一下之前的同步与异步的例子，推敲一下阻塞与非阻塞的场景，与之有何不同。

好的，其实阻塞和同步之所以容易混淆，是因为他们似乎都是有一个`等待`的过程，不过之前在解释什么是同步的时候，已经指出了它的重点在于同步是一种消息通信的机制，它的重点在机制上。

什么样的场景是同步阻塞呢，比如我给你打电话，你没有给我回消息，我保持电话畅通，这时候来了个快递，我为了等你就没管，然后你给我回复了消息，我听到后就挂断了电话，再去签收了快递。

而同步非阻塞呢，同样类似上面的例子，我给你打电话，你没有给我回消息，我保持电话畅通，这时候来了个快递，我顺手就签收了，然后你给我回复了消息，我听到后就挂断了电话。

这样是不是就理解了何为阻塞，何为同步。**阻塞是一种状态**，当你阻塞时，你不可以干别的事，只能等，**而同步是一种机制**，当你使用了同步的机制的时候，你没有获取到结果时，你就要保持住，不能够返回。至于你在保持的期间，啥也不干，还是做些别的事，这不是同步机制所关心的。

异步其实类似，这里不再赘述。



最后用那个帖子里的一个回复来说明下问题吧，假设你要买一本书，现在你要问老板有没有，这时候你要去问下老板，于是乎，有了以下这几种情况：
>  **同步阻塞**：你打电话问老板有没有某书，老板去查，在老板给你结果之前，你一直拿着电话等待老板给你结果，你此时什么也干不了。
>  **同步非阻塞**：你打电话过去后，在老板给你结果之前，你拿着电话等待老板给你结果，但是你拿着电话等的时候可以干一些其他事，比如嗑瓜子。
>  **异步阻塞**：你打电话过去后，老板去查，你挂掉电话，等待老板给你打电话通知你，这是异步，你挂了电话后还是啥也干不了，只能一直等着老板给你打电话告诉你结果，这是阻塞。
>  **异步非阻塞**：你打电话过去后，你就挂了电话，然后你就想干嘛干嘛去。只用时不时去看看老板给你打电话没。



#### 2、IO模型

聊完前面的几个概念之后，我们再来看IO模型的时候理解起来就比较轻松了。常见的几种IO模型有：

>  传统IO——同步阻塞IO
>
>  NONBLOCK的socket——同步非阻塞IO
>
>  IO多路复用(Reactor设计模式)——异步阻塞IO
>
>  Proactor设计模式——异步非阻塞IO

我们这里讨论的就是IO多路复用，也就是epoll，当然还包括select和poll这两个前辈，这三个IO多路复用机制都是属于异步阻塞IO。

##### read()

首先，让我们从最原始的IO模型说起，也就是`read()`。read的函数原型如下：

```C
size_t read(int fd, void *buf, size_t count)
```

read函数向操作系统发起一个IO申请，申请从指定的文件描述符fd中读取数据，存放至buf缓冲区当中，每次最多读取count个字节。read这个函数在文件操作和socket编程中常常会使用到。

例如在socket通信中，如果我们使用默认的socket，那么在发起read请求之后，在尚未有数据通过网络传输至网口上时，进程会一直阻塞。假设此时有数据通过网络传输至网口上时，网口会产生硬件中断，通知CPU有数据到达，此时CPU会进行上下文切换，陷入网卡的中断处理程序当中，将网卡内的数据读取至内存当中。将数据拷贝至内存中后，CPU退出中断处理程序，告诉read函数socket的输出缓冲区中已经有数据了，此时read将数据从内核中拷贝至进程的地址空间(buf缓冲区)当中，并返回读取到的字节数。这就是一次read的全部流程。

`read()`是最为简单的一种IO模型，使用起来很简单，但是问题也很明显，那就是他一次只能监听一个文件描述符，如果我要监听多个文件描述符，那么我就得创建多个线程去分别监听不同的文件描述符，这无疑是一件十分浪费资源的事。

##### select()

因此，`select()`出现了。它的函数原型如下：

```C
int select(int nfds, fd_set *readfds, fd_set *writefds, 
           fd_set *exceptfds, struct timeval *timeout);
```

`select()`函数相比于`read()`或者`recv()`这种IO最大的优点就是，它允许进程监视多个文件描述符！

当调用select()函数后，进程会进入阻塞状态，等待所监视的一个或者多个文件描述符变为ready状态。所谓的ready状态是指：文件描述符不再是阻塞状态，可以用于某类IO操作了，包括**可读**、**可写**、**发生异常**三种。

nfds表示操作系统需要监听的文件描述符个数，这里实际上它的值应该为你所需要监听的最大的文件描述符+1，也就是说实际上操作系统监听了0~max_fd+1之间的所有文件描述符。select采用的是轮询机制，它会一直去检测fd_set中被监视的文件描述符是否有变为ready状态的，当其中的一个或者多个变为ready状态后，select函数会返回ready状态的文件描述符的个数。

`select()`解决了read和recv一次只能监听一个文件描述的问题，但是单个进程所能监听的文件描述符是有限的，并且采用了轮询机制，时间复杂度为O(n)，当文件描述符越多的时候，它的性能也就越差，并且会频繁的产生内核态与用户态之间的切换。

##### poll()

`poll()`其实与`select()`没有太大的区别，它的原型如下：

```c
int poll(struct pollfd fds[], nfds_t nfds, int timeout)；
```

poll和select实现功能差不多，但poll效率高，它通过一个结构体数组存放需要检测其状态的Socket描述符；每当调用这个函数之后，系统不会清空这个数组，操作起来比较方便；特别是对于socket连接比较多的情况下，在一定程度上可以提高处理的效率；这一点与select()函数不同，调用select()函数之后，select()函数会清空它所检测的socket描述符集合，导致每次调用select()之前都必须把socket描述符重新加入到待检测的集合中；因此，select()函数适合于只检测一个socket描述符的情况，而poll()函数适合于大量socket描述符的情况。

##### epoll()

鉴于`poll()`和`select()`存在的诸多问题，Linux为处理大批量文件描述符，对`poll()`进行改进后用`epoll()`取而代之。`epoll()`对于文件描述符的处理方式与前两个策略有所不同： `select()`在底层是用的数组去存储文件描述符，`poll()`则是用链表来存储，而epoll选择了用**红黑树**来存储文件描述符。

epoll()的三个主要函数：

```C
/*创建一个epoll文件描述符--epfd，其中size目前无实际意义，取址大于0即可*/
int epoll_create(int size) ； 
/* epfd的控制函数，用于增加/删除epoll句柄上监听的文件描述符，或者修改其模式。
op为epoll_ctl的动作，有三种：EPOLL_CTL_ADD、 EPOLL_CTL_MOD、EPOLL_CTL_DEL。
fd为被增加/删除/修改的文件描述符
event为描述需要监听事件的结构体。*/    
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event) ； 
/*等待epfd上所有监听的文件描述符，直到有事件触发或timeout超时，timeout=0立即返回，timeout=-1阻塞*/
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

epoll使用红黑树去**监听**并**维护**所有文件描述符，在调用`epoll_create()`时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后`epoll_ctl()`传来的socket外，**还会再建立一个list链表，用于存储准备就绪的事件。**当调用`epoll_wait()`时，仅仅观察这个链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后，即使链表没数据也返回。所以，`epoll_wait()`非常高效。

当我们执行`epoll_ctl()`添加文件描述符时时，除了把fd放到epoll在文件系统里的对象上对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，这个回调函数的功能是告诉内核，如果这个文件描述符的中断到了，就把它放到准备就绪链表里。所以，epoll()可以从就绪链表中，只对活跃或者说就绪状态的文件描述符调用回调函数，这可以显著的提升它的效率。因此，epoll()返回就绪的文件描述符的时间复杂度为O(1)。当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后，就把对应socket的fd插入到准备就绪链表里了。通常情况下即使我们要监控数以百万计的文件描述符，一次也只返回很少量“就绪”的文件描述符而已，因此，`epoll_wait()`仅需要从内核态拷贝少量的文件描述符到用户态，这是它相比于其他IO多路复用机制的一个显著优点。

`epoll()`对于打开的文件描述符没有限制，一般是基于系统内存，而这个数字一般远大于1024。

然而，epoll相比于select并不是在所有情况下都要高效，例如在如果有少于1024个文件描述符监听，且大多数socket都是出于活跃繁忙的状态，这种情况下，select要比epoll更为高效，因为epoll会有更多次的系统调用，用户态和内核态会有更加频繁的切换。

最后说下`epoll()`的2种工作方式：LT和ET。

1. LT(level triggered)，也就是水平触发，是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。

2. ET (edge-triggered)，也就是边缘触发，是高速工作方式，只支持non-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过`epoll()`告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了。但是请注意，如果一直不对这个fd作IO操作，内核不会发送更多的通知，也就是说，内核只会通知你一次。

ET和LT的区别就在这里体现，LT事件不会丢弃，而是只要读buffer里面有数据可以让用户读，则不断的通知你。而ET则只在事件发生之时通知。

总结，`epoll()`高效的本质在于：

> 减少了用户态和内核态的文件句柄拷贝
>
> 减少了对可读可写文件句柄的遍历
>
> mmap 加速了内核与用户空间的信息传递，epoll是通过内核与用户mmap同一块内存，避免了无谓的内存拷贝
>
> IO性能不会随着监听的文件描述的数量增长而下降
>
> 使用红黑树存储fd，以及对应的回调函数，其插入，查找，删除的性能不错，相比于hash，不必预先分配很多的空间

#### 3、epoll例程解析

以epoll三个基本函数实现了一个回声服务端+客户端的小例程。

功能是将客户端输入的字符转换为大写后返回。可以设置最大连接数量，超出的连接被丢弃。代码比较简陋，待完善。

代码放在码云上了，链接：[myepolldemo](https://gitee.com/zhanghh0624/myepolldemo)

**1.ep.h  头文件**

```C
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <stdlib.h>
#include <time.h>
#include <error.h>
#include <sys/socket.h>
#include <unistd.h>
#include <netinet/in.h> /* 定义数据结构sockaddr_in */
#include <arpa/inet.h>  /* 提供IP地址转换函数 */
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <ctype.h>

#define EP_TABLE_MAX_NUM  20   /* epoll最大处理 fd 数量 */

#define CLI_CONNECT_NUM   2  /* 客户端最大连接数 */

#define BUF_SIZE 1000          /* recv/send buf大小 */

#define SER_PORT 11111         /* 服务端监听端口 */
#define CLI_PORT 12111         /* 客户端绑定起始端口 */

#define THROW_ERR  strerror(errno)

#define SER_IP "192.168.1.6" /* 服务端ip */
#define CLI_IP "192.168.1.6" /* 客户端ip */

/* LOG，调试用函数，直接打印输出*/
#define LOG_STDOUT(format, ...)      \  
    do                               \
    {                                \
        printf(format, ##__VA_ARGS__); \
    } while (0)
```

**2.ep.c   服务端部分**

```C
#include "../lib/ep.h"

int main(int argc, char argv[])
{
    int ep_fd = 0;
    int cli_fd = 0;
    int nfds = 0;
    int cli_len = 0;
    int CLI_NUM = 0;
    int ret = 0;

    struct sockaddr_in ser_addr, cli_addr;

    ep_fd = epoll_create(1); /* 创建epoll的文件描述符 */

    int ser_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    set_noblocking(ser_fd); /* epoll监听的文件描述符需要设置为非阻塞模式 */

    bzero((void *)&ser_addr, sizeof(ser_addr));
    bzero((void *)&cli_addr, sizeof(cli_addr));
    ser_addr.sin_family = AF_INET; /* sin_port和sin_addr都必须是网络字节序（NBO），一般可视化的数字都是主机字节序（HBO） */
    ser_addr.sin_addr.s_addr = inet_addr(SER_IP);
    ser_addr.sin_port = htons(SER_PORT);

    if ((ret = bind(ser_fd, (struct sockaddr *)&ser_addr, sizeof(ser_addr))) == -1)
    {
        LOG_STDOUT("bind failed!%s\n", THROW_ERR);
    }

    if (listen(ser_fd, 100) == -1)
    {
        LOG_STDOUT("listen failed!%s\n", THROW_ERR);
    }

    struct epoll_event ep_ev;
    struct epoll_event ep_table[EP_TABLE_MAX_NUM]; /* epoll_wait返回的描述符存放在这个结构体数组中 */
    ep_ev.data.fd = ser_fd;                        /* 监听的文件描述符为ser_fd */
    ep_ev.events = EPOLLIN;                        /* 监听ser_fd的输入事件，即连接请求 */

    if (epoll_ctl(ep_fd, EPOLL_CTL_ADD, ser_fd, &ep_ev) < 0)
    {
        LOG_STDOUT("ep_ctl failed!\n", 0);
    }

    for (;;)
    {
        nfds = epoll_wait(ep_fd, ep_table, EP_TABLE_MAX_NUM, -1);

        if (nfds <= 0) /* 无描述符触发事件 */
            continue;
        else
        {
            for (int i = 0; i < nfds; i++)
            {
                if (ep_table[i].data.fd == ser_fd) /* 有连接请求接入 */
                {
                    CLI_NUM++; /* 接入计数+1 */
                    cli_fd = accept(ser_fd, (struct sockaddr *)&cli_addr, &cli_len);

                    if (CLI_NUM <= CLI_CONNECT_NUM) /* 小于组大连接数 */
                        LOG_STDOUT("recv connect from ip:%s,port:%d,client num:%d\n", inet_ntoa(cli_addr.sin_addr), ntohs(cli_addr.sin_port), CLI_NUM);
                    else
                    {
                        LOG_STDOUT("Too much connect...\n");
                        close(cli_fd);
                        continue;
                    }

                    /* 将接入的用户加入epoll的监听队列 */
                    set_noblocking(cli_fd);
                    ep_ev.data.fd = cli_fd;
                    ep_ev.events = EPOLLIN | EPOLLET;
                    if (epoll_ctl(ep_fd, EPOLL_CTL_ADD, cli_fd, &ep_ev) < 0)
                    {
                        perror("epoll ctl:");
                    }

                    continue;
                }

                /* 有客户端写入数据 */
                if (handle(cli_fd) < 0)
                    perror("handle:");
            }
        }
    }
close:
    close(ep_fd);

    LOG_STDOUT("Server is closing......\n");
    return 0;
}

int set_noblocking(int fd)
{
    if ((fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK)) < 0)
        perror("set fd noblock:");
}
/* 处理用户数据，转换为大写，并返回总长度。 */
int handle(int fd)
{
    char recv_buf[BUF_SIZE] = {0};
    char send_buf[BUF_SIZE] = {0};
    int ret;
    if (recv(fd, recv_buf, BUF_SIZE, 0) < 0)
    {
        return -1;
    }
    LOG_STDOUT("%s", recv_buf);
    for (int i = 0; recv_buf[i] != '\n'; i++)
    {
        recv_buf[i] = toupper(recv_buf[i]);
    }

    if ((ret = send(fd, recv_buf, BUF_SIZE, 0)) < 0)
    {
        return -1;
    }
    return ret;
}
```

**3.client.c     客户端部分**

```C
#include "../lib/ep.h"

int main()
{
    struct sockaddr_in ser_addr, cli_addr;
    int ser_len, cli_len;
    int ret = 0;
    int num = 0;
    char recv_buf[BUF_SIZE] = {0};
    char send_buf[BUF_SIZE] = {0};

    bzero((void *)&ser_addr, sizeof(ser_addr));
    bzero((void *)&cli_addr, sizeof(cli_addr));

    cli_addr.sin_family = AF_INET;
    cli_addr.sin_addr.s_addr = inet_addr(CLI_IP);
    cli_addr.sin_port = htons(CLI_PORT);

    ser_addr.sin_family = AF_INET;
    ser_addr.sin_addr.s_addr = inet_addr(SER_IP);
    ser_addr.sin_port = htons(SER_PORT);

    int cli_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    bind(cli_fd, (struct sockaddr *)&cli_addr, sizeof(cli_addr));
    if ((ret = connect(cli_fd, (struct sockaddr *)&ser_addr, sizeof(cli_addr))) == -1)
    {
        LOG_STDOUT("Connect error!%s\n", strerror(errno));
    }

    while (1)
    {
        memset(send_buf, 0, BUF_SIZE);
        memset(recv_buf, 0, BUF_SIZE);
        if (fgets(send_buf, BUF_SIZE, stdin) == NULL)
            break;
        send(cli_fd, send_buf, BUF_SIZE, 0);
        recv(cli_fd, recv_buf, BUF_SIZE, 0);
        LOG_STDOUT("%s", recv_buf);
        if (num == 100)
            break;
    }
    close(cli_fd);

    return 0;
}
```

有不完善的地方会定期更新。

以上。

参考文章：

[1] [深入理解Epoll](https://zhuanlan.zhihu.com/p/93609693)

[2] [IO多路复用的三种机制Select，Poll，Epoll](https://www.jianshu.com/p/397449cadc9a)

[3] [IO多路复用机制详解](https://www.cnblogs.com/yanguhung/p/10145755.html)

[4] [如果这篇文章说不清epoll的本质，那就过来掐死我吧！](https://zhuanlan.zhihu.com/p/63179839)

