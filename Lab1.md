## Lab1 Xv6 and Unix utilities

Lab1的实现过程，该实验主要是安装和运行，一个熟悉的过程， 并无特别，照着实验手册做就行了 ^ __ ^

[实验手册](https://pdos.csail.mit.edu/6.828/2021/labs/util.html)



### 0. debug

分屏开窗口，两个shell都应该在xv6目录里操作；

```shell
# 左边窗口
make qemu-gdb

#右边窗口
gdb-multiarch

#在gbd页面内
target remote localhost:26000 # 非常重要，绑定端口！
file user/_ls
b main # break的缩写，打断点
c # continue的缩写
```

运行截图如下

![image-20230608150617225](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230608150617225.png)



### 1. Boot xv6

安装，没啥好说的，顺便提了一下git的使用，快速过掉。

```shell
# 前置
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu

# 克隆仓库
git clone git://g.csail.mit.edu/xv6-labs-2021

# 编译运行
cd xv6-labs-2021
git checkout util
make qemu
```

运行截图

![image-20230607164355133](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230607164355133.png)



### 2. sleep

#### 实验要求

> Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.

简单来讲就是实现一个睡眠的程序，需要自己写在 `user/sleep.c` 里，一定要读一读实验手册里的hints，非常有用并且涉及到一些细节问题！

吸收之前项目的教训，我决定从代码主体结构开始构思，构思好以后再动手写，细节问题可以慢慢调试，但代码的主体结构如果出问题要更改就非常麻烦了。



#### 代码框架

```c
int main(int argc, char * argv[])
{
    // 首先检测输入参数是否合理
    if () 
    {

    }

    // 如果合理，先转换成int，然后开始调用sleep
    
    // printf("sleep %d ticks.\n", t0);


    // 退出
    exit(0);
}
```



#### 细节

- 考虑输入为空或其他
- 将字符串转换为int类型
- 头文件的引用



#### 实验结果

截图如下：

![image-20230607175151071](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230607175151071.png)





### 3. pingpong

#### 实验要求

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

大义就是：用两个管道接收父子进程的数据，父进程发送一个byte，然后子进程接收后打印，然后子进程发送父进程一个byte，最后父进程打印输出。



#### 代码框架

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define W 1 // 读
#define R 0 // 写

int main()
{
    // 创建两个管道


    // fork
    int pid = fork();
    char buf[64];

    // 分情况，父子进程
    if (pid == -1)
    {
        fprintf(2, "fork filed.\n");
        exit(-1);
    }
    else if (pid == 0) // 子进程接收然后打印ping
    {

    }
    else    // 父进程接收然后打印
    {

    }

    // 退出
    exit(0);
}
```



#### 细节

- p[1]是写端，p[0]是读端，建议使用后关闭，可以定义宏来方便理解。
- 检测程序是按照字符串判断的，一开始字符串不能出错（**尤其是“ : ”前面没有空格**，非常坑，调了半天发现是这个问题……太难受了）



#### 实验结果

![image-20230607204229027](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230607204229027.png)



### 4. prime

#### 实验要求

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

用管道编写一个筛选质数的筛选器，可以参考超链接和下图：

目标是使用pipe和fork来设置管道。第一个进程将数字2到35输入到管道中。对于每个素数，您将安排创建一个进程，该进程通过一个管道从左边的邻居读取数据，并通过另一个管道向右边的邻居写入数据。由于 xv6的文件描述符和进程数量有限，因此第一个进程可以停留在35个。

![image-20230607204654067](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230607204654067.png)



#### 思路

因为稍微有一点复杂，建议理一理思路后再动手。

每次将符合要求的全部传入管道中，第一个传的一定是素数，直接打印，然后将其他符合要求的再次传入管道并传给子进程。我自己的方法是递归处理，并没有完全利用到多进程的性能，如果有改进的办法，可以提出来讨论一下。

一开始想的是循环，但是循环就比较烧脑，因为子进程还会创建子进程，非常痛苦，最后选择递归。

总的来说实现比较让人难受，想了挺久的，最后还是参考了别人的思路。



#### 代码框架

```c
// 示例伪代码如下
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```



```c
void pipe_prime(const int *p)
{   
	// 传入并输出素数。
    while((read(p[READ_P], &num, 4)) != 0)
    {

    }

	// 递归调用该函数
    
    // 等待子进程结束，很重要
}

int main()
{
    int p[2];
    pipe(p);

    // 先全部传，然后在函数中处理
    for (int i = 2; i <= 35; ++i)
    {
        write(p[WRITE_P], &i, 4);
    }

    close(p[WRITE_P]);
    pipe_prime(p);
    close(p[READ_P]);

    exit(0);
}
```



#### 细节

- 需要关闭管道的文件描述符以免资源耗尽；
- fork函数返回两个值：父进程返回子进程的pid，子进程返回0；
- 需要等待子进程结束，否则结果可能不全。



#### 实验结果

![image-20230608165542882](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230608165542882.png)



### 5. find

#### 实验要求

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`.

写一个在目录树中查找含有特定名字的程序；



#### 思路

1. 输入字符串（可能有多个）；
2. 递归获取所有的文件名（包括目录）；
3. 找出匹配的文件名；
4. 打印；



#### 代码框架

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

int find(char * dir, char * path)
{
	// 参考ls定义变量和结构体，注意该函数我加了最后是否查找成功的功能，所以需要一个变量来判断。

    if ((fd = open(dir, 0)) < 0)
    {
        fprintf(2, "find: cannot open %s\n", dir);
        return -1;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return -1;
    }

    strcpy(buf, dir);
    p = buf + strlen(buf);
    *p++ = '/';

    while((read(fd, &de, sizeof(de))) == sizeof(de))
    {
		// 判断文件状态，普通文件就直接比较字符串，文件夹直接递归调用即可。
    }
    return findSuccess;
}

int main(int argc, char* argv[])
{
    if (argc < 2 || argc > 3)
    {
        fprintf(2, "Usage: find ...");
        exit(0);
    }
    int flag;
    
    if (argc == 2)
    {
        flag = find(".", argv[1]);
    }
    else flag = find(argv[1], argv[2]);
    if (flag != 1)
    {
        fprintf(2, "find error.\n");
        exit(-1);
    }

    exit(0);
}

```



#### 细节

- 跳过de.inum == 0 和 de.name == "." 以及 de.name == ".."(使用strcmp来进行字符串的判断)；
- 只能使用user.h中的函数，我妄图使用strcat，但是发现并没有, 其实也不需要strcat；
- 查找目录最好还是递归调用。

#### 实验结果

总体来说不是很难，但是很多步骤需要参考ls.c，所以有些繁琐，慢慢理一理还是能理清楚的。

![image-20230613210504211](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230613210504211.png)



### 6. xargs

#### 实验要求

> Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file `user/xargs.c`.

编写一个简单版本的 UNIX xargs 程序: 从标准输入中读取行，并为每一行运行一个命令，将该行作为参数提供给命令。您的解决方案应该在 user/xargs.c 文件中。



#### 思路

没什么头绪，我也借鉴了别人的思路，过后想起来，觉得自己之前没懂的地方就是如何从管道中读取到来的参数，其实直接read（0，buf，sizeof buf）就可以了， 因为管道相当于走的标准输入流。

- 首先创建字符串数组存放行参数，可以用index来表示执行的顺序；
- 从管道中循环读取参数，遇到换行符，就直接跳过；
- 子进程执行，父进程等待。



#### 代码框架

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"

const int MAXN = 512;

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(2, "Useage: xargs ...");
        exit(-1);
    }

    int i, n;
    char* args[MAXARG];
    char buf[MAXN], buf_child[MAXN/2];
    char *p1;
    char *p2;

    // 读取到args中，方便后续执行,注意这里的i是整个程序通用的变量，并不是局部变量;
    for (i = 0; i < argc - 1; ++i)
    {

    }
    p1 = buf; // 指向buf的头部

    while ((n = read(0, p1, MAXN)) > 0)
    {
        	// read已经对buf填充了，这里是在调整指针的位置，防止覆盖
    }
    p1 = buf; // 再次指向头部

    // 循环查找\n，直到字符串的结尾
    while ((p2 = strchr(p1, '\n')) != (char *)0)
    {
        *p2 = 0; // 替换为字符串的结束符
        if (fork() == 0)
        {
            // 子进程的p1并不影响父进程，所以可以随意修改p1而不影响主程序
            memmove(buf_child, p1, strlen(p1));
            p1 = buf_child;
            while((p2 = strchr(p1, ' ')) != (char*) 0)
            {
                // 以空格为分隔符，来分割字符串，strlen在找到0(字符串结束符)时会停止计算
				// 调整到空格的下一位，然后重新开启循环
            }
            // 最后一个空格，循环无法处理，所以到外面处理
			
            exec(argv[1], args);
            exit(0);
        }
        else 
        {
            p1 = p2 + 1;    // 父进程仅仅是读取字符串，交给子进程处理，所以这里需要跳到下一个字符串的位置
            wait(0); 
        }   
    }

    exit(0);
}
```



#### 细节

- 可能需要修改find（），以达到作为文件使用的目的（./a这种形式）；
- 三个缓冲区， 每个都有不同的意义；
- i非常奇妙，专门将其 `main `函数范围内的局部变量，生命周期和main函数一致而非 `for` 循环；
- 父进程一定要wait；
- 两个指针，交替使用，来达到分割字符串的目的。

#### 实验结果

总的来说是一个比较综合性的实验，几乎结合了前面的所有知识点， 也复习了很多知识点，收获良多，但其中有一些关于pipe的操作还不太熟练，比如0就是管道默认输入流，以及对空间分配的使用，在使用memmove之前，一定要先分配空间；对于字符串也有一定收获，因为字符串的结尾可以用0表示，此时字符串指针如果需要使用strlen之类的，就会在0时停止，打印和使用指针也是，在遇到0之后就会停止。

![image-20230613222120878](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230613222120878.png)



### Grade

后续的other实验感觉有些挺有意思的，有时间的话做来看看， 但现在忙着准备期末复习，所以就先鸽了。

综合检测结果如下：

需要在文件目录下的命令行中输入 `make grade` ，注意：需要创建一个time.txt，内容就是你完成的小时数（一个数字即可，单位默认为小时）;

![image-20230614114944140](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230614114944140.png)

### Other

挑战项目：后续有点想做sh的自动补全和上键显示之前的代码，不然和linux的shell用起来感觉不一样，不太方便。

后面有机会一定更新！（可能也许大概不鸽）
