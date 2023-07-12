# Lab4 : traps

到实验四貌似就比较难了，读课本的时候都读的有点吃力，看了Lecture也有点懵，先做着走吧，希望别卡太久。

项目地址：



## RISC-V assembly

### 实验要求

这个小实验主要是看源码，回答问题，让你熟悉一些RISC-V的指令的，这些我自己做了以后参考了网上的答案，但因为没找到标准答案（五花八门，什么都有），所以如果有错误或者想交流一下，**烦请私信或评论指教**。



### 实验过程

以下是回答，需要创建txt文档，命名为 `answers-traps.txt`，`make grade` 的时候会检查这个文档是否存在；

```c
1. Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?
	A：单printf的话，只用了a1和a2，其中a2保存13，不过在 `g` 中 a0保存了x；
	
2. Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)
	A：编译器直接优化了，直接保存12在a1寄存器中，所以我的理解就是在 `26:` 那行；
	
3. At what address is the function printf located?
	A：看不太懂，搜了一圈也没理解。
  	// 34:	5f8080e7          	jalr	1528(ra) # 628 <printf>
    按我的理解就是ra的地址+1528（转16进制）；
        
4. What value is in the register ra just after the jalr to printf in main?
	printf返回main的地址，看下一步的起始地址就行了，0x38；
        
5. Run the following code.

	unsigned int i = 0x00646c72;
	printf("H%x Wo%s", 57616, &i);
What is the output? Here's an ASCII table that maps bytes to characters.
The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?
    
	A： 首先转化为16进制，57616 -> 0xe110;
	所以前半段就是he110；
    后半段因为是小端输出，所以地址转字符串需要从后往前读，即 72 6c 64 -> r l d；
    如果是大端的话，就是64 6c 72 -> d l r；
    综合输出 ： hell0 world
        
6. In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?

	printf("x=%d y=%d", 3);
	A：第二个参数会读取a2的值，a2为空就是一个未定义的值，我认为会报错，网上的答案认为是输出一个未定义的值，不太理解。
```



这里挂一个网上找到的寄存器分配表格，感觉对这个实验的后续挺有用的：

```text
reg    | name  | saver  | description
-------+-------+--------+------------
x0     | zero  |        | hardwired zero
x1     | ra    | caller | return address
x2     | sp    | callee | stack pointer
x3     | gp    |        | global pointer
x4     | tp    |        | thread pointer
x5-7   | t0-2  | caller | temporary registers
x8     | s0/fp | callee | saved register / frame pointer
x9     | s1    | callee | saved register
x10-11 | a0-1  | caller | function arguments / return values
x12-17 | a2-7  | caller | function arguments
x18-27 | s2-11 | callee | saved registers
x28-31 | t3-6  | caller | temporary registers
pc     |       |        | program counter
```



## Backtrace

### 实验要求

> For debugging it is often useful to have a backtrace: a list of the function calls on the stack above the point at which the error occurred.
>
> Implement a `backtrace()` function in `kernel/printf.c`. Insert a call to this function in `sys_sleep`, and then run bttest, which calls `sys_sleep`. Your output should be as follows:
>
> ```
> backtrace:
> 0x0000000080002cda
> 0x0000000080002bb6
> 0x0000000080002898
>   
> ```
>
> After `bttest` exit qemu. In your terminal: the addresses may be slightly different but if you run `addr2line -e kernel/kernel` (or `riscv64-unknown-elf-addr2line -e kernel/kernel`) and cut-and-paste the above addresses as follows:
>
> ```
>     $ addr2line -e kernel/kernel
>     0x0000000080002de2
>     0x0000000080002f4a
>     0x0000000080002bfc
>     Ctrl-D
>   
> ```
>
> You should see something like this:
>
> ```
>     kernel/sysproc.c:74
>     kernel/syscall.c:224
>     kernel/trap.c:85
> ```

实验要求实现 `backtrace()` 函数，并且运行 `bttest` 来进行测试；



### 实验思路

1. 按照实验hint来先做着走，比较关键的就是那个课堂笔记；

   ![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230710194744609.png)

2. 要获取fp，就需要调用`r_fp()`， 这个函数在hint中给出来了；

3. `return address` 在fp-8，`sp` 在 fp - 16的位置；

