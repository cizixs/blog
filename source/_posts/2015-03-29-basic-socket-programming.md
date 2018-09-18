---
layout: post
title: "socket 编程基础知识"
excerpt: "how to write socket programs, first step."
categories: 程序技术
tags: [socket,linux,tutorial,example]
comments: true
share: true
---


这篇文章介绍了网络的基本概念，socket 编程的基础知识和 C 语言提供的 socket 库使用。

1. 本文只考虑 ipv4，不考虑 ipv6。
2. 只考虑网络上 AF_INET socket 类型，不考虑 UNIX 域协议以及其他类型。

TL;DR

## 概念

### 什么是 socket
计算机里面最令人烦的就是这些名词，它们都很抽象，而且解释起来就和没有解释差不多。socket 就是这样的一个概念，不过我还是要试着说明一下。

简单来说，**socket 是对底层网络通信的一层抽象，让程序员可以像文件那样操作网络上发送和接收的数据。**

再说的详细一点，假设现在你要编程网络程序，进行服务器端和客户端的通信（数据交换）。不适用 socket 的话，你会做下面的一堆事情：

+ 管理缓存区来接收和发送数据
+ 告诉操作系统自己要监听某个端口的数据，还有自己处理这些系统调用来读取数据
+ 当没有连接的时候或者另外一端要还没有发送数据时候，要处理 IO 阻塞，把自己挂起
+ 封装和解析 tcp/ip 协议的数据，维护数据发送的顺序
+ 等等

做了一大堆东西，发现最重要的还没有做：发送/接收数据。如果有一个程序能够自动帮我们把上面的东西都做掉，这样我们就可以只关心数据的读写，编程就简单的多了。那么这样一个程序就是 socket，它现在已经是操作系统的一部分，在 linux 中是标准的系统调用，只要调用它提供的一组接口（下面会详解常用函数的使用），就能轻松地建立连接，读写数据，关闭连接，让网络操作就像文件操作一样简单。

这下体会到 `Everything is file!`  unix 哲学的优点了吧。

