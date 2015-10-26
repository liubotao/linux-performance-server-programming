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
    
    




