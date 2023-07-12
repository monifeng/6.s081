# Lec6 isolation and system call Entry_exit

### 1. 内核态到用户态的转换

完整的流程，其中 `uservec` 和 `usere` t都是汇编代码，其余都是c语言代码。

![imag![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230710204758923.png)e-20230706202346207](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230706202346207.png)



### 2. ecall做的三件事

1. 转换到管理员模式（supervisor）；
2. 将用户函数的程序计数器（pc）保存到sepc中；
3. 跳转到stvec指向的指令（trap处理程序的地址）；



### 3. 保存用户寄存器的状态

#### 保存寄存器的原因

从ecall开始，即将执行系统调用，想要恢复到用户函数被trap的状态，就需要保存寄存器，然后执行完毕从ecall返回后，加载这些被保存的寄存器。



### 不保存在用户程序栈中的原因

某些语言并没有用户栈，为了安全性，内核不能假定用户栈以什么形式保存寄存器的状态，所以为了安全性考虑，需要保存在内核中；



#### 成功转换到内核页表后，不会立刻执行其他函数

通过 `trampoline `转换到内核页表时，最后一行依然是 `trampoline` 的地址，也就是说，程序会继续执行 `trampoline` ，建议观看[原视频](https://www.bilibili.com/video/BV19k4y1C7kA?t=3476.5&p=5)，往前拉1~2min，没看懂就反复多看几遍，非常巧妙的一个设计，莫里斯大神讲的也非常通透。

所以叫 `trampoline` （蹦床），你能在内核态和用户态来回弹跳 :  ) 

执行完这一切后，跳回到 `trampoline` ，然后执行 `usertrap` ；



#### 如何保存寄存器状态

- trapframe保存了32个空的寄存器插槽，方便保存寄存器的状态；

- 系统调用允许交换寄存器的地址（sscratch），里面保存的是trapframe的地址；

  一般步骤是：交换trapframe和某个寄存器的地址，然后在原有寄存器的上通过trapframe来保存用户程序执行时的所有寄存器状态（有点难理解，建议去看看[原视频](https://www.bilibili.com/video/BV19k4y1C7kA?t=2553.1&p=5)，已经定位好了，往前拉1~2min进度条就行）；

  ![image-20230706205922978](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230706205922978.png)

