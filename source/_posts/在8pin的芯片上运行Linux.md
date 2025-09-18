---
title: 在8pin的芯片上运行Linux
author: WangGaojie
---

今年早些时候我在[Dmitry Grinberg](https://dmitry.gr/index.php?r=05.Projects&proj=36.%208pinLinux)的blog上看到一个有趣的项目，他用3个8pin的芯片来运行Linux。这个项目让我进一步理解现代计算机的工作原理，所以我决定自己尝试实现这个项目。

原项目通过引脚时序上的一些复用逻辑来让8pin的STM32G031得以同时驱动一片PSRAM和SD卡，但是这种用法看起来非常的麻烦，我想找一种更简单的方法来达到这个目的。

后来我接触到[RV32IMA](https://github.com/cnlohr/mini-rv32ima)项目，该模拟器方案视乎要比uMIPS要更简单，可以运行更简单的Linux固件，这样就可以不使用SD卡了，通过一片NorFlash来存放一个简单的Linux固件。

现在我的想法就是通过4片8pin的芯片来运行Linux，分别是STM32G031J6M6，APS64064L，W25Q64JV，CH340N。这4个芯片的组合可以构建一个直接插在电脑USB接口上运行的8pin的Linux系统。

让我们来思考一下如何连接4个芯片。这似乎是一个非常具有挑战性的问题。

各个部件应该这样连接起来，STM32需要串口一对引脚，一对下载调试引脚，SPI的3个引脚，两个片选信号引脚，还有STM32供电的3个引脚。STM32需要分配11个引脚，这已经超出了STM32G031的8个引脚了。

![硬件连接](Hard.png)

某些引脚或许可以分时复用来达到需求。首先，下载调试引脚只有在芯片启动时需要，芯片正常运行后就不需要了，所以可以复用为普通引脚。此外NorFlash芯片在上电后可以将数据拷贝到APS6404L芯片中，这样系统串口还未使用时就可以使用NorFlash芯片，这又能够节约一个片选信号线。这样减少了3个信号线正好满足8pin芯片的需求。

哪些信号复用是一个需要好好考虑的问题。首先调试信号线固定为STM32芯片的7、8脚，且它能够与其它任何信号进行分时复用。如果SPI要使用硬件SPI的话，4、5、6引脚直接分配到SPI。剩余的2、3脚为电源引脚，最后还剩下1、7、8引脚，这三个引脚只有1脚能够作为硬件串口的RX引脚，剩下的7、8脚分配作为两个SPI芯片的片选信号线。此外8脚还需要承担串口TX的功能，所以8脚的片选必须连接到norFlash的片选。因为NorFlash可以和串口分时复用，当从Flash读取数据时，假定串口输出不可用，且从Flash读取数据只在开机读取一次，对串口的功能影响较小。

最终设计的原理图如下：
![原理图](Schematic.png)

这里除了4个8pin的芯片外，还添加了一个3脚的LDO供电芯片。USB接口选择了印制板型的TYPE-C接口，可以直接插在USB TYPE-C公头上。

最终的实现效果如下，一个非常小巧的结构。PCB板材厚度一定要是0.8mm，只有该厚度能够正好插入TYPE-C公头接口中。

<center>
<table>
    <tr>
        <td><center><img src="layout-top.png"/><br>顶层</center></td>
        <td><center><img src="layout-bottom.png"/><br>底层</center></td>
        <td><center><img src="real.jpg"/><br>实物</center></td>
    </tr>
</table>
</center>

软件的移植参考了[pico-rv32ima](https://github.com/ElectroBoy404NotFound/pico-rv32ima)，我也将STM32的相关代码放在了Github上，[8pin-linux](https://github.com/DazzlingOkami/8pin-linux)。软件实现中除了RV32软核部分，最有学习价值都就是cache的实现，使用该cache方案在一定程度上弥补了外部串行PSRAM读写效率低的状况，解决了RV32软核运行速度与外部IO速度不匹配的问题，这种技术手段可以借鉴到其它项目中。

STM32的固件编译出来接近32kb，ram使用也接近8kb，基本耗尽STM32的存储资源。

Linux固件文件为Image，在STM32的源码相关目录中存在。它需要提前写入到W25Q64芯片中，可用使用JLink等工具来完成。

STM32芯片可以飞线引出SWD引脚，在上电后5秒内连接调试器，连接调试器后需要使用STLink修改芯片的选项字节。将nRST引脚修改为普通GPIO使用，之后再下载STM32的程序。芯片运行5秒后相关的引脚会切换到其它功能，下载调试就不可用了。该SWD接口几乎不具备调试的能力，除非将相关复用功能临时关闭。

该项目几乎没有额外的调试手段来观察RV32内核的运行情况，唯一的手段是使用示波器观察SPI的CS引脚，通过该引脚可以观察软核的内存IO情况，同时也能判断RV32内核死机问题。

可以看看Linux启动日志，时间戳为终端自动添加的：
```text
[15:24:47.759] Start
[15:24:47.769] ********************************************************************************************************************************************************************************************************************************************** // 重复输出，此时是STM32在将Linux固件从NorFlash中写入到APS64064L中。
Image loaded sucessfuly!
[15:25:33.150] Linux version 6.1.14mini-rv32ima (user@buildvm) (riscv32-buildroot-linux-uclibc-gcc.br_real (Buildroot -g61a3fe7e) 12.2.0, GNU ld (GNU Binutils) 2.38) #3 Thu May  4 18:56:08 UTC 2023
[15:25:33.315] Machine model: riscv-minimal-nommu,qemu
[15:25:33.346] Zone ranges:
[15:25:33.390]   Normal   [mem 0x0000000080000000-0x0000000080ffefff]
[15:25:33.425] Movable zone start for each node
[15:25:33.459] Early memory node ranges
[15:25:33.499]   node   0: [mem 0x0000000080000000-0x0000000080ffefff]
[15:25:33.540] Initmem setup node 0 [mem 0x0000000080000000-0x0000000080ffefff]
[15:25:33.679] riscv: base ISA extensions aim
[15:25:33.719] riscv: ELF capabilities aim
[15:25:33.780] Built 1 zonelists, mobility grouping off.  Total pages: 4063
[15:25:33.829] Kernel command line: earlycon=uart8250,mmio,0x10000000,1000000 console=hvc0
[15:25:33.880] Unknown kernel command line parameters "earlycon=uart8250,mmio,0x10000000,1000000", will be passed to user space.
[15:25:33.938] Dentry cache hash table entries: 2048 (order: 1, 8192 bytes, linear)
[15:25:34.080] Inode-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[15:25:34.130] mem auto-init: stack:off, heap alloc:off, heap free:off
[15:25:34.169] Memory: 13340K/16380K available (1445K kernel code, 269K rwdata, 146K rodata, 874K init, 108K bss, 3040K reserved, 0K cma-reserved)
[15:25:34.227] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[15:25:34.269] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[15:25:34.309] riscv-intc: 32 local interrupts mapped
[15:25:34.448] clint: clint@11000000: timer running at 1000000 Hz
[15:25:34.500] clocksource: clint_clocksource: mask: 0xffffffffffffffff max_cycles: 0x1d854df40, max_idle_ns: 3526361616960 ns
[15:25:34.550] sched_clock: 64 bits at 1000kHz, resolution 1000ns, wraps every 2199023255500ns
[15:25:34.600] Console: colour dummy device 80x25
[15:25:34.839] printk: console [hvc0] enabled
[15:25:35.318] Calibrating delay loop (skipped), value calculated using timer frequency.. 2.00 BogoMIPS (lpj=10000)
[15:25:35.490] pid_max: default: 4096 minimum: 301
[15:25:37.777] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[15:25:38.149] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[15:25:51.993] devtmpfs: initialized
[15:26:06.741] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[15:26:50.500] clocksource: Switched to clocksource clint_clocksource
[15:29:15.250] workingset: timestamp_bits=30 max_order=12 bucket_order=0
[15:38:46.700] Freeing unused kernel image (initmem) memory: 872K
[15:38:46.818] This architecture does not have kernel memory protection.
[15:38:47.094] Run /init as init process
[15:49:25.583] Welcome to pico-rv32ima Linux
[15:51:33.919] Jan  1 00:00:49 login[32]: root login on 'console'
[15:51:34.185] 
[15:52:19.570] ~ # 
[15:52:57.861] ~ # 
[15:52:59.748] ~ # uname -a
[15:53:15.205] Linux pico 6.1.14mini-rv32ima #3 Thu May  4 18:56:08 UTC 2023 riscv32 GNU/Linux
[15:53:16.133] ~ # 
[15:53:17.588] ~ # uname -a
[15:53:23.321] Linux pico 6.1.14mini-rv32ima #3 Thu May  4 18:56:08 UTC 2023 riscv32 GNU/Linux
[15:53:24.100] ~ # 
[15:53:25.459] ~ # free
[15:53:36.194]               total        used        free      shared  buff/cache   available
[15:53:36.287] Mem:          14212        1892       10984           0        1336       10376
[15:53:41.886] ~ # 
[15:53:43.249] ~ # echo hello
[15:54:10.550] hello
[15:54:11.507] ~ # 
[15:54:12.871] ~ # pwd
[15:54:29.969] /root
[15:54:30.643] ~ # 
[15:54:32.110] ~ # ls
```

第30秒成功加载固件。
第46秒输出第一行Linux启动消息。
第64秒初始化设备文件系统。
第14分执行初始程序。
第27分42秒启动终端，显示命令提示符。
...

启动花费了将近半小时是让我很意外的。执行pwd和uname -a命令都能正常响应。准备执行ls命令，等了很久都没有输出结果，我认为它应该是还在慢慢运行吧！也有可能是系统挂了！😅

该软件方案在550MHz的STM32H7芯片上配合高速的SDRAM，可以非常流畅的运行Linux，终端交互都没有任何问题。除了STM32G031芯片的主频低外，外部的串行PSRAM带宽低也是导致Linux运行慢的主要原因。
