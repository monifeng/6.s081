# Lab9: file system

这几天做Lab的速度快了很多，因为我没看课程，直接看的课本，发现自己真的不适合看视频，看着看着就要睡着了，莫里斯教授的还好，另一个教授讲的真的让我昏昏欲睡，每个人都有自己的学习方法，大家也不用盲信网上讲的各种学习方法，适合自己的最重要。再者课程上讲的东西，基本都是课本的内容，课程内容主要还是带着研究课本上提到的源码，但如果自己去读的话，明显会有些吃力，但就我个人而言，我觉得读不通就多读两遍，这个视频看了感觉不进脑子，不太适合我，再者时间有点紧张了，没空去慢慢看视频了，如果做Lab或者读课本读的很吃力再去看视频也来得及。

我的[项目地址](https://github.com/monifeng/6.s081.git)

本实验前置要求：

> Before writing code, you should read "Chapter 8: File system" from the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf) and study the corresponding code.

预感此图可能会很有用：

![image-20230719203236661](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230719203236661.png)

## Large files

### 实验要求

> In this assignment you'll increase the maximum size of an xv6 file. Currently xv6 files are limited to 268 blocks, or 268*BSIZE bytes (BSIZE is 1024 in xv6). This limit comes from the fact that an xv6 inode contains 12 "direct" block numbers and one "singly-indirect" block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256=268 blocks.

本次实验很多都和文本相关联，可能我会引用大量课本内容，并指出它们的位置，方便大家阅读，因为本章文本量很大，所以推荐阅读一下中文文档，第一次阅读也不需要太深刻理解，知道什么位置即可，后面做实验用到知道在哪里查阅就行了。

这个实验应该是要修改 8.10 inode content 这小节的代码。

![image-20230719204628291](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230719204628291.png)

需要查看的代码：`struct dinode` in `fs.h`，`bmap()` in `fs.c`，

#### Your job

> Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you're not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); the 13th should be your new doubly-indirect block. You are done with this exercise when `bigfile` writes 65803 blocks and `usertests` runs successfully:



### 实验步骤

1. 画出一个简略的映射二级间接块的图，并排布二级间接块号的位置，明显需要排在最后，方便 `bmap` 来进行操作，画工不好，但图还是比较简单明了，将就看看哈；

   ![image-20230719213918338](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230719213918338.png)

2. 修改 `NDIRECT` 为11，并将 ` struct inode` 和 `struct dinode` 中修改为 ` uint addrs[NDIRECT+2];`

3. 修改 `bmap()`函数，让其能够映射二级间接块；

   ```c
   static uint
   bmap(struct inode *ip, uint bn)
   {
     uint addr, *a;
     struct buf *bp;
   
     if(bn < NDIRECT){
       if((addr = ip->addrs[bn]) == 0)
         ip->addrs[bn] = addr = balloc(ip->dev);
       return addr;
     }
     bn -= NDIRECT;
   
     if(bn < NINDIRECT){
       // Load indirect block, allocating if necessary.
       if((addr = ip->addrs[NDIRECT]) == 0)
         ip->addrs[NDIRECT] = addr = balloc(ip->dev);
       // 查阅间接块
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       if((addr = a[bn]) == 0){
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     // 开始映射二级间接块
     bn -= NINDIRECT;
     // NDOUINDIRECT定义在了fs.h中，大小当然是NINDIRECT * NINDIRECT
     if (bn < NDOUINDIRECT)
     {
       uint offset = bn % NINDIRECT;
       bn /= NINDIRECT;
       // 一级
       if((addr = ip->addrs[NDIRECT+1]) == 0)
         ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       if((addr = a[bn]) == 0){
         a[bn] = addr = balloc(ip->dev);
       }
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       // 二级
       bn = offset;
       if ((addr = a[bn]) == 0)
       {
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     panic("bmap: out of range");
   }
   ```

4. 修改 `itrunc` 函数，让其能够释放使用过的二级间接块

   ```c
   void
   itrunc(struct inode *ip)
   {
     int i, j;
     struct buf *bp;
     uint *a;
   
     for(i = 0; i < NDIRECT; i++){
       if(ip->addrs[i]){
         bfree(ip->dev, ip->addrs[i]);
         ip->addrs[i] = 0;
       }
     }
   
     if(ip->addrs[NDIRECT]){
       bp = bread(ip->dev, ip->addrs[NDIRECT]);
       a = (uint*)bp->data;
       for(j = 0; j < NINDIRECT; j++){
         if(a[j])
           bfree(ip->dev, a[j]);
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT]);
       ip->addrs[NDIRECT] = 0;
     }
   
     // 释放二级块
     if(ip->addrs[NDIRECT+1]){
     bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
     a = (uint*)bp->data;
   
     for(j = 0; j < NINDIRECT; j++){
       struct buf *bp1 = bread(ip->dev, a[j]);
       uint *a1 = (uint*)bp->data;
       for (int k = 0; k < NINDIRECT; ++k)
       {
         if (a1[k])
           bfree(ip->dev, a1[k]);
       }
   
       brelse(bp1);
       bfree(ip->dev, a[j]);
     }
     brelse(bp);
     bfree(ip->dev, ip->addrs[NDIRECT]);
     ip->addrs[NDIRECT] = 0;
     }
   
     ip->size = 0;
     iupdate(ip);
   }
   ```




### 实验细节

