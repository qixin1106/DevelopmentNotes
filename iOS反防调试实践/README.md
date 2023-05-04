# iOS反防调试实践(攻)

分别是对ptrace, syscall, sysctl, dlsym 4种方式进行反防调试

主要利用[fishhook](https://github.com/facebook/fishhook)来hook对应的函数,破解对方防御手段

[代码示例Demo](https://github.com/qixin1106/iOS_Anti_Debug_Demo)

[iOS防调试](https://github.com/qixin1106/DevelopmentNotes/blob/master/iOS反调试实践/README.md)介绍

## 反ptrace

```c
int (*ptrace_p)(int _request, pid_t _pid, caddr_t _addr, int _data);

int my_ptrace(int _request, pid_t _pid, caddr_t _addr, int _data) {
    if (_request != PT_DENY_ATTACH) {
        return ptrace_p(_request, _pid, _addr, _data);
    }
    printf("[AntiAntiDebug]检测到ptrace\n");
    return 0;
}

void anti_ptrace(void) {
    struct rebinding rebindings[1] = {{"ptrace", my_ptrace, (void*)&ptrace_p}};
    rebind_symbols(rebindings, 1);
}
```

> 当项目使用ptrace时,会进入my_ptrace函数, 通过判断request的值是否是`PT_DENY_ATTACH`来跳过验证.

## 反dlsym 

```c
#import <dlfcn.h>
void * (*dlsym_p)(void * __handle, const char * __symbol);

void * my_dlsym(void * __handle, const char * __symbol) {
    if (strcmp(__symbol, "ptrace") != 0) {
        return dlsym_p(__handle, __symbol);
    }
    printf("[AntiAntiDebug]检测到dlsym:ptrace\n");
    return my_ptrace;
}

void anti_dlsym(void) {
    struct rebinding rebindings[1] = {{"dlsym", my_dlsym, (void*)&dlsym_p}};
    rebind_symbols(rebindings, 1);
}
```

> 通过判断符号名是否是`ptrace`,来验证,如果是`ptrace`的话则不返回正常的ptrace,而是返回`my_ptrace`函数.(参考反ptrace方式)

## 反sysctl

```c
#import <sys/sysctl.h>
// 原始地址
int (*sysctl_p)(int *, u_int, void *, size_t *, void *, size_t);

int my_sysctl(int *name, u_int nameSize, void *info, size_t *infoSize, void *newInfo, size_t newInfoSize) {
    int ret = sysctl_p(name, nameSize, info, infoSize, newInfo, newInfoSize);
    if (nameSize == 4 &&
        name[0] == CTL_KERN &&
        name[1] == KERN_PROC &&
        name[2] == KERN_PROC_PID &&
        info &&
        (int)*infoSize == sizeof(struct kinfo_proc)) {
        struct kinfo_proc *info_p = (struct kinfo_proc *)info;
        if (info_p && (info_p->kp_proc.p_flag & P_TRACED) != 0) {
            printf("[AntiAntiDebug]sysctl 查询 trace 状态\n");
            info_p->kp_proc.p_flag ^= P_TRACED;
            if ((info_p->kp_proc.p_flag & P_TRACED) == 0) {
                printf("[AntiAntiDebug]trace状态移除了\n");
            }
        }
    }
    return ret;
}

void anti_sysctl(void) {
    struct rebinding rebindings[1] = {{"sysctl", my_sysctl, (void *)&sysctl_p}};
    rebind_symbols(rebindings, 1);
}
```

> 通过查询kinfo_proc结构的`kp_proc.p_flag`的`P_TRACED`位来判断是否被调试

> 当hook时,我们把状态位改成未调试状态,这样无论对方如何查询,都会返回`未调试`状态.

## 反syscall

```c
int (*syscall_p)(int, ...);

int my_syscall(int code, ...) {
    va_list args;
    va_start(args, code);
    if (code == 26) {
        int request = va_arg(args, int);
        if (request == PT_DENY_ATTACH) {
            printf("[AntiAntiDebug]syscall 在调用 ptrace\n");
            return 0;
        }
    }
    va_end(args);
    return (int)syscall_p(code, args);
}

void anti_syscall(void) {
    struct rebinding rebindings[1] = {{"syscall", my_syscall, (void *)&syscall_p}};
    rebind_symbols(rebindings, 1);
}
```

> 拦截syscall函数,判断第一个参数是否是26,26意味着,调用的是`ptrace`函数,第二个参数如果是`PT_DENY_ATTACH`31的话,代表对方使用syscall来调用`ptrace`,过滤掉.

