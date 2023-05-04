# iOS防调试实践(防)


### ptrace函数

ptrace()是一个系统调用函数，提供了一种进程（the “tracer”）监察和控制另一个进程（the “tracer”）的方法。并且可以检查和改变“tracee”进程的内存和寄存器里的数据。它可以用来实现断点调试和系统调用跟踪。在多线程进程中，每个线程都可以被附着到一个tracer。

`int ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);
`
***ptrace()函数的参数如下：***

* **request**：表示要执行的操作，例如PTRACE_ATTACH表示附加到另一个进程，PTRACE_DETACH表示从另一个进程分离。
* **pid**：表示要操作的进程ID。如果pid为0，则表示当前进程。
* **addr**：表示要读取或写入的内存地址。
* **data**：表示要写入内存地址的数据或从内存地址读取的数据。

***request参数如下:***

```c
#define PT_TRACE_ME     0       /* child declares it's being traced */
#define PT_READ_I       1       /* read word in child's I space */
#define PT_READ_D       2       /* read word in child's D space */
#define PT_READ_U       3       /* read word in child's user structure */
#define PT_WRITE_I      4       /* write word in child's I space */
#define PT_WRITE_D      5       /* write word in child's D space */
#define PT_WRITE_U      6       /* write word in child's user structure */
#define PT_CONTINUE     7       /* continue the child */
#define PT_KILL         8       /* kill the child process */
#define PT_STEP         9       /* single step the child */
#define PT_ATTACH       ePtAttachDeprecated     /* trace some running process */
#define PT_DETACH       11      /* stop tracing a process */
#define PT_SIGEXC       12      /* signals as exceptions for current_proc */
#define PT_THUPDATE     13      /* signal for thread# */
#define PT_ATTACHEXC    14      /* attach to running process with signal exception */

#define PT_FORCEQUOTA   30      /* Enforce quota for root */
#define PT_DENY_ATTACH  31

#define PT_FIRSTMACH    32      /* for machine-specific requests */
```
* **PT_TRACE_ME** 表示子进程声明自己正在被跟踪；
* **PT_READ_I**、**PT_READ_D**、**PT_READ_U** 分别表示读取子进程的I空间、D空间和用户结构；
* **PT_WRITE_I**、**PT_WRITE_D**、**PT_WRITE_U** 分别表示写入子进程的I空间、D空间和用户结构；
* **PT_CONTINUE** 表示继续执行子进程；
* **PT_KILL** 表示杀死子进程；
* **PT_STEP** 表示单步执行子进程；
* **PT_ATTACH** 表示跟踪某个正在运行的进程；
* **PT_DETACH** 表示停止跟踪某个进程；
* **PT_SIGEXC** 表示当前进程的信号作为异常处理；
* **PT_THUPDATE** 表示线程号信号；
* **PT_ATTACHEXC** 表示使用信号异常附加到正在运行的进程。


-------

### 方法1: dlopen & dlsym

dlopen和dlsym函数是用于打开动态链接库中的函数，将动态链接库中的函数或类导入到本程序中。dlopen函数是用于打开一个动态链接库，包含头文件：#include \<dlfcn.h>。函数定义：void * dlopen (const char * pathname, int mode)。函数描述：在dlopen的（）函数以指定模式打开指定的动态连接库文件，并返回一个句柄给调用进程。通过这个句柄来使用库中的函数和类。使用dlclose（）来卸载打开的库。 

* RTLD_LAZY 暂缓决定，等有需要时再解出符号；

* RTLD_NOW 立即决定，返回前解除所有未决定的符号引用。

* RTLD_LOCAL：这是RTLD_GLOBAL的相反，如果没有指定任何标志，则为默认值。在此库中定义的符号不可用于解析随后加载的库中的引用。

* RTLD_GLOBAL是一个参数，它的含义是使得库中的解析的定义变量在随后的其它的链接库中变得可以使用。

* RTLD_NODELETE：（自glibc 2.2以来）在dlclose（）期间不卸载库。

* RTLD_NOLOAD：如果库已加载，则不要加载库。

* RTLD_DEEPBIND：使用此标志时，符号解析器将首先搜索由dlopen（）调用显式加载的对象，然后才搜索全局符号表。

* RTLD_FIRST：查找第一个出现的所需符号，使用默认库搜索顺序。

* RTLD_NEXT：查找当前库之后按搜索顺序出现的函数的下一个出现。这允许在另一个共享库中提供函数的包装器。

* RTLD_DEFAULT：查找默认库搜索顺序中所需符号的第一个出现。

* RTLD_SELF：在当前对象中解析符号。

```c
typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);

#if !defined(PT_DENY_ATTACH)
#define PT_DENY_ATTACH 31
#endif

void disable_debug(void) {
    void* handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
}

```
> 这段代码的第一行定义了一个名为ptrace_ptr_t的函数指针类型，该类型接受四个参数并返回一个整数。
> 
> 第二行定义了一个名为PT_DENY_ATTACH的常量，它等于31。
> 
> 第三行定义了一个名为disable_debug的函数，该函数使用dlopen和dlsym函数来获取ptrace函数的地址，并将PT_DENY_ATTACH作为请求参数传递给ptrace函数。
> 
> 这段代码的作用是禁止调试器附加到应用程序进程上。
 
 
-------

