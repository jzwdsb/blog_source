---
date: 2018-05-06 21:30:00
category: 计算机网络
title: 并发服务器
description: 框架，示例
---

## 框架

```C
pid_t pid;
int listenfd, connfd;
listenfd = Socket(...);
Bind(listenfd, ...);
Listen(listenfd, LISTENQ);
while (true)
{
    connfd = Accept(listenfd, ...);
    if ((pid = fork()) == 0)
    {
        Close(listenfd);
        doit(connfd);
        Close(connfd);
        exit(0);
    }
    Close(connfd);
}
```

　　并发的起始点在第 9 行的 fork(), 由创建的子进程处理请求，父进程关闭连接套接字后继续等待新的请求．
　　fork 函数通过将父进程拷贝的方法创建子进程，调用完毕后，父进程和子进程都从 fork 返回的位置开始执行，由于 fork 在 linux 下存在的历史很长，所以 fork 经过极大的优化，它的调用开销比实际拷贝父进程要小很多，优化方法例如写时复制．linux 下没有线程的概念，线程是通过 fork 后通过在多进程间共享页面实现的．
　　fork 的特点是它调用一次，在表现上返回两次，在父进程返回一次，返回值为子进程的 pid; 在子进程中返回一次，返回 0, 通过这个可以区分父进程和子进程从而执行不同的代码．
　　由于进程正常 exit 后会关闭所有的文件描述符，所以子进程中显式调用 close 并非必须，但是注意到父进程中也调用了 Close.
　　对于每个文件或套接字，其都存在一个引用计数，引用计数维护在文件表项里，它是当前打开着的引用该文件的或套接字的描述符的个数．当 fork 返回后，父进程和子进程共享相同的文件描述符，所以其引用计数为 2, 当一个套接字描述符被关闭时会导致发送一个 FIN, 套接字真正被关闭是在其引用计数变为 0, 时发生的，所以在父进程中同样也要调用 Close, 如果想立即在某个套接字上发送 FIN, 可以使用 shutdown.

## 示例

　　以下分别列出一个 echo 服务器的代码和客户端的代码

### 服务器端

```C
int main(int argc, char* argv[])
{
    int listenfd, connfd, n;
    pid_t pid;
    sockaddr_in serveraddr{0}, clientaddr{0};
    socklen_t  size;
    char buff[MAXLINE];
    char address[MAXLINE];
    time_t ticks;
    listenfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(8080);
    
    inet_ntop(AF_INET, &serveraddr.sin_addr, address, sizeof(serveraddr));
    printf("server address: %s\n", address);
    fflush(stdout);
    Bind(listenfd, (sockaddr*)&serveraddr, sizeof(serveraddr));
    Listen(listenfd, LISTENQ);
    while (true)
    {
        size = sizeof(clientaddr);
        connfd = Accept(listenfd, (sockaddr*)&clientaddr, &size);
        if ((pid = fork()) == 0)
        {
            inet_ntop(AF_INET, &clientaddr.sin_addr, address, size);
            printf("connect from %s, child process %d processing\n", address, getpid());
            while ((n = read(connfd, buff, MAXLINE)) > 0)
            {
                printf("pid %d, client send %d byes: %s\n",getpid(), n, buff);
                Write(connfd, buff, strlen(buff) + 1);
            }
            if (n < 0)
                err_sys("read error");
            printf("connect to client %s closed\n\n", address);
            Close(connfd);
            exit(0);
        }
        Close(connfd);
    }
}
```

### 客户端

```C
int main(int argc, char* argv[])
{
    int sockfd, n;
    char buff[MAXLINE];
    char msg[MAXLINE];
    sockaddr_in serveraddr;
    if (argc not_eq 1)
    {
        printf("usage: a.out <IPaddress>\n");
        return -1;
    }
    sockfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_port = htons(8080);
    Inet_pton(AF_INET, "127.0.0.1", &serveraddr.sin_addr);
    Connect(sockfd, (sockaddr*) &serveraddr, sizeof(serveraddr));
    printf("connect Success\n");
    while (scanf("%s", msg) == 1)
    {
        printf("client input: %s\n", msg);
        Write(sockfd, msg, strlen(msg) + 1);
        n = read(sockfd, buff, strlen(msg) + 1);
        if (n < 0)
            break;
        printf("server send: %s\n", buff);
    }
    if (n < 0)
        err_sys("read error");
    return 0;
}
```