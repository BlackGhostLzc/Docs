## 1.什么是socket
socket 的原意是“插座”，是一种操作系统提供的进程间通信机制。

在UNIX/Linux系统中，一切皆文件，网络连接也是一个文件，它也有文件描述符。
我们可以通过 `socket()` 函数来创建一个网络连接，或者说打开一个网络文件，`socket()` 的返回值就是文件描述符。有了文件描述符，我们就可以使用普通的文件操作函数来传输数据了，例如：
* 用 `read()` 读取从远程计算机传来的数据；
* 用 `write()` 向远程计算机写入数据。

还有一个比较重要的就是网络字节序。
网络字节序一般都是大端表示的，而主机字节序一般是小端表示的，所以需要包含一段这两者之间的转换函数。
```c
// 将一个短整形从主机字节序 -> 网络字节序
uint16_t htons(uint16_t hostshort);	
// 将一个整形从主机字节序 -> 网络字节序
uint32_t htonl(uint32_t hostlong);	

// 将一个短整形从网络字节序 -> 主机字节序
uint16_t ntohs(uint16_t netshort)
// 将一个整形从网络字节序 -> 主机字节序
uint32_t ntohl(uint32_t netlong);
```

而对于IP地址，我们一般见到的表示形式都是点分十进制，把这种点分十进制转换为网络字节序需要函数`inet_pton()`,`inet_ntop`将大端的整形数, 转换为小端的点分十进制的IP地址。


## 2.TCP通信流程

TCP是一个面向连接的，安全的，流式传输协议，这个协议是一个传输层协议。