1. 非常细节的一个地方，要将 `dinode` 和 `inode` 中的addr数组修改为+2，一边保持总量不变；

   ```c
   // On-disk inode structure
   struct dinode {
     // ...
     uint addrs[NDIRECT+2];   // Data block addresses  // lab9-1
   };
   
   // in-memory copy of an inode
   struct inode {
     // ...
     uint addrs[NDIRECT+2];    // lab9-1
   };
   ```

2. 在我运行的时候，始终显示如下错误：

   ```shell
   panic: virtio_disk_intr status
   ```

   原因不是特别懂，但我自己的问题出在 `NINDIRECT` 和 `NDIRECT` 上，在计算二级块的时候，因为需要除以一级间接块（想想为什么）来获得第一级的块数，然后加一个偏移量，需要 `bn%NINDIRECT` ，很容易写成NDIRECT，而且这个错误比较难发现，因为运行始终没问题，但就是会报错，网上查了一些资料又检查很久代码才发现问题。



### 实验结果

成功！

![image-20230722215747227](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230722215747227.png)



## Symbolic links

### 实验要求

> You will implement the `symlink(char *target, char *path)` system call, which creates a new symbolic link at path that refers to file named by target. For further information, see the man page symlink. To test, add symlinktest to the Makefile and run it. Your solution is complete when the tests produce the following output (including usertests succeeding).

实现symlink，增加一个系统调用函数。



### 实验步骤

1. 添加系统调用：`user/usys.pl`  ,`user/user.h`, `kernel/syscall.h`, 在 `kerne/syscall.c` 文件中添加一个空函数 `sys_symlink`。

2. kernel/stat.h 中 添加一个新的文件类型 `T_SYMLINK`；

3. kernel / fcntl.h 中添加一个新的 flag类型，并且在Makefile中添加 `$U/_symlinktest\`；

4. 实现symlink，第一个参数是被链接的文件，第二个参数是该软链接的路径；

   > You will need to choose somewhere to store the target path of a symbolic link, for example, in the inode's data blocks. 

   毫无疑问，需要参考硬链接，它们的行为大多都是一致的，区别在于软链接创建的文件类型是 `T_SYMLINK`，而硬链接直接连接到另一个inode，符号链接相当于一个特殊的独立的文件, 其中存储的数据即目标文件的路径；

   ```c
   uint64 sys_symlink(void)
   {
     char name[DIRSIZ], new[MAXPATH], old[MAXPATH];
     struct inode *dp, *ip;
     int n;
   
     if(argstr(0, old, MAXPATH) < 0 || argstr(1, new, MAXPATH) < 0)
       return -1;
   
     // 操作文件之前必须的操作
     begin_op();
     // 创建一个软链接文件
     if (ip = create(new, T_SYMLINK, 0, 0) == 0)
     {
       end_op();
       return -1;
     }
   
     // 写入target文件路径
     if (writei(ip, 0, (uint64)new, 0, n) != n)
     {
       iunlockput(ip);
       end_op();
       return -1;
     }
   
     iunlock(ip);
     end_op();
     return 0;
   }
   ```

   

5. 修改 `open` 系统调用，因为当文件类型是符号链接时，系统希望能够直接到被链接的文件那里，所以需要递归调用，但是要小于10次（不需要单独判断是否存在环，超过10次直接中断就行了）。

   设计一个单独的递归查找函数：

   ```c
   struct inode* follow_symlink(struct inode* ip) {
     uint inums[MAXFOLLOW];
     int i, j;
     char target[MAXPATH];
   
     for(i = 0; i < MAXFOLLOW; ++i) {
       inums[i] = ip->inum;
       // read the target path from symlink file
       if(readi(ip, 0, (uint64)target, 0, MAXPATH) <= 0) {
         iunlockput(ip);
         printf("open_symlink: open symlink failed\n");
         return 0;
       }
       iunlockput(ip);
       
       // get the inode of target path 
       if((ip = namei(target)) == 0) {
         printf("open_symlink: path \"%s\" is not exist\n", target);
         return 0;
       }
       for(j = 0; j <= i; ++j) {
         if(ip->inum == inums[j]) {
           printf("open_symlink: links form a cycle\n");
           return 0;
         }
       }
       ilock(ip);
       if(ip->type != T_SYMLINK) {
         return ip;
       }
     }
   
     iunlockput(ip);
     printf("open_symlink: the depth of links reaches the limit\n");
     return 0;
   }
   ```

6. 最后修改 `open` ：

   ```c
     if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
       iunlockput(ip);
       end_op();
       return -1;
     }
     
     // 增添一个T_SYMLINK的判断
     if(ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
       if((ip = follow_symlink(ip)) == 0) {
         // 此处不用调用iunlockput()释放锁,因为在follow_symlinktest()返回失败时ip的锁在函数内已经被释放
         end_op();
         return -1;
       }
     }
   
     if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
       if(f)
         fileclose(f);
       iunlockput(ip);
       end_op();
       return -1;
     }
   ```



### 实验结果

成功。

![image-20230723110356253](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230723110356253.png)



## Make grade

比较复杂的实验，有些地方写的比较恶心，而且调试并不好调试，debug的有点崩溃；感觉最近各种事情比较多，有点浮躁了，静不下心来，参考了别人的实现。

决定休息一天，最后一个实验可能延后几天再完成。

![image-20230723111317688](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230723111317688.png)

