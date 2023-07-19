# Lab8：locks

开始实验之前，请阅读：

> - Chapter 6: "Locking" and the corresponding code.
> - Section 3.5: "Code: Physical memory allocator"
> - Section 8.1 through 8.3: "Overview", "Buffer cache layer", and "Code: Buffer cache"

中文资料我放在了[项目地址](https://github.com/monifeng/6.s081.git)；



## Memory allocator

### 实验要求

> Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty. You must give all of your locks names that start with "kmem". That is, you should call `initlock` for each of your locks, and pass a name that starts with "kmem". Run kalloctest to see if your implementation has reduced lock contention. To check that it can still allocate all of memory, run `usertests sbrkmuch`. Your output will look similar to that shown below, with much-reduced contention in total on kmem locks, although the specific numbers will differ. Make sure all tests in `usertests` pass. `make grade` should say that the kalloctests pass.

下面这段很重要：

```
	To remove lock contention, you will have to redesign the memory allocator to avoid a single lock and list. The basic idea is to maintain a free list per CPU, each list with its own lock. Allocations and frees on different CPUs can run in parallel, because each CPU will operate on a different list. The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. Stealing may introduce lock contention, but that will hopefully be infrequent.
```

大概意思就是，为每个CPU设计锁，并每个CPU维护一个空闲链表，并需要实现“偷取”空闲链表的操作；



### 实验步骤

1. 修改kmem，将其设置为数组，让每个CPU拥有一个该结构体；
2. 让freerange分配内存给当前CPU，需要获取cpuid，注意此时需要关闭中断；
3. 修改 `kalloc` ，实现偷取功能；



### 代码框架

```c
void *
kalloc(void)
{
  struct run *r;

  // 关闭中断，并获取当前CPUid
  push_off();
  int cid = cpuid();

  acquire(&kmem[cid].lock);
    r = kmem[cid].freelist;
  release(&kmem[cid].lock);

  // 如果当前CPU有空闲列表
  if (r)
  {

  }
  // 否则就去偷！
  else 
  {

  }

  pop_off();
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```



```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  // 这里都可以只获取push_off，但是kalloc就不能这样，奇怪，望大神解惑
  push_off();
  int i = cpuid();
  pop_off();

  acquire(&kmem[i].lock);
  r->next = kmem[i].freelist;
  kmem[i].freelist = r;
  release(&kmem[i].lock);
}
```



### 实验细节

1. 在kalloc中，偷取别人的空闲链表需要加锁，但如果此时被偷和偷的是同一个cpu，要注意不能获取两次锁！
2. 算是一个坑，`pop_off` 要在最后释放，看实验描述那里，以为只有获取cpuid才需要关闭中断，看来是需要关闭到分配完内存为止，原理暂时未知；
3. 一定要注意锁的使用范围，分辨清楚临界区；



### 实验结果

成功。

![image-20230718205739799](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230718205739799.png)



## Buffer cache

又是hard奥，这个实验是单独的，与前面的无关；



### 实验要求

> Modify the block cache so that the number of `acquire` loop iterations for all locks in the bcache is close to zero when running `bcachetest`. Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is less than 500. Modify `bget` and `brelse` so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks (e.g., don't all have to wait for `bcache.lock`). You must maintain the invariant that at most one copy of each block is cached. When you are done, your output should be similar to that shown below (though not identical). Make sure usertests still passes. `make grade` should pass all tests when you are done.

修改块缓存，bget，brelse；



### 实验步骤

1. 关键句：**We suggest you look up block numbers in the cache with a hash table that has a lock per hash bucket.**

   这句直接给出了哈希桶的数据结构，直接照搬：

   ```c
   #define NBUCKET 7
   
   struct {
     struct spinlock lock;
     struct buf buf[NBUF];
   
     // Linked list of all buffers, through prev/next.
     // Sorted by how recently the buffer was used.
     // head.next is most recent, head.prev is least.
   } bcache[NBUCKET];
   ```

2. 一个简单的哈希函数：

   ```c
   int hashkey(int key)
   { 
     return key % NBUCKET;
   }
   ```

3. 在binit中初始化所有的锁，注意：buf自带锁，每个buf的锁都需要初始化

   ```c
   binit(void)
   {
     struct buf *b;
     
     
     for(int i=0;i<NBUCKET;i++) 
     {  
         	for(b = bcache[i].buf; b < bcache[i].buf+NBUF; b++){ 
                initsleeplock(&b->lock, "buffer");
            }
         initlock(&bcache[i].lock, "bcache.bucket");
     }
   }
   ```

4. buf结构体中新增：

   ```c
   struct buf {
     int valid;   // has data been read from disk?
     int disk;    // does disk "own" buf?
     uint dev;
     uint blockno;
     struct sleeplock lock;
     uint refcnt;
     struct buf *prev; // LRU cache list
     struct buf *next;
     uchar data[BSIZE];
     uint time_stamp; // 时间戳
     int bucket; // 属于哪个桶
   };
   ```

5. 然后就是修改 `bget` ，非常重要的一点：**sleep-lock保护的是块的缓冲内容的读写，而bcache.lock保护被缓存块的信息。**

   ```c
   // Look through buffer cache for block on device dev.
   // If not found, allocate a buffer.
   // In either case, return locked buffer.
   static struct buf*
   bget(uint dev, uint blockno)
   {
     struct buf *b;
     int i = hashkey(blockno);
     uint min_time_stamp = -1; // 因为类型是uint，其实这里是最大的uint, 2^16-1;
     struct buf *min_b = 0;
   
     acquire(&bcache[i].lock);
   
     // Is the block already cached?
     for (b = bcache[i].buf; b != bcache[i].buf + NBUF; ++b)
     {
       if (b->dev == dev && b->blockno == blockno)
       {
         b->refcnt ++;
         release(&bcache[i].lock);
         acquiresleep(&b->lock);
         return b;
       }
       if (b->refcnt == 0 && b->time_stamp < min_time_stamp)
       {
         min_b = b;
         min_time_stamp = b->time_stamp;
       }
     }
     // Not cached.
     // 直接在时间戳最小的buf上分配
     b = min_b;
     if (b != 0)
     {
       b->dev = dev;
       b->blockno = blockno;
       b->valid = 0; // 已从磁盘读取了数据
       b->refcnt = 1;
       release(&bcache[i].lock);
       acquiresleep(&b->lock);
       return b;
     }
     
     panic("bget: no buffers");
   }
   ```

6. 修改 `brelse`，**释放之前在bget()调用的睡眠锁**：

   ```c
   // Release a locked buffer.
   // Move to the head of the most-recently-used list.
   void
   brelse(struct buf *b)
   {
     if(!holdingsleep(&b->lock))
       panic("brelse");
   
     releasesleep(&b->lock);
     int i = hashkey(b->blockno);
   
     acquire(&bcache[i].lock);
     b->refcnt --;
     if (b->refcnt == 0)
     {
       b->time_stamp = 0;
     }
     release(&bcache[i].lock);
   }
   ```

7. 注意：将NBUCKET改成小一点的质数（比如7），否则锁的数目会超出限制！（推测是文件描述符不够用）；



### 实验结果

成功。

![image-20230718235516432](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230718235516432.png)



## Make grade

总体来说，第二个实验有点难度，比较难想到。我也参考了一下别人的思路，仔细一想，有些地方确实比较坑：尤其是那个hash bucket，我读实验要求的时候，翻来覆去，没读懂，看了别人的思路才知道，就是个哈希表嘛，专门用来装buf数组的；buf数组的思路也就和第一个实验非常像，实在不应该，下次多思考思考！

![image-20230718235727800](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230718235727800.png)