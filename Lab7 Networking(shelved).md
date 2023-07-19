# Lab7 Networking（已搁置）

一看就很难奥，争取两天做完，做实验之前请复习第五章；

我的[项目地址](https://github.com/monifeng/6.s081.git)

## 实验背景

> You'll use a network device called the E1000 to handle network communication. To xv6 (and the driver you write), the E1000 looks like a real piece of hardware connected to a real Ethernet local area network (LAN). In fact, the E1000 your driver will talk to is an emulation provided by qemu, connected to a LAN that is also emulated by qemu. On this emulated LAN, xv6 (the "guest") has an IP address of **10.0.2.15**. Qemu also arranges for the computer running qemu to appear on the LAN with IP address **10.0.2.2**. When xv6 uses the E1000 to send a packet to 10.0.2.2, qemu delivers the packet to the appropriate application on the (real) computer on which you're running qemu (the "host").
>
> You will use QEMU's "user-mode network stack". QEMU's documentation has more about the user-mode stack [here](https://wiki.qemu.org/Documentation/Networking#User_Networking_.28SLIRP.29). We've updated the Makefile to enable QEMU's user-mode network stack and the E1000 network card.
>
> The Makefile configures QEMU to record all incoming and outgoing packets to the file `packets.pcap` in your lab directory. It may be helpful to review these recordings to confirm that xv6 is transmitting and receiving the packets you expect. To display the recorded packets:
>
> ```
> tcpdump -XXnr packets.pcap
> ```
>
> We've added some files to the xv6 repository for this lab. The file `kernel/e1000.c` contains initialization code for the E1000 as well as empty functions for transmitting and receiving packets, which you'll fill in. `kernel/e1000_dev.h` contains definitions for registers and flag bits defined by the E1000 and described in the Intel E1000 [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2021/readings/8254x_GBe_SDM.pdf). `kernel/net.c` and `kernel/net.h` contain a simple network stack that implements the [IP](https://en.wikipedia.org/wiki/Internet_Protocol), [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol), and [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) protocols. These files also contain code for a flexible data structure to hold packets, called an `mbuf`. Finally, `kernel/pci.c` contains code that searches for an E1000 card on the PCI bus when xv6 boots.



## 实验要求

> Your job is to complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets. You are done when `make grade` says your solution passes all the tests.

要求还是比较简略，就是完成两个函数，并通过测试，但是明显感觉很困难啊，有一点无从下手的感觉，反正就先跟着hint走，多读一读源码，还要读一读开发手册，按着实验提到的重要的章节来读：

> While writing your code, you'll find yourself referring to the E1000 [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2021/readings/8254x_GBe_SDM.pdf). Of particular help may be the following sections:
>
> - Section 2 is essential and gives an overview of the entire device.
> - Section 3.2 gives an overview of packet receiving.
> - Section 3.3 gives an overview of packet transmission, alongside section 3.4.
> - Section 13 gives an overview of the registers used by the E1000.
> - Section 14 may help you understand the init code that we've provided.

看出来该实验是一个文本量很大的实验，并且阅读材料有点多，需要很耐心，找了一圈没找到中文文档，将就英文读吧，实在遇到不会的就翻译一下；

> 该手册有与以太网相关的控制器。QEMU 模拟 82540EM。浏览 Ch2 了解设备。为写驱动，需要熟悉 Ch3 Ch14 Ch4.1。也需要参考 Ch13。其他章节都是与实验驱动无关的 E1000 的组件。
> 		开始**不用关心细节**，只需了解手册如何组织，便于之后查阅。E1000 有许多高级特性，大多可以忽略，完成本实验只需小部分基本特性。

说实话，文本量有点恶心了 **: )**

找了一下实验要求的[中文翻译](https://www.cnblogs.com/seaupnice/p/15932038.html)，不然看着太头疼了。

### 实验步骤

1. 通过读取 `E1000_TDT` 来获取TX_RING的索引：

   tx_ring是什么呢，经过查阅资料：

   > 也有一个 transmit ring，驱动把想要 E1000 发送的 packet 放在这里。`e1000_init()` 配置两个 rings，大小分别为 `RX_RING_SIZE` 和 `TX_RING_SIZE`。

   tx_ring就是存放**传输packet**的地方；

2. 然后检查 ring 是否溢出了：

   检查该索引下的`E1000_TXD_STAT_DD` 标志位，如果没设置，就是还未结束之前的传输；

3. 读源码

4. 读文档



### 实验结果

说实话，看不进去，我决定搁置一段时间，等以后有空了重新来实现（咕咕咕）。