### 主机字节系和网络字节系



    #include <stdio.h>
    
    void byteorder() {
        union {
            short value;
            char union_bytes[sizeof(short)];
        } test;
    
        test.value = 0x0102;
        if ((test.union_bytes[0] == 1) && (test.union_bytes[1]) == 2) {
            printf("big endian\n");
        } else if ((test.union_bytes[0] == 2) && (test.union_bytes[1] == 1)) {
            printf("little endian\n");
        } else {
            printf("unkonwn...\n");
        }
    }
    
    int main() {
        byteorder();
        return 0;
    }

### 通用socket地址

socket网络编程接口种表示socket地址的是结构体sockaddr，其定义如下:

     struct sockaddr {
        sa_family_t  sa_family;
        char sa_data[14];
    };
14字节的sa_data无法容纳多数协议族的地址值.因此,Linux定义了新的socket地址结构体:

    struct sockaddr {
        sa_family_t sa_family;
        unsigned long int __ss_align;
        char __ss_padding[128 - sizeof(__ss_align)];
    };
    

### 专用socket地址

通用socket结果体不够好用,比如设置或者获取IP地址和端口号就需要繁琐的位运算。所以Linux为各个协议族提供了专门的socket地址结构体。

UNIX本地协议族使用如下socket地址结构体:

    struct sockaddr_un {
        sa_family_t sa_family;  /* 地址族:AF_UNIX */
        char sun_path[108];     /* 文件路径名 */
    };

TCP/IP协议族有sockaddr_in和sockaddr_in6二个专用socket地址结构体,它们分别使用于Ipv4和Ipv6:

    struct sockaddr_in {
        sa_family_t sin_family;  /* 地址族:AF_INET */
        u_int16_t sin_port; /* 端口号,要用网络字节序表示 */
        struct in_addr sin_addr; /* Ipv4地址结构体 */
    };
    struct in_addr {
        u_int32_t s_addr; /* Ipv4地址,要用网络字节序表示 */
    };
    
    struct sockaddr_in6 {
        sa_family_t sin6_family;  /* 地址族:AF_INET6 */
        u_int16_t sin6_port; /* 端口号,要用网络字节序表示 */
        u_int32_t sina6_flowinfo; /* 流信息,应设置为0 */
        struct in6_addr sin6_addr; /* IPv6地址结构体*/
        u_int32_t sin6_scope_id; /* scope ID, 实验阶段 */
    };
    struct in6_addr {
        unsigned char sa_addr[16]; /* Ipv6地址,要用网络字节序表示 */
    };
   
### IP地址转换函数

    #include <arpa/inet.h>
    in_addr_t inet_addr(const char* strptr);    /* 将用点分十进制字符串表示的IPv4地址转化为网络字节系整数表示的IPv4地址 */
    int inet_aton(const char* cp, struct in_addr* inp); /* 功能如上，但是将转化结果存储与参数inp指向的结构体中 */
    char* inet_ntoa(struct in_addr in); /* 将网络字节序整数表示的IPv4地址转化为用点分十进制字符串表示的IPv4地址 */
    int inet_pton(int af, const char* src, void* dst); /*将用字符串表示的IP地址src转换成网络字节序整数表示的IP地址,并将结果存储与dst指向内存中 */
    const char* inet_ntop(int af, const void* src, char* dst, socklen_t cnt); /* 进行如上相反的转换 */
    
### 创建socket

UNIX/linux的哲学:所有东西都是文件.socket也不例外,他就是可读，可写，可控制，可关闭的文件描述符。

    #include <sys/types.h>
    #include <sys/socket.h>
    int socket(int domain, int type, int protocol);
    
### 命名socket
创建socket时，我们给它指定了地址族,但是并未指定使用地址族种具体socket。将一个socket与socket地址绑定称为socket命名。    
服务器程序中我们通常要命名socket,因为只有命名后客户端才知道如何连接它。    
客户端则通常不需要命名socket，而是采用匿名方式,即使用操作系统自动分配的socket地址。

    #include <sys/types.h>
    #include <sys/socket.h>
    int bind(int sockfd, const struct sockaddr* my_addr, socklen_t addrlen);  

### 监听socket

socket被命名之后,还不能马上接受客户端连接，需要使用系统调用创建一个监听队列用以存放待处理的客户连接:

    #include <sys/socket.h>
    int listen(int sockfd, int backlog); 
