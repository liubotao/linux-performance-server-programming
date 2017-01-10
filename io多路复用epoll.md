## I/O复用

I/O复用使得程序能同时监控多个文件描述符,这对提高程序的性能至关重要。通常，网络程序在下列情况下需要使用I/O复用技术

*  客户端程序同时要处理多个socket(比如非阻塞connect技术)
*  客户端程序要同时处理用户输入和网络请求(比如聊天室程序)
*  TCP服务器要同时监听socket和连接socket
*  服务器要同时处理TCP请求和UDP请求
*  服务器要同时监听多个端口或者处理多种服务

需要支出的是,I/O复用虽然能同时监听多个文件描述符,但它本身是堵塞的。并且当多个文件描述符同时就绪时,如果不采用额外的措施，程序只能依次处理其中的每一个文件描述符，这使得服务器看上去是串行工作的。如果要实现并发，则只能使用多进程或者多线程等编程手段。  

Linux下实现I/O复用的系统调用主要有select、poll和epoll

### select系统调用

select系统调用的用途是: 在一段指定时间内,监听用户感兴趣的文件描述符上的可读、可写和异常等事件。  

select API  

select系统调用的原型如下:  

    #include <sys/selet.h>
    int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds,
    struct timeval*timeout);
    
    1) nfds参数指定被监听的文件描述符的总数.它通常被设置为select监听的所有文件描述符的最大值加1,因此文件描述符是从0开始计数的
    2) readfds,writefds和excpetfds参数分别指向可读、可写和异常等事件对应的文件描述符集合。应用程序调用select函数时,通过这3个参数传入自己感兴趣的文件描述符。select调用返回时，内核将修改他们来通知应用哪些文件描述符已经就绪。这3个参数是fd_set结构指针类型.
    fd_set的接口体仅包含一组整形数组,该数组的每个元素的每一位标记一个文件描述符.fd_set能容纳的文件描述符的数量由FD_SETSIZE指定,这就限制了select能同时处理的文件描述符的总量.
    由于位操作过于繁琐,我们应该使用下面一组鸿访问fd_set结构体中的位;
    #include <sys/select.h>
    FD_ZERO*fd_set *fdset); /*清除fdset的所有位 */
    FD_SET(int fd, fd_set *fdset); /* 设置fdset的位fd */
    FD_CLR(int fd, fd_set *fdset); /* 清除fdset的位fd */
    int FD_ISET(int fd, fd_set *fdset) /* 测试fdset的位是否被设置 */
    
    3) timeout参数来设置select函数的超时时间。它是一个timeval结构类型的指针,采用指针参数是因为内核将修改它以告诉应用程序select等待了多久，不过我们不能完全信任select调用返回后的timeout值，比如调用失败时timeout值是不确定的。timeval结构体的定义如下:
    
    struct timeval {
        long tv_sec;   /* 秒数 */
        long tv_usec;   /* 微秒数 */
    }
    
    由以上定义可见,select给我们提供一个微秒级的定时方案.如果给timeout变量的tv_sec成员和tv_usec成员都传递0,则select将立即返回。如果timeout传递NULL，则select将一直堵塞，直到某个文件描述符就绪。
    select成功时返回就绪(可读、可写和异常)文件描述符的总数.如果在超时时间内没有任何文件描述符就绪,select将返回0。select失败时返回-1并设置errno。如果在select等待期间,程序接收到信号，则select立即返回-1，并设置errno为EINTR。
    
    


### EPOLL服务器DEMO