![Socket通信流程_socket通信程序流程图_fightsyj的博客-CSDN博客](https://img-blog.csdnimg.cn/20191225154007754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maWdodHN5ai5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)


### 2.1服务器端通信流程
一般而言，服务器需要一个监听套接字和多个通信套接字，监听套接字负责与客户端建立连接，通信套接字则是与客户端进行数据传送。
1. 调用`socket()`函数创建用于监听的套接字，得到一个文件描述符。
2. 将得到的监听文件描述符和本地的IP端口进行绑定，`bind()`函数。
3. `listen()`函数设置监听。
4. `accept()`函数等待客户端的连接请求，会返回一个用于与客户端通信的文件描述符，如果没有请求就会堵塞。
5. `read()`或`recv()`接受数据，`write()`或`send()`发送数据。
6. `close()`断开连接，关闭套接字。

- 注意：服务端需要绑定一个特定的端口，不能随意分配。这是因为客户端在请求与服务器端进行连接的时候，需要指定IP地址以及端口，所以服务器的端口对于客户端来说是已知的。
例如，当我们输入网址`https://www.bilibili.com`，其实https的默认端口就是443，所以相当于我们省略了端口，输入`https://www.bilibili.com:443`也是一样的。再讲一句题外话，我们按下F12，点击网络，也会得到远程服务器的IP地址和端口，发现端口也是443，前提是你要关闭vpn代理。

### 2.2 客户端的通信流程
一般来说客户端通信的文件描述符只有一个，不需要坚挺的文件描述符。
1. `socket()`函数创建一个套接字，返回文件描述符。
2. `connect()`函数连接服务器，需要知道服务器绑定的IP和端口。
3. `read()`或`recv()`函数接受数据，`write()`或`send()`发送数据。
4. `close()`断开连接。



## 3.基本API
- `int socket(int domain, int type, int protocol)`：
参数：
  `domain`地址族，常见的有AF_INET（IPv4）和AF_INET6（IPv6）。
  `type` 套接字类型,常见的有SOCK_STREAM（用于TCP协议）和SOCK_DGRAM（用于UDP协议）。
  `protocal`协议，通常置为0，根据前两个参数自动选择合适的协议。
  

返回值是一个文件描述符。



- `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`
参数：
  `sockfd`需要绑定的套接字文件描述符。
  `addr`:存放了服务端用于通信的地址和端口。
  `addrlen`:结构体addr的大小。

`sockaddr_in`结构体：

```c
struct sockaddr_in
{
    sa_family_t sin_family;		/* 地址族协议: AF_INET */
    in_port_t sin_port;         /* 端口, 2字节-> 大端  */
    struct in_addr sin_addr;    /* IP地址, 4字节 -> 大端  */
    /* 填充 8字节 */
    unsigned char sin_zero[sizeof (struct sockaddr) - sizeof(sin_family) -
               sizeof (in_port_t) - sizeof (struct in_addr)];
};  
```




- `int listen(int sockfd, int backlog)`
参数：
  `sockfd`: 文件描述符, 可以通过调用`socket()`得到，在监听之前必须要绑定 `bind()`
  `backlog`: 同时能处理的最大连接要求，最大值为128
  
  
  
- `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`
参数：
  `sockfd`: 监听的文件描述符
  `addr`: **传出参数**, 里边存储了建立连接的客户端的地址信息,如客户端的端口和IP。
  `addrlen`: 传入传出参数，用于存储addr指向的内存大小

这个函数是一个阻塞函数，当没有新的客户端连接请求的时候，该函数阻塞；当检测到有新的客户端连接请求时，阻塞解除，新连接就建立了，得到的返回值也是一个文件描述符，基于这个文件描述符就可以和客户端通信了。



- `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`
参数：
  `sockfd`: 通信的文件描述符, 通过调用socket()函数就得到了
  `addr`: 存储了要连接的服务器端的地址信息: IP 和 端口，这个IP和端口也需要转换为大端然后再赋值
  `addrlen`: `addr`指针指向的内存的大小 `sizeof(struct sockaddr)`
  
  

## 4.示例代码
在我们的示例代码中，服务器端我们要使用多线程，每当一个客户端发过来一个请求时，服务器端都需要创建一个线程去处理这个请求。

```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<errno.h>
#include<pthread.h>
#include<netinet/in.h>
#include<sys/socket.h>
#include<ctype.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

#define SERV_PORT 9527
#define BUFSIZE 256

void *client_handler(void *socket_desc) {
    int client_socket = *(int *)socket_desc;
    char buf[1024];
    int ret = 0;

    while(1){
        ret = read(client_socket, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, ret);

        for(int i = 0; i < ret; i++){
            buf[i] = toupper(buf[i]);
        }

        write(client_socket, buf, ret);
    }

}

int main(int argc, char* argv[]){

    int lfd = 0, cfd = 0;
    int ret;
    char buf[BUFSIZE];

    struct sockaddr_in serv_addr, clit_addr;
    pthread_t thread_id;

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);


    lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1){
        sys_err("socket error");
    }

    bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    listen(lfd, 128);

    while(1){
        socklen_t clit_addr_len = sizeof(clit_addr);
        cfd = accept(lfd, (struct sockaddr *)&clit_addr, &clit_addr_len);

        if (cfd == -1) {
            perror("Accept failed");
            continue;
        }

        printf("Client connected\n");

        int *new_sock = (int *)malloc(1);
        *new_sock = cfd;

        if (pthread_create(&thread_id, NULL, client_handler, (void *)new_sock) < 0) {
            perror("Could not create thread");
            return 1;
        }
    }

    close(lfd);
    close(cfd);

    return 0;
}
```

`serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);`
这行代码的意思是可以绑定本地的任何一个IP地址，服务器端自动地读网卡的实际IP，并与这个实际IP进行绑定。
现在我们还没有客户端的代码，但我们仍然可以对服务器代码进行测试。
首先编译并运行服务器端代码，然后另外开几个终端，输入`nc 127.0.0.1 9527`命令。
>Linux中的nc命令是一个功能强大的网络工具，也被称为netcat，可以实现TCP/UDP端口的侦听，127.0.0.1代表本地主机，9527端口是服务器监听套接字绑定了的。
>