socketfd参数表示被监听的socket
backlog参数表示内核监听队列最大长度，内核2.2之后，它只表示处于完全连接状态的socket的上限，处于半连接状态的scoket上限则由/proc/sys/net/ipv4/tcp_max_syn_backlog内核参数定义。backlog典型的值是5.

    #include <stdio.h>
    #include <sys/_types/_sa_family_t.h>
    #include <stdbool.h>
    #include <sys/signal.h>
    #include <libgen.h>
    #include <stdlib.h>
    #include <sys/socket.h>
    #include <assert.h>
    #include <netinet/in.h>
    #include <strings.h>
    #include <arpa/inet.h>
    #include <unistd.h>
    
    static bool stop = false;
    
    static void handle_term(int sig) {
        stop = true;
    }
    
    int main(int argc, char *argv[]) {
        signal(SIGTERM, handle_term);
        if (argc <= 3) {
            printf("usage: %s ip_address port_number backlog\n", basename(argv[0]));
            return 1;
        }
    
        const char *ip = argv[1];
        int port = atoi(argv[2]);
        int backlog = atoi(argv[3]);
    
        int sock = socket(PF_INET, SOCK_STREAM, 0);
        assert(sock >= 0);
    
        struct sockaddr_in address;
        bzero(&address, sizeof(address));
        address.sin_family = AF_INET;
        inet_pton(AF_INET, ip, &address.sin_addr);
        address.sin_port = htons(port);
    
        int ret = bind(sock, (struct sockaddr *) &address, sizeof(address));
        assert(ret != -1);
    
        ret = listen(sock, backlog);
        assert(ret != -1);
    
        while (!stop) {
            sleep(1);
        }
    
        close(sock);
        return 0;
    }

执行     

    $ ./testlisten 127.0.0.1 12345 5 # 监听12345端口,给backlog传递值5
    $ telnet 127.0.0.1 12345    #多次执行
    $ netstat -nt | grep 12345  #多次执行
通过验证,处于ESTABLISHED状态的连接只有6个(backlog值+1)

### 接受连接

系统从listen监听队列中接受一个连接:
	
	#include <sys/types.h>
    #include <sys/socket.h>
    int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
    
我们把执行过listen调用,处于LISTEN状态的socket称为监听socket,处于ESTABLISHED状态的socket称为连接socket.

    #include <stdio.h>
    #include <sys/_types/_sa_family_t.h>
    #include <libgen.h>
    #include <stdlib.h>
    #include <sys/socket.h>
    #include <assert.h>
    #include <netinet/in.h>
    #include <strings.h>
    #include <arpa/inet.h>
    #include <unistd.h>
    #include <sys/errno.h>
    
    
    int main(int argc, char *argv[]) {
    
        if (argc <= 2) {
            printf("usage: %s ip_address port_number\n", basename(argv[0]));
            return 1;
        }
    
        const char *ip = argv[1];
        int port = atoi(argv[2]);
    
        struct sockaddr_in address;
        bzero(&address, sizeof(address));
        address.sin_family = AF_INET;
        inet_pton(AF_INET, ip, &address.sin_addr);
        address.sin_port = htons(port);
    
        int sock = socket(PF_INET, SOCK_STREAM, 0);
        assert(sock >= 0);
    
        int ret = bind(sock, (struct sockaddr *) &address, sizeof(address));
        assert(ret != -1);
    
        ret = listen(sock, 5);
        assert(ret != -1);
    
        sleep(20);
        struct sockaddr_in client;
        socklen_t client_addrlength = sizeof(client);
        int connfd = accept(sock, (struct sockaddr *) &client, &client_addrlength);
        if (connfd < 0) {
            printf("errorno is : %d\n", errno);
        } else {
            char remote[INET_ADDRSTRLEN];
            printf("connected with ip: %s and port : %d\n", inet_ntop(AF_INET, &client.sin_addr, remote, INET_ADDRSTRLEN),
                   ntohs(client.sin_port));
            close(connfd);
        }
        close(sock);
        return 0;
    }
    
 accept只是从监听队列中取出连接，而不论连接处于何种状态，更不关心任何网络状态的变化。
 
 
### 发起连接

如果服务器通过listen调用来被动接受连接,那么客户端通过如下系统调用来主动与服务器建立连接

    #include <sys/types.h>
    #include <sys/socket.h>
    
    int connect(int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen);
    
### 关闭连接

关闭一个连接实际上是关闭该连接对应的socket,这可以通过如下关闭普通文件描述符的系统调用来完成.

    #include <unistd.h>

    int close(int fd);
fd是待关闭的socket，不过close系统调用并发总数立即关闭一个连接,而是将fd的引用计数减1。只有当fd的引用计数为0的时候，才真正的关闭连接。多进程程序种，一次fork系统调用默认将父进程种打开的socket的引用计数加1，因此我们必须在父进程于子进程中都对该socket执行close调用才能将连接关闭。

如果我们无论如何都要立即关闭连接，可以使用shutdown系统调用。

    #include <unistd.h>

    int shutdown(int fd, int howto);