4. 然后就是打印什么的问题，我们根据实验要求去看源码：三个地址经过转换定位到文件中：

   ```shell
       kernel/sysproc.c:74
       kernel/syscall.c:224
       kernel/trap.c:85
   ```

   依次查看后，发现是三个函数 ：`sys_sleep` ，`syscall` ， `usertrapret`；

   然后看了看系统调用的过程，发现有意思的东西；

   ![image-20230710204758923](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230710204758923.png)

   正好就是从 `sys_sleep` 返回到 `syscall` 开始的系统调用过程，然后就是打印这三个函数的地址。

5. 然后再看 调用 `backtrace` 的位置，就是在 `sys_sleep` 中，于是乎要做的事情就非常明了了：

   **找到当前函数返回的位置，打印然后到下一个函数，直到某个地址超出函数栈的范围**

6. 搞清楚 `fp`，`return address`，`saved frame pointer` 的含义，想要回溯到 `call` 该函数的 **函数地址**，就保存在 `saved frame pointer` 中，所以如果需要跳转到之前的函数，就对 `saved frame pointer`进行取值；



### 代码框架

函数比较简短，但是需要深刻理解函数栈的结构和各个指针的调用过程

```c
void backtrace()
{
  uint64 fp = r_fp();
  printf("backtrace:\n");
  while (fp >= PGROUNDDOWN(fp) && fp < PGROUNDUP(fp))
  {
    printf("%p\n", * ((uint64 *) (fp-8)));	// 取值，找到当前栈帧返回的地址;
    fp = * ((uint64 *) (fp - 16)); // 取值，获取上一栈帧;
  }
}
```



### 实验细节

1. 对fp，需要先转换成地址，然后对地址取址，才是另一个地址；
2. 不要被图中的toPrev. Frame（fp）里面那个fp骗了，我一开始就以为fp取出来是这个位置，一直不懂怎么跳到上一栈帧，其实获得fp是在 `return address` 的上方！！！；
3. 测试用代码 `addr2line -e kernel/kernel` 是退出了qemu才能执行的，我一开始直接在qemu中执行，一直提示错误，我还觉得挺怪的；



### 实验结果

成功。

![image-20230710221245565](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230710221245565.png)



## Alarm-test0

### 实验要求

> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes alarmtest and usertests.

添加一个定期（针对CPU事件）的警告，实现用户级别的中断/错误 处理程序；因为是hard实验，所以我决定先看完课程视频，多读两遍书再来做（backtrace感觉做的慢就是因为没有理解到栈的结构和指针）；

具体需要实现  `sigalarm` and  `sigreturn` 并将 `user/alarmtest.c` 添加到Makefile中，完成测试。



### 实验思路

看的出来是一个非常复杂的实验，但拖太久了，争取今天一天就搞定。

总之先跟着实验的hint一步一步来；

1. Makefile中添加alarmtest，然后根据hint提到的几个文件一步一步完善两个系统调用；
2. 开始写两个系统调用，先仅仅让 `sigreturn` 返回0，但是对于 `sigalarm` ，需要存储两个参数，`interval` 和 `handler`，后者需要保存到 `proc` 的新字段中，新字段需要自己添加到 `proc.h` 当中，两个系统调用都写在 `kernel/sysproc.c` 中，具体的操作函数写在 `proc.c` 中，并且在 `defs.h` 中定义；
3. 修改 `usertrap()` ，当ticks 达到预设时间时，就执行处理程序（**具体查看实验细节第3点**）；



### 代码框架

```c
// in sysproc.c
uint64 sys_sigalarm(void)
{
  int ticks;
  uint64 p_handler;
  // 用系统函数来取值并判错，注意ticks不能小于0！
  
  // 如何定义就如何传参，我自己定义的参数类型是函数指针，就传函数指针。反正都是地址，uint64也是可以的
  sigalarm(ticks, (void (*)()) p_handler);

  return 0;
}
```

下面这个函数可以不用定义，直接写在 `sys_sigalarm` 也是可以的，仅为个人习惯；如果和我一样，需要在 `defs.h` 中定义才能被其他文件引用！

```c
// in proc.c
void sigalarm(int ticks, void (*fn)())
{
  // 变量作用看名字就能看懂，定义在了proc.h中
  struct proc *p = myproc();
  p->pass_ticks = 0;
  p->ticks = ticks;
  p->p_handler = fn;
}
```