```
#include "stdio.h"
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <sys/types.h>

#include <pthread.h>

#define IPADDRESS "127.0.0.1"
#define PORT 9091
#define MAXSIZE 1024
#define LISTENQ 5
#define FDSIZE 1000
#define EPOLLEVENTS 100


int socket_bind(const char* ip, int port);

void do_epoll(int listenfd);

void handle_events(int epollfd, struct epoll_event *event, int num, int listenfd, char* buf);
void handle_accept(int epollfd, int listenfd);

void do_read(int epollfd, int fd, char* buf);
void do_write(int epollfd, int fd, char* buf);

void add_event(int epollfd, int fd, int state);
void modify_event(int epollfd,int fd, int state);
void delete_event(int epollfd, int fd, int state);

int main (){
    int listenfd;
    listenfd = socket_bind(IPADDRESS, PORT);
    listen(listenfd, LISTENQ);
    do_epoll(listenfd);
    return 0;
} 

int socket_bind(const char* ip, int port) {
    int listenfd;
    struct sockaddr_in serverAddr;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&serverAddr, sizeof(serverAddr));
    serverAddr.sin_family = AF_INET;
    inet_pton(AF_INET, ip,  &serverAddr.sin_addr);
    serverAddr.sin_port = htons(port);
    if((bind(listenfd, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) == -1)) {
        perror("bind error:");
        exit(1);
    } else {
        printf("server is Running : %s, port:%d\n", ip, port);
    }
    
    return listenfd;
}

void do_epoll(int listenfd) {
    int epollfd;
    struct epoll_event events[EPOLLEVENTS];
    int ret;
    char buf[MAXSIZE];
    memset(buf, 0, MAXSIZE);
    
    epollfd = epoll_create(FDSIZE);
    add_event(epollfd, listenfd, EPOLLIN);
    for (;;) {
        ret = epoll_wait(epollfd, events, EPOLLEVENTS, -1);
        handle_events(epollfd, events, ret, listenfd, buf);
    }
    close(epollfd);
}

void handle_events(int epollfd, struct epoll_event *events, int num, int listenfd, char *buf) {
    int fd;
    
    for (int i = 0; i < num; i++) {
        fd = events[i].data.fd;
        if ((fd ==listenfd) && (events[i].events & EPOLLIN)) {
            handle_accept(epollfd, listenfd);
        } else if (events[i].events & EPOLLIN) {
            do_read(epollfd, fd, buf);
        } else if (events[i].events & EPOLLOUT) {
            do_write(epollfd, fd, buf);
        }
    }
}


void handle_accept(int epollfd, int listenfd) {
    int clientfd;
    struct sockaddr_in clientAddr;
    socklen_t clientAddrLen;
    clientfd = accept(listenfd, (struct sockaddr*)&clientAddr, &clientAddrLen);
    if (clientfd == -1) {
        perror("accept error:");
    } else {
        printf("accept a new client :%s:%d\n", inet_ntoa(clientAddr.sin_addr), clientAddr.sin_port);
        add_event(epollfd, clientfd, EPOLLIN);
    }
}

void do_read(int epollfd, int fd, char* buf) {
    int nread;
    nread = read(fd, buf, MAXSIZE);
    if (nread == -1) {
        perror("read error:");
        close(fd);
        delete_event(epollfd, fd, EPOLLIN);
    } else if (nread == 0) {
        fprintf(stderr, "client close.\n");
        close(fd);
        delete_event(epollfd, fd, EPOLLIN);
    } else {
        printf("read message is : %s", buf);
        modify_event(epollfd, fd, EPOLLOUT);
    }
}

void do_write(int epollfd, int fd, char *buf) {
    int nwrite;
    nwrite = write(fd, buf, strlen(buf)); 
    if (nwrite == -1) {
        perror("write error:");
        close(fd);
        delete_event(epollfd, fd, EPOLLOUT);
    } else {
        modify_event(epollfd, fd, EPOLLIN);
    }
    memset(buf, 0, MAXSIZE);
}

void add_event(int epollfd, int fd, int state) {
    struct epoll_event event;
    event.events = state;
    event.data.fd = fd; 
    epoll_ctl(epollfd, EPOLL_CTL_ADD,fd, &event);
}

void delete_event(int epollfd, int fd, int state) {
    struct epoll_event event;
    event.data.fd = fd;
    event.events = state;
    epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, &event);
}

void modify_event(int epollfd, int fd, int state) {
    struct epoll_event event;
    event.events = state;
    event.data.fd = fd;
    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}
```
