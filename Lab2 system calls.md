# Lab2: system calls

时隔一段时间，终于结束期末周，开始实验二。

[github项目地址](https://github.com/monifeng/6.s081)

### 01 System call tracing

#### 实验要求

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

在这个作业中，您将添加一个系统调用跟踪特性，它可以帮助您调试以后的实验室。您将创建一个新的跟踪系统调用来控制跟踪。它应该采用一个参数，一个整数“掩码”，其位指定哪个系统调用跟踪。例如，为了跟踪 fork 系统调用，程序调用 trace (1 < < SYS _ fork) ，其中 SYS _ fork 是 kernel/syscall.h 中的系统调用号。如果在掩码中设置了系统调用的编号，则必须修改 xv6内核，以便在每个系统调用即将返回时输出一行。这一行应该包含进程 id、系统调用的名称和返回值; 您不需要打印系统调用参数。跟踪系统调用应该能够跟踪调用它的进程及其随后分支的任何子进程，但不应影响其他进程。

这个实验的hint非常关键，一定要看完再做实验，不然根本不知道从何下手。



#### 思路

1. 严格按照hint的步骤阅读并实现，不看hint只会一头雾水（虽然看了也有点懵）；
2. 需要理解syscall的具体过程，多读几遍hint提到的源码，需要在各个地方添加trace的函数调用（函数指针syscalls数组等等）
3. 1.user/user.h中添加trace()函数
   2.user/usys.pl中添加trace的entry，编译时会自动在.S文件中添加trace的汇编代码
   3.kernel/syscall.h中定义trace的系统调用编码
   4.kernel/sysproc.c中添加sys_trace()函数
   5.kernel/proc.h中的proc添加mask掩码（掩码是1 << 系统调用编码的位运算结果，用十进制来表示）；
   6.从用户空间接收用户参数，参考kernel/syscall.c以及kernel/sysproc.c
   7.修改fork()，复制mask
   8.修改syscall()，声明一个字符串数组来记录名字方便打印，并且syscall来打印调用的数组；

#### 代码框架

本实验并不是写代码的实验， 更多的是理清楚一次系统调用的过程，这里没什么代码可以放，按部就班的修改已有的代码即可。为了理清思路，我建议自己去画一画系统调用的过程，这里我用xmind自己画了一个，仅供参考；

![image-20230628203008783](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230628203008783.png)



#### 细节

1. 解读实验要求给出的输出格式：

   ```c
   <pid>: syscall <syscall_name> -> <return_value>
   ```

2. 创建一个字符串数组，保存系统调用的名字，方便打印，注意system_call的系统编号从1开始，第一个要置为空；

   

#### 实验结果

成功。

![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230628203547899.png)



### 02 Sysinfo

#### 实验要求

> In this assignment you will add a system call, `sysinfo`, that collects information about the running system. The system call takes one argument: a pointer to a `struct sysinfo` (see `kernel/sysinfo.h`). The kernel should fill out the fields of this struct: the `freemem` field should be set to the number of bytes of free memory, and the `nproc` field should be set to the number of processes whose `state` is not `UNUSED`. We provide a test program `sysinfotest`; you pass this assignment if it prints "sysinfotest: OK".

在这个作业中，您将添加一个系统调用 sysinfo，它收集关于正在运行的系统的信息。系统调用有一个参数: 一个指向 struct sysinfo 的指针(参见 kernel/sysinfo.h)。内核应该填充这个结构的字段: freemem 字段应该设置为可用内存的字节数，nproc 字段应该设置为状态不是 UNUSED 的进程数。我们提供了一个测试程序 sysinfotest; 如果它输出“ sysinfotest: OK”，就视为通过。



#### 思路

- 简单来说就是一个收集系统信息的系统调用，经过上一个实验，应该是比较轻松能完成这些步骤的;
- 写两个函数，空闲空间大小的，和一个在运行的进程数量的函数；



#### 代码框架

```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/sysinfo.h"
#include "user/user.h"

int main(int argc, char **argv)
{
    if (argc > 1)
    {
        fprintf(2, "Usage: %s need not param", argv[0]);
        exit(-1);
    }

    struct sysinfo info;
    sysinfo(&info);
    printf("free space: %d\nused process: %d\n", info.freemem, info.nproc);
    exit(0);
}
```



```c
int sys_sysinfo()
{
	struct proc *p;
	struct sysinfo info;
	uint64 addr;

	if (argaddr(0, &addr) < 0)
	{
		return -1;
	}

	p = myproc();
	
	info.freemem = free_mem();
	info.nproc = get_nproc();	
	
	// copyout，在这里从内核复制到用户空间
    
    
	return 0;
}
```

free_mem()和get_nproc()都是需要自己写的，可以多研究一下kmem的定义，freelist就是空闲链表的最后一个位置，一直往前遍历，直到为NULL即可；

nproc更简单，遍历所有proc，状态不为UNUSED就可以count++；

总体来说不是特别难的实验，主要需要多读源码， 要对系统调用的整个过程有一个了解，可以参考网上的资料；

#### 细节

- 空闲空间大小的单位是比特，而不是页数（需要乘以PGSIZE）；
- 需要使用copyout将数据从内核传输到用户空间，按照hint里的提示，去参考一下源码；
- 需要自己写一个 `sysinfo.c` 在user文件夹中，然后调用sysinfo;



#### 实验结果

成功

![image-20230629112346481](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230629112346481.png)



### 03 总体测试

需要添加time.txt文件，然后 `make grade` ，结果如下图所示；

![image-20230629112543098](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230629112543098.png)
