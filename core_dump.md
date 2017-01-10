### 1、前言

　　一直在从事linux下后台开发，经常与core文件打交道。还记得刚开始从事linux下开发时，程序突然崩溃了，也没有任何日志。我不知所措，同事叫我看看core，我却问什么是core，怎么看。同事鄙视的眼神，我依然在目。后来学会了从core文件中分析原因，通过gdb看出程序挂再哪里，分析前后的变量，找出问题的原因。当时就觉得很神奇，core文件是怎么产生的呢？难道系统会自动产生，可是我在自己的linux系统上面写个非法程序测试，并没有产生core问题？这又是怎么回事呢？今天在ngnix的源码时候，发现可以在程序中设置core dump，又是怎么回事呢？在公司发现生成的core文件都带有进程名称、进程ID、和时间，这又是怎么做到的呢？今天带着这些疑问来说说core文件是如何生成，如何配置。

### 2、基本概念

　　 当程序运行的过程中异常终止或崩溃，操作系统会将程序当时的内存状态记录下来，保存在一个文件中，这种行为就叫做Core Dump（中文有的翻译成“核心转储”)。我们可以认为 core dump 是“内存快照”，但实际上，除了内存信息之外，还有些关键的程序运行状态也会同时 dump 下来，例如寄存器信息（包括程序指针、栈指针等）、内存管理信息、其他处理器和操作系统状态和信息。core dump 对于编程人员诊断和调试程序是非常有帮助的，因为对于有些程序错误是很难重现的，例如指针异常，而 core dump 文件可以再现程序出错时的情景。

### 3、开启core dump

　　可以使用命令ulimit开启，也可以在程序中通过setrlimit系统调用开启。

*  打开 core dump功能
* 在终端中输入命令ulimit -c,如果输出结果为0，说明默认是关闭core dump的，即当程序异常终止时，也不会生出core dump文件
* 我们可以使用命令ulmit -c unlimited 来开启core dump功能，并且不限制core dump的大小，如果限制文件的大小，将unlimited改成你想生成core
  文件的大小，注意单位为KB
* 用上面的命令只会对当前的终端有效，如果想永久生效，可以修改/etc/security/limits.conf文件，关于此文件的设置

```
# /etc/security/limits.conf
* soft core unlimited
```

修改core文件保存的路径
 
 * 默认生成的core文件保存在可执行文件所在的目录下，文件名就为core 
 * 通过修改/proc/sys/kernel/core_uses_pid文件可以让生成的core文件名是否自动加上pid号
 * 还可以通过修改/proc/sys/kernel/core_pattern来控制生产core文件保存的位置以及文件名格式
   例如可以用 echo "/tmp/corefile-%e-%p-%t" > /proc/sys/kernel/core_pattern设置生成的core文件保存在"/tmp/corefile"目录下，文件
   名格式为 "core-命令名-pid-时间戳"
   
程序中开启core dump，通过如下API可以查看和设置RLIMIT_CORE

```
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);
```
参考程序如下所示：

```
#include <unistd.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <stdio.h>
#define CORE_SIZE   1024 * 1024 * 500
int main()
{
    struct rlimit rlmt;
    if (getrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   
    printf("Before set rlimit CORE dump current is:%d, max is:%d\n", (int)rlmt.rlim_cur, (int)rlmt.rlim_max);

    rlmt.rlim_cur = (rlim_t)CORE_SIZE;
    rlmt.rlim_max  = (rlim_t)CORE_SIZE;

    if (setrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   

    if (getrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   
    printf("After set rlimit CORE dump current is:%d, max is:%d\n", (int)rlmt.rlim_cur, (int)rlmt.rlim_max);

    /*测试非法内存，产生core文件*/
    int *ptr = NULL;
    *ptr = 10; 

    return 0;
}
```
执行./main, 生成的core文件如下所示



GDB调试core文件，查看程序挂在位置。当core dump 之后，使用命令 gdb program core 来查看 core 文件，其中 program 为可执行程序名，core 为生成的 core 文件名。


