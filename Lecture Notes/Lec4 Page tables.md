## Lec4 Page tables

### 虚拟地址

通过虚拟地址映射到物理地址，实现了操作系统的隔离性，这里非常关键，因为操作系统不希望一个程序干扰另一个程序的内存。



#### 虚拟地址的映射

CPU运行，然后通过MMU对程序的虚拟地址进行转换（有一个satp表，每个程序都不同，并且该表只能通过特权指令修改（内核态）），然后再到物理地址；

每个虚拟地址要映射到物理地址，需要读取index找到所在的页，然后再通过offset找到相应的位置，64位的寄存器而言，虚拟地址index只有27位，offset只有**12**位，最高的25位并不使用，也就限制了虚拟地址最多只有2^39^的大小，大概就是512G。

而一般设计会使得虚拟内存小于物理内存，防止用尽。

![image-20230630194601239](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230630194601239.png)

实际映射会并不一样，例如RISC-V就是将index分为L0~L2，分为三级页表，每级都会索引向下一级页表，直到最后一级，然后调用offset找到物理地址。



#### 多级页表

参考：[多级页表](https://blog.csdn.net/ibless/article/details/81275009)

优点：

1. 可以让页表在物理内存中离散存储；
2. 可以节省页表内存（因为可以更精细的划分内存，一级页表一经创建，就需要4M）；

缺点：

- 增加寻址次数，延长了访存时间，所以页表并不是越多级别越好，RISC-V就是三级页表。

正因多级页表的缺点，有个**TLB**，会保存最近访问的页表，如需访问，直接查找TLB。

如果切换页表，需要刷新TLB，否则会访问错误的地址。

![image-20230630203355301](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230630203355301.png)

**注意：多级页表的查找的编号必须是物理地址，否则会导致递归查找，没有意义。**



#### 页的大小

4kb=4096b=2^12^b，大多数操作系统都支持4KB，这也是为什么最后12位是offset的原因。