### 方法2: 导入头文件\<sys/ptrace.h>
 
 先创建一个macos平台项目, 随便找个位置导入#import \<sys/ptrace.h>,追踪到ptrace.h中, 将该头文件复制到iOS项目中, 你可以自定义一个Header文件加入到iOS项目中. 代码如下:
 
```c
//来自<sys/ptrace.h>
#ifndef _SYS_PTRACE_H_
#define _SYS_PTRACE_H_

#include <sys/appleapiopts.h>
#include <sys/cdefs.h>
#include <sys/types.h>

enum {
    ePtAttachDeprecated __deprecated_enum_msg("PT_ATTACH is deprecated. See PT_ATTACHEXC") = 10
};


#define PT_TRACE_ME     0       /* child declares it's being traced */
#define PT_READ_I       1       /* read word in child's I space */
#define PT_READ_D       2       /* read word in child's D space */
#define PT_READ_U       3       /* read word in child's user structure */
#define PT_WRITE_I      4       /* write word in child's I space */
#define PT_WRITE_D      5       /* write word in child's D space */
#define PT_WRITE_U      6       /* write word in child's user structure */
#define PT_CONTINUE     7       /* continue the child */
#define PT_KILL         8       /* kill the child process */
#define PT_STEP         9       /* single step the child */
#define PT_ATTACH       ePtAttachDeprecated     /* trace some running process */
#define PT_DETACH       11      /* stop tracing a process */
#define PT_SIGEXC       12      /* signals as exceptions for current_proc */
#define PT_THUPDATE     13      /* signal for thread# */
#define PT_ATTACHEXC    14      /* attach to running process with signal exception */

#define PT_FORCEQUOTA   30      /* Enforce quota for root */
#define PT_DENY_ATTACH  31

#define PT_FIRSTMACH    32      /* for machine-specific requests */

__BEGIN_DECLS


int     ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);


__END_DECLS

#endif  /* !_SYS_PTRACE_H_ */

```
 
 
```c
// MARK: 第二种方式,直接导入头文件
ptrace(PT_DENY_ATTACH, 0, 0, 0);
```

-------

### 方法3: syscall

在iOS平台下，syscall函数是一个系统调用，它是程序请求操作系统提供服务的编程方式。它可以包括与硬件相关的服务（例如，访问硬盘驱动器或访问设备的相机），创建和执行新进程以及与内核服务进行通信。

在iOS内核中，syscall的功能号是有限的，只有512个（有待扩展）功能方法，也即功能号。这些功能号包括了文件操作、网络、界面、算法等等。

syscall()函数的第一个参数是系统调用号，后面的参数是传递给系统调用的参数。

`int	 syscall(int, ...);`

26 = ptrace

```c
 // MARK: 第三种方式,syscall
 // syscall(26, 31, 0, 0, 0);
```

[第一个参数的编号说明参考Wiki](https://www.theiphonewiki.com/wiki/Kernel_Syscalls)


-------

### 方法4: sysctl

sysctl()函数用于在内核运行时动态地修改内核的运行参数，可用的内核参数在目录 /proc/sys 中。它包含一些TCP/IP堆栈和虚拟内存系统的高级选项，可以让有经验的管理员提高引人注目的系统性能。用sysctl()可以读取设置超过五百个系统变量。

使用sysctl()函数来检查当前进程是否正在被调试。它通过调用sysctl()函数来获取当前进程的kinfo_proc结构体信息，然后检查该结构体中的p_flag成员是否包含P_TRACED标志。如果包含，则说明当前进程正在被调试，函数返回YES，否则返回NO。

```c
#import <sys/sysctl.h>
BOOL test_sysctl(void) {
    int name[4];
    struct kinfo_proc info;
    size_t info_size = sizeof(info);
    
    info.kp_proc.p_flag = 0;
    
    name[0] = CTL_KERN;
    name[1] = KERN_PROC;
    name[2] = KERN_PROC_PID;
    name[3] = getpid();
    
    if (sysctl(name, 4, &info, &info_size, NULL, 0) == -1) {
        NSLog(@"sysctl error ...");
        return NO;
    }
    
    return ((info.kp_proc.p_flag & P_TRACED) != 0);
}
```

> 该方法由于和之前三个方式不同,只是做为标志位查询,获取的结果之后的crash等操作,需要由您来自定义完成.当然也可以不crash,只做记录也可以.

> 另外,该方法仅仅是查询一次,如果您需要轮询查询,则需要自己实现轮询逻辑.


## 反防调试

[iOS反防调试](https://github.com/qixin1106/DevelopmentNotes/blob/master/iOS反防调试实践/README.md)