proc.h中新增变量：

```c
  int ticks;                   // CPU提醒间隔时间
  int pass_ticks;              // 离上一次调用过去的时间
  void (* p_handler)();        // 指向处理程序的指针
```



### 实验细节

1. 涉及到了函数指针的定义和使用，可以去网上查资料学习了解一下，这里只展示一下怎么调用和定义；

   ```c
   int Func(int x);   /*声明一个函数*/
   int (*p) (int x);  /*定义一个函数指针*/
   p = Func;          /*将Func函数的首地址赋给指针变量p*/
   ```

2. 做完实验2以后，panic时可以调用之前写的 `backtrace` 了

   ```shell
   # 在linux shell中执行
   addr2line -e kernel/kernel
   # backtrace打印出的地址
   ```

3. 非常需要理解的就是：如何调用写好的函数（handler），handler是一个**用户页表下的**虚拟地址，因此并不能直接在`uesrtrap`中调用，因为usertrap处于内核态，早已切换到了**内核页表**，并不能直接映射；所以需要先保存到**程序计数器**（请查看    `proc.h` ）中定义的 PC ，在返回到用户态后再执行handler。

   ![image-20230711210638145](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711210638145.png)

4. 



### 实验结果

成功。

![image-20230711211838867](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711211838867.png)



## Alarm-test1/2

### 实验要求

写出 `sigreturn`，需要了解哪些寄存器需要保存，并需要对实验整体的流程有一个整体的理解；

![image-20230711221329347](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711221329347.png)

可以见的：trapframe中的一些数据被覆盖了，导致执行完handler之后，无法恢复到中断前的现场了；而该实验要做的就是，在handler执行之前保存必要的寄存器的状态，在`sigreturn()` 中恢复这些寄存器；

并且在执行完handler之前，不能再次执行；

### 实验步骤

1. 添加trapframe的副本，添加指示是否在执行的标识符；
2. 需要保证不能重复执行，所以有一个简单的办法；就是直接在`sigreturn`才将pass_ticks置零，保证了调用结束才会重新开始计时；



### 代码框架

代码非常简单，但是整个思考的过程比较困难，有很多坑容易掉进去，比如会情不自禁的想有哪些寄存器需要保存然后恢复，但是事实上并不需要这样，直接保存一份trapframe 的副本就行（需要使用 `memmove` ）；

```c
// in trap.c usertrap()
if(which_dev == 2)
  {
    yield();
    p->pass_ticks ++;
    if (p->ticks == p->pass_ticks)
    {
      // 一定要分配位置，否则默认从0地址开始，就会执行出错；
      p->trapframe_copy = p->trapframe + sizeof(struct trapframe)+1;
      // 拷贝一份副本，方便后续还原，然后再将epc设置为p->handler的地址
      memmove(p->trapframe_copy, p->trapframe, sizeof(struct trapframe));
      p->trapframe->epc = (uint64)p->p_handler;  // 调用函数指针执行处理程序
    }
  }
```



```c
// in sysproc.c
uint64 sys_sigreturn(void)
{
  struct proc *p = myproc();
  memmove(p->trapframe, p->trapframe_copy, sizeof(struct trapframe));
  p->trapframe_copy = 0;
  p->pass_ticks = 0;
  return 0;
}
```

关于为什么trapframe无论在内核页表还是用户页表都能使用的问题：**tramframe和trampoline会存在于所有虚拟地址的最高处**，可以直接映射到物理地址；

![image-20230712090525608](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230712090525608.png)

![image-20230712091355680](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230712091355680.png)

一次是在虚拟地址空间的顶部， 一次是**直接映射**。

### 实验结果

成功！

![image-20230711224553544](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711224553544.png)



## make grade

整体来说，实验的难度是逐步递增的，但是代码量并不是很大，这次的实验非常有意思，也一直引导我在查阅资料和读源码，并且模仿源码实现自己的特性，虽然有时候实在想不到或者实现不出来，就去查阅网上的资料。但归根结底，还是需要对整个trap的过程要非常清楚，然后才能比较迅速和准确的做出实验；

实验结果是非常成功的，收获很大，但是花的时间有点多了（接近一周），下一个实验要更加仔细和认真才行。

![image-20230712091831426](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230712091831426.png)
