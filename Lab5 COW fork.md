# Lab5 COW fork

实现写时复制的一个实验，主要是对页面错误的了解和应用；

我的[项目地址](https://github.com/monifeng/6.s081.git)；

## Implement copy-on write

### 实验要求

> Your task is to implement copy-on-write fork in the xv6 kernel. You are done if your modified kernel executes both the cowtest and usertests programs successfully.

实现写时复制，并完成测试；

实验要求非常简单易懂哈，但是这个hard标签怎么这么扎眼 : )

### 实验步骤

1. 无论如何，先去看看所谓写时复制是什么东西，把大概流程了解清楚，再做实验，参见课本第四章4.6节，我的github中收录了中文翻译材料；

   > COW fork ()只为子级创建一个分页表，用于用户内存的 PTE 指向父级的物理页面。COW fork ()将父级和子级中的所有用户 PTE 都标记为**不可写**。当任何一个进程尝试编写这些 COW 页面中的一个时，CPU 将强制出现页面错误。内核页面错误处理程序检测到这种情况，为错误进程分配一个物理内存页面，将原始页面复制到新页面，并在错误进程中修改相关的 PTE 以引用新页面，这一次 PTE 标记为可写。当页面错误处理程序返回时，用户进程将能够编写其页面的副本。
   >
   > COW fork ()使得释放实现用户内存的物理页面变得更加棘手。给定的物理页面可以由多个进程的页表引用，并且只有在最后一个引用消失时才应该释放。

   ![image-20230712211744974](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230712211744974.png)

2. 按照实验引导来进行实验，修改 `uvmcopy()` ，并不分配页面，而是让子进程映射到父进程的页面，并将 `PTE_W` 置为0；

3. 需要修改 `usertrap()` ，让它能够处理页面故障（page fault），如下图所示：页面故障所需要的三个信息：

   - 错误的地址（虚拟地址）：保存在 **stval** 寄存器中；
   - 错误的类型（读，写，指令错误）：**scause** 寄存器中，根据指导书，`r_scause == 13 || 15  ` 时是页面故障；
   - 异常程序计数器（保存错误的指令地址，方便恢复）：保存在sepc中，`trapframe->epc`；

   ![image-20230713181202422](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230713181202422.png)

   

4. 为所有页面设置计数器，否则会因为父进程的释放而释放物理页面，当子进程访问时，就会出错，按照实验提示来写计数器，并不是很难，但有些坑需要注意：例如需要加锁，解锁（多进程）；freepage时需要cnt+1，否则会出现负数，修改的函数如下，我全部列出来了；

   ```c
   struct ref_count
   {
     struct spinlock lock;
     int cnt[PHYSTOP / PGSIZE];
   } ref;
   
   int kaddrefcnt(void* pa) { // 放在uvmcopy，增加引用计数
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       return -1;
     acquire(&ref.lock);
     ++ref.cnt[(uint64)pa / PGSIZE];
     release(&ref.lock);
     return 0;
   }
   
   void
   freerange(void *pa_start, void *pa_end)
   {
     char *p;
     p = (char*)PGROUNDUP((uint64)pa_start);
     for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
     {
       acquire(&ref.lock);
       ref.cnt[(uint64)p / PGSIZE] = 1;
       release(&ref.lock);
       
       kfree(p);
     }
   }
   
   void
   kfree(void *pa)
   {
     struct run *r;
   
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       panic("kfree");
   
     // Fill with junk to catch dangling refs.
     memset(pa, 1, PGSIZE);
   
     r = (struct run*)pa;
     acquire(&ref.lock);
     ref.cnt[(uint64)r/PGSIZE]--;
     if (ref.cnt[(uint64)r/PGSIZE] == 0)
     {
       acquire(&kmem.lock);
       r->next = kmem.freelist;
       kmem.freelist = r;
       release(&kmem.lock); // ! kmem.lock
     }
     release(&ref.lock); // ! ref.lock
   }
   
   void *
   kalloc(void)
   {
     struct run *r;
   
     acquire(&kmem.lock);
     r = kmem.freelist;
     if(r)
     {
       kmem.freelist = r->next;
       acquire(&ref.lock);
       ref.cnt[(uint64)r / PGSIZE] = 1;
       release(&ref.lock);
     }
       
     release(&kmem.lock);
   
     if(r)
       memset((char*)r, 5, PGSIZE); // fill with junk
     
     return (void*)r;
   }
   
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     pte_t *pte;
     uint64 pa, i;
     uint flags;
   
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte should exist");
       if((*pte & PTE_V) == 0)
         panic("uvmcopy: page not present");
       pa = PTE2PA(*pte);
       flags = PTE_FLAGS(*pte);
   
       if(flags & PTE_W) {
         // 禁用写并设置COW Fork标记
         flags = (flags | PTE_F) & ~PTE_W;
         *pte = PA2PTE(pa) | flags;
       }
   
       if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
         goto err;
       }
       kaddrefcnt((void *)pa);
     }
     return 0;
   
    err:
     uvmunmap(new, 0, i / PGSIZE, 1);
     return -1;
   }
   ```

   千万不要漏掉uvmcopy，困扰我很久。。。

5. 修改copyout，和页面错误相同的处理方案；

   ```c
   int
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
     uint64 n, va0, pa0;
   
     while(len > 0){
       va0 = PGROUNDDOWN(dstva);
       pa0 = walkaddr(pagetable, va0); // 物理地址
   
       // 处理COW页面的情况
       if(cowpage(pagetable, va0) == 0) {
         // 重新申请物理地址
         pa0 = (uint64)cowalloc(pagetable, va0);
       }
   
       if(pa0 == 0)
         return -1;
       n = PGSIZE - (dstva - va0);
       if(n > len)
         n = len;
       memmove((void *)(pa0 + (dstva - va0)), src, n);
   
       len -= n;
       src += n;
       dstva = va0 + PGSIZE;
     }
     return 0;
   }
   ```

6. 修改usertrap，让usertrap能够响应页面错误

   查risc-v手册可知：r_scause() == 13 || 15就是页面错误；

   ```c
   else if(r_scause() == 13 || r_scause() == 15) {
     uint64 fault_va = r_stval();  // 获取出错的虚拟地址
     if(fault_va >= p->sz
       || cowpage(p->pagetable, fault_va) != 0
       || cowalloc(p->pagetable, PGROUNDDOWN(fault_va)) == 0)
       p->killed = 1;
     } 
      else if((which_dev = devintr()) != 0){
       // ok
   ```



### 实验结果

成功，本次实验因为涉及到很多文件和很多函数，还需要定义一些结构体，所以我放在了实验步骤中，不然可能会显得很混乱，就没有单独再分代码框架和实验细节的问题；

比较困难的实验，而且很多坑，也涉及到了锁的使用和并发的处理，但是收获非常大，脑中对写时复制有了个整体的印象：trap处理页面错误，在写时遇到COW页面需要分配内存，否则就会出现拷贝错误；

也了解到一些trap机制非常有用，也非常奇妙，实现了操作系统很多神奇的特性。用错误来找到页面并进行自己想要的处理，这是一种非常巧妙地思想，我认为值得我学习；

![image-20230714084809181](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230714084809181.png)