![](https://image.slidesharecdn.com/scalelinuxperformance-130224171331-phpapp01/95/linux-performance-analysis-and-tools-16-638.jpg?cb=1362166290)

### 通信地址
现实生活中，两个人要邮寄信件，必须知道对方的地址。网通信也是如此，只不过这里通信的是程序。程序的地址由三元组（ip 地址，端口，协议）界定。

如果你了解网络协议模型的话，你就会知道，ip 地址是网络层用来路由和通信的标识符，端口（port） 是传输层管理的。而 socket 是在这两层之上，所以需要这两个地址来标识。这里的协议指的是 ipv4，ipv6 或者其他协议。

### socket 类型
创建 socket 的时候需要指定 socket 的类型，一般有三种：

1. SOCK_STREAM：面向连接的稳定通信，底层是 TCP 协议，我们会一直使用这个。
2. SOCK_DGRAM：无连接的通信，底层是 UDP 协议，需要上层的协议来保证可靠性。
3. SOCK_RAW：更加灵活的数据控制，能让你指定 IP 头部



### 术语表

名称     | 含义
-----   | -------
socket  |  创建一个通信的管道
bind    |  把一个地址三元组绑定到 socket 上
listen  |  准备接受某个 socket 的数据
accept  |  等待连接到达
connect  | 主动建立连接
send    |  发送数据
receive  |  接受数据
close    | 关闭连接

![](https://lh3.ggpht.com/-48jccDGoPY8/UYALpBdKtmI/AAAAAAAAArc/JuqOtnMNOP0/network4_thumb%25255B4%25255D.png?imgmax=800)

### 字节序
不同的计算机对数据的存储格式不一样，比如 32 位的整数 0x12345678，可以在内存里从高到低存储为 12-34-56-78 或者从低到高存储为 78-56-34-12。关于字节序的内容不会详细介绍，不了解的可以自己查阅相关的资料。

但是这对于网络中的数据来说就带来了一个严重的问题，当机器从网络中收到 12-34-56-78 的数据时，它怎么知道这个数据到底是什么意思？

解决的方案也比较简单，在传输数据之前和接受数据之后，必须调用 htonl/htons 或 ntohl/ntohs 先把数据转换成网络字节序或者把网络字节序转换为机器的字节序。


+ TCP 和 UDP 的端口是互不干扰的，也就是说系统可以同时开启 TCP 80 端口和 UDP 80 端口。
+ socket 不属于任何一层网络协议，它是对 TCP 层的封装，方便网络编程。

## C 语言 socket 使用

### 创建一个 socket
    int socketid = socket(family, type, protocol);
   
+ socketid: socket 描述符，可以看做是一个文件描述符，通过它来读/写数据
+ family：整数，通信域。
    + AF_INET：因特网协议协议，网络地址，最常用。
    + AF_UNIX，本地通信，文件地址
+ type：通信类型
    + SOCK_STREAM：可靠的，面向连接的服务，TCP 协议
    + SOCK_DGRAM：不可靠，无连接的服务，UDP 协议
    + SOCK_RAW：需要自己管理 IP 头部的数据
+ protocol：协议
    + IPPROTO_TCP，IPPROTO_UDP
    + 一般设为 0， 表示使用默认协议

如果出错的话，socketid 返回值是 -1。

### 关闭（close） socket
    int status = close(socketid);

关闭连接，释放端口。如果有错，返回值 status 为 -1，否则为 0。

### 绑定（bind）地址三元组到 socket
    int bind(int fd, const struct sockaddr *, socklen_t);

把 socket 绑定到某个地址三元组，用于 server 端监听端口。第一个参数是 socket 的描述符，第二个参数 `struct sockaddr` 是地址结构体，第三个参数是地址结构体的长度。绑定失败的话返回值为负数，否则为 -1，并且设置 `errno`。

其中最重要的就是地址结构体，它在 `netinet/in.h` 中被定义：

    struct sockaddr_in
    {
      short   sin_family; /* must be AF_INET */
      u_short sin_port;   /* 端口号，必须要通过 htons 转换为网络格式 */
      struct  in_addr sin_addr;  /* ip 地址 */
      char    sin_zero[8]; /* Not used, must be zero */
    };

其中， `in_addr` 也是在同一个文件夹被定义，格式为：

    struct in_addr
    {
        uint32_t s_addr; //32位整数
    };

服务器端的 s_addr 是本机地址，可以用 `INADDR_ANY` 变量表示接受来自任何地址的连接，记得在使用之前把地址变量初始化为全 0。
`sockaddr` 是通用的 socket 地址结构，`sockaddr_in` 是网络 socket 的结构，参数有一个类型转换的过程。

### 监听（listen）socket
    listen(sockfd, 5);

`listen` 系统调用让进城坚监听在制定的 socket 上面，第一个参数是 socket 描述符，第二个参数是最大连接数，表示发来请求但是没有被 accept 的连接数量。`listen` 函数在成功时返回 0，失败时返回 -1，并且设置错误代码。

### 请求连接（connect）
客户端要连接自己的 socket 和服务器端监听 socket 的方式就是 `connect`：

    int connect(int socket, const struct sockaddr* address, size_t address_len);
    
socket 是客户端本地创建的套接字，`address` 是服务器的三元组地址。成功调用时，服务器端将收到请求，`accept` 连接之后，就在两者之间建立了 socket 通信的管道，之后的读写就是直接对 socket 进行操作。

### 接受（accept）连接

    new_socket = accept(socket_desc, (struct sockaddr *)&client, (socklen_t*)&c);
    
当客户端有连接请求过来时，`accept` 函数接受该连接，把客户端的 socket 地址信息保存到 `client` 变量里，新建一个 socket，返回其描述符，然后数据的读写就能通过新 socket 进行。
新 socket 的地址和服务器监听 socket 是一样的，如果不关心客户端地址信息的话，可以把第二个和第三个参数都设置为空指针 `NULL`。

有了 `client` 变量，就能得到客户端的 ip 和 port ：

    char *client_ip = inet_ntoa(client.sin_addr);
    int client_port = ntohs(client.sin_port);

如果没有客户端连接，`accept` 函数将会阻塞，直到有连接过来。

### 读/写（Write）数据
上面那么多的函数调用，只是建立了服务器端和客户端的连接，算是通信前的准备工作，两者都有了自己的 socket 描述符。
有了 socket 描述符，就可以像文件那样进行读写数据：

    write(socket_des, message, strlen(message));
    read(socket_des, buffer, sizeof(buffer));

需要注意的是，`read` 函数调用是阻塞的，也就是说如果没有数据发送过来的话，该函数会一直等待，直到可以读到数据。

`read`和 `write` 返回的是实际读写的数据，这个数据最大是 buffer 的大小。如果传输的数据大于 buffer 的话，需要在程序里显式地去读取，否则会出错。

你可能会想，我一直读到返回的数据小于 sizeof(buff) 不就行了。嗯，这是一个解决方案，不过要判断返回值不是 0，因为返回值是 0 表示连接已经中断（需要调用 close 来关闭 socket），而不是没有数据发送过来。

### 其他常用函数

1. 获取 ip 地址

    很多时候，我们只知道服务器的域名，并不知道 ip 地址。`gethostbyname` 函数就能完成这个功能，`netdb.h` 文件里有它的定义，它的原型是：
    
            #include <netdb.h>
            
            struct hostent * gethostbyname(const char *name);
    
    参数 `name` 是诸如 `www.google.com` 的字符串，返回值是 `struct hostent` 结构体，用来存储得到的地址信息。
    
            struct hostent
            {
              char *h_name;         /* Official name of host.  */
              char **h_aliases;     /* Alias list.  */
              int h_addrtype;       /* Host address type.  */
              int h_length;         /* Length of address.  */
              char **h_addr_list;       /* List of addresses from name server.  */
            };
    
    如果函数调用失败，返回空指针 `NULL`。
    
    
2. 把 long 类型的 ip 转换为字符串类型

            #include <arpa/inet.h>
            
            char *inet_ntoa(struct in_addr);
            
            int inet_aton(const char *cp, struct in_addr *inp);
        
    上面的函数返回可用的 in_addr 结构体，需要你手动赋值。下面的函数把转换后的结构拷贝到 inp 指向的结构体里面，然后 inp 就可以直接使用了。
        
3. 把字符串类型的 ip 转换为 long 类型

            #include <arpa/inet.h>
            
            in_addr_t inet_addr(const char *ip);

4. 把字符串转换成整数

        int atoi(const char *nptr);
        
    这个可以把从键盘输入的端口号转换成可用的整数。
    
5.  getpeername：获取连在某个 socket 另一端的客户地址(ip 和 port)

        int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
        
    返回的信息保存在 addr 结构体里。
        

## 简单的 echo server
有了上面的知识，我们就来写一个简单的 echo server。这个 server 的功能非常简单，它默认监听在本机的 54321 端口，接受 client 端连接，然后把客户端发送的数据加上时间戳发送回去。

    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <arpa/inet.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <errno.h>
    #include <string.h>
    #include <sys/types.h>
    #include <time.h>
    
    #define BUFFER_SIZE 1024
    #define PORT 54321
    #define BACKLOG 5
    
    int setup_sock(int port)
    {
        /*
            * This function sets up a socket listening on local port.
            *
            * port: port number to listen on.
            * :return: socket file descriptor.
            */
        int listen_fd;
        struct sockaddr_in serv_addr;
        
        listen_fd = socket(AF_INET, SOCK_STREAM, 0);
        memset(&serv_addr, 0, sizeof(serv_addr));
        
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        serv_addr.sin_port = htons(port);
        
        bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        listen(listen_fd, BACKLOG);
        
        return listen_fd;
    }
    
    
    void echo_request(int conn_fd)
    {
        int n;
        time_t ticks;
        char sendBuff[BUFFER_SIZE];
        char recvBuff[BUFFER_SIZE];
        
        memset(sendBuff, 0, sizeof(sendBuff));
        memset(recvBuff, 0, sizeof(recvBuff));
        
        while( (n = recv(conn_fd, recvBuff, sizeof(recvBuff), 0)) > 0)
        {
            ticks = time(NULL);
            snprintf(sendBuff, sizeof(sendBuff), "%.24s: ", ctime(&ticks));
        
            recvBuff[n] = '\0';
            printf("received:  %s", recvBuff);
            strcat(sendBuff, recvBuff);
            send(conn_fd, sendBuff, strlen(sendBuff), 0);
        }
        printf("received 0 bytes, close.\n");
        close(conn_fd);
    }
    
    
    int main(int argc, char *argv[])
    {
        int listen_fd = 0, conn_fd = 0;
        socklen_t cli_len;
        struct sockaddr_in cli_addr;
        
        printf("start server...\n");
        memset(&cli_addr, '0', sizeof(cli_addr));
        
        listen_fd = setup_sock(PORT);
        printf("listening on 0.0.0.0 %d...\n", PORT);
        
        cli_len = sizeof(cli_addr);
        while(1)
        {
            conn_fd = accept(listen_fd, (struct sockaddr*)&cli_addr, &cli_len);
            printf("client ip: %s, port: %d\n",
                    inet_ntoa(cli_addr.sin_addr),
                    ntohs(cli_addr.sin_port));
        
            echo_request(conn_fd);
        }
    }

用 telnet 测试的结果如下：

![](http://i3.tietuku.com/2bffcd42526f9631.png)

这个程序没有很好的错误检查，而且每次只能和一个客户端进行通信，后面接进来的客户端必须要等到前面的客户端主动结束之后才能开始。以后会讲到怎么处理多连接的问题，这个例子只是 socket 基础知识的 demo。

## 参考资料

+ [Introduction to Sockets Programming in C using TCP/IP](http://www.csd.uoc.gr/~hy556/material/tutorials/cs556-3rd-tutorial.pdf)
+ [Beej's Guide to Network ProgrammingUsing Internet Sockets](http://beej.us/guide/bgnet/)
+ [socket programming in C on Linux](http://www.binarytides.com/socket-programming-c-linux-tutorial/)
