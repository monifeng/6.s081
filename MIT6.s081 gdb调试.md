# MIT6.s081 gdb调试

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

最好打开tui来方便查看

```shell
# gdb页面内
tui enable
layout reg # 查看寄存器
layout asm # 查看汇编代码
```

![image-20230711202014532](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711202014532.png)



一切就绪以后，按c开始调试，并在左边调用想要调试的程序；

![image-20230711202113035](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230711202113035.png)

逐步按s即可慢慢调试了。

