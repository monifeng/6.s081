# Lab6 Multithreading

多线程相关的实验，需要对进程切换的过程熟悉一下，然后实现用户线程版本；

我的[项目地址](https://github.com/monifeng/6.s081.git)

## Uthread: switching between threads

### 实验要求

> In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it. To get you started, your xv6 has two files user/uthread.c and user/uthread_switch.S, and a rule in the Makefile to build a uthread program. uthread.c contains most of a user-level threading package, and code for three simple test threads. The threading package is missing some of the code to create a thread and to switch between threads.
>
> Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan. When you're done, `make grade` should say that your solution passes the `uthread` test.

在 `user/uthread.c` 中实现用户进程切换的程序，需要保存并恢复寄存器。



### 实验步骤

1. 修改 `thread` 结构体；

   ```c
   struct context {
     uint64 ra;  // 返回地址
     uint64 sp;  // 栈指针
   
     // callee-saved
     uint64 s0;
     uint64 s1;
     uint64 s2;
     uint64 s3;
     uint64 s4;
     uint64 s5;
     uint64 s6;
     uint64 s7;
     uint64 s8;
     uint64 s9;
     uint64 s10;
     uint64 s11;
   };
   
   struct thread {
     char       stack[STACK_SIZE]; /* the thread's stack */
     int        state;             /* FREE, RUNNING, RUNNABLE */
     struct context context;
   };
   ```

2. 完善`thread_create()` 函数，需要保存传入的方法，并将栈指针保存为当前最高栈位置（从高向低增长）；

   ```c
   void 
   thread_create(void (*func)())
   {
     struct thread *t;
   
     for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
       if (t->state == FREE) break;
     }
     t->state = RUNNABLE;
     // YOUR CODE HERE
     t->context.ra = (uint64)func;
     t->context.sp = (uint64)t->stack + STACK_SIZE;
   }
   ```

3. 修改 `threat_schedule()` 函数，调用 `thread_switch` ；

   ```c
       /* YOUR CODE HERE
        * Invoke thread_switch to switch from t to next_thread:
        * thread_switch(??, ??);
        */
       thread_switch((uint64)&t->context, (uint64)&next_thread->context);
   ```

4. 编写 `thread_switch` ，可以参照 `swtch` 函数，保存所需的callee寄存器，并切换到caller寄存器；

   ```asm
   	.text
   
   	/*
            * save the old thread's registers,
            * restore the new thread's registers.
            */
   
   	.globl thread_switch
   thread_switch:
   	/* YOUR CODE HERE */
   	sd ra, 0(a0)
   	sd sp, 8(a0)
   	sd s0, 16(a0)
   	sd s1, 24(a0)
   	sd s2, 32(a0)
   	sd s3, 40(a0)
   	sd s4, 48(a0)
   	sd s5, 56(a0)
   	sd s6, 64(a0)
   	sd s7, 72(a0)
   	sd s8, 80(a0)
   	sd s9, 88(a0)
   	sd s10, 96(a0)
   	sd s11, 104(a0)
   
   	ld ra, 0(a1)
   	ld sp, 8(a1)
   	ld s0, 16(a1)
   	ld s1, 24(a1)
   	ld s2, 32(a1)
   	ld s3, 40(a1)
   	ld s4, 48(a1)
   	ld s5, 56(a1)
   	ld s6, 64(a1)
   	ld s7, 72(a1)
   	ld s8, 80(a1)
   	ld s9, 88(a1)
   	ld s10, 96(a1)
   	ld s11, 104(a1)
   
   	ret    /* return to ra */
   
   ```



### 实验细节

1. 一定不能在thread中把context定义为指针变量！否则因为没有分配内存和初始化（想念`new`了），交换两个空指针！
2. ra存放func，sp存放栈的最高地址：因为栈是从高到低开始增加的。

### 实验结果

成功。

![image-20230716213733177](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230716213733177.png)



## Using threads

### 实验要求

> In this assignment you will explore parallel programming with threads and locks using a hash table. You should do this assignment on a real Linux or MacOS computer (not xv6, not qemu) that has multiple cores. Most recent laptops have multicore processors.
>
> This assignment uses the UNIX `pthread` threading library. You can find information about it from the manual page, with `man pthreads`, and you can look on the web, for example [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_lock.html), [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_init.html), and [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_create.html).
>
> The file `notxv6/ph.c` contains a simple hash table that is correct if used from a single thread, but incorrect when used from multiple threads. In your main xv6 directory (perhaps `~/xv6-labs-2021`), type this:

在linux系统上做实验；

> Why are there missing keys with 2 threads, but not with 1 thread? Identify a sequence of events with 2 threads that can lead to a key being missing. Submit your sequence with a short explanation in answers-thread.txt

指出导致错误的代码，并解释一下，然后修改代码，让程序正确。



### 实验步骤

指出问题：

```c
static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}
```

在insert中，最后一行是改变头节点为当前节点，而insert则是获取当前头节点，两个线程可能出错，丢失第二个线程的头节点位置，也就无从谈起*获取头节点的地址；

我认为最小粒度的锁，就是在insert函数的位置加锁（不能进入函数再加，否则p已经丢失了）：

锁初始化的位置在main函数，main函数执行之前不可能创建出线程来。

```c
// in static void put(int key, int value)
if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&lock);
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock);
  }
```



### 实验结果

成功。

![image-20230716222638866](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230716222638866.png)



## Barrier

### 实验要求

> In this assignment you'll implement a [barrier](http://en.wikipedia.org/wiki/Barrier_(computer_science)): a point in an application at which all participating threads must wait until all other participating threads reach that point too. You'll use pthread condition variables, which are a sequence coordination technique similar to xv6's sleep and wakeup.

实现barrier，使用条件变量，来达到sleep和wakeup的效果。



### 实验步骤

1. 读源码，利用已给出的数据结构来完成barrier，需要利用bstate里的数据结构和外面的全局变量；

2. 每次所有线程都到达barrier的时候，需要增加barrier，测试条件就是看round增加是否准确；

3. 网上查找条件变量的用法，大概就是获取锁，然后条件变量wait时会释放锁，唤醒以后会重新获取锁；

   大概就是样的用法；

   ```c
   pthread_mutex_lock(&mutex);
   while(global_count<=0) {
   	pthread_cond_wait(&cond, &mutex);
   }
   global_count--;
   pthread_mutex_unlock(&mutex);
   ```

4. debug，并不是很复杂，但是有些地方很容易遗漏，可以看看实验的issue：

   > - You have to deal with a succession of barrier calls, each of which we'll call a round. `bstate.round` records the current round. You should increment `bstate.round` each time all threads have reached the barrier.
   > - You have to handle the case in which one thread races around the loop before the others have exited the barrier. In particular, you are re-using the `bstate.nthread` variable from one round to the next. Make sure that a thread that leaves the barrier and races around the loop doesn't increase `bstate.nthread` while a previous round is still using it.



### 代码框架

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
    
  // 加锁和条件变量的位置请自己填写，如果实在debug不出来可以参考我的gitee源码，在文章开头的github地址中跳转；
    
  bstate.nthread ++;
  if (bstate.nthread < nthread) 
  {
    // printf("sleep in thread %d\n", bstate.nthread);
      
  }
  else
  {
    bstate.round ++;
    bstate.nthread = 0;
   	// printf("round ++, round = %d.\n", bstate.round);

  }
  // 一定要在else外面解锁，否则wait过后无法解锁，会导致死锁的。
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```



### 实验细节

1. 虽然cond_wait经常和while连用，但这里明显不能，因为如果使用while，因为阻塞在while，一旦nthread归零，就永远也不可能到达后面的round++了；可以if语句里面的日志注释取消，打印出来看看是什么情况；
2. 解锁一定要在else外面，我之前放在else里面，始终卡在最后一个循环无法取消，原因就是最后一个线程结束后，按理来说需要释放锁，并不触发else语句，无法释放锁，另一个线程又在获取锁，导致死锁。



### 实验结果

成功。

![image-20230717095320167](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230717095320167.png)

说句题外话， 之前一直没通过测试，然后我就好奇的去看了一下测试程序，用python写的，一看才发现，原来就是简单的字符串匹配，如果打印日志就不可能通过，需要**注释掉日志**，但是我就想了一下，如果用邪道一点的办法，比如直接打印结果，会怎么样？

![image-20230717095718244](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230717095718244.png)

咳咳，果然是正确的，看来这个测试可以hack掉啊哈哈哈哈，还是有那么一点点改进的空间（可能是我的解法太邪道了^ _ ^，大家还是要走正道的哈，毕竟是学习为目的，不仅仅是为了通过一个测试哈）。



## Make garde

实验总体不是特别复杂，但是这种多线程环境的debug和编写程序，是一门非常复杂的工艺，稍不留神就容易出错，而且多线程程序一般只能通过打印日志来解决，gdb使用场景很少，但是有很多技巧和工具，比如sanitizer什么的。实验是做完了， 但是仅仅是一个学习的开始，算是小小的了解了一下多线程，但其实连门槛都没摸到，道阻且长啊。

![image-20230717101000047](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230717101000047.png)