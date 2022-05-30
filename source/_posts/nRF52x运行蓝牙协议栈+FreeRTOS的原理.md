---
title: nRF52x运行蓝牙协议栈+FreeRTOS的原理
author: WangGaojie
---

> 之前有个研发的产品为了满足蓝牙连接的需求，选择了nRF52840这个芯片，它的蓝牙协议栈是一种类似使用动态库的方式进行调用，官方仅仅给出了协议栈的二进制包(SoftDevice)，并给出了动态调用的方法，其调用的关键就是使用ARM单片机的SVC调用，这是一种比较好的设计，我们开发软件能够非常简单的调用协议栈的程序来实现蓝牙通信相关的功能。
后来产品的使用需求发生的变化，蓝牙功能已经不需要的，而是需要更加复杂的通信功能。为了保持业务功能代码的一致性，所以并没有替换芯片，但是由于通信功能更加复杂后，迫不得已需要RTOS的支援。后续就是移植FreeRTOS等一系列操作。因为该单片机是CM4内核，所以这些操作都很简单。

# SVC调用冲突的疑惑
今天在看这两套基于nRF单片机的设计方案时，我对其中的有个细节产生了疑惑。了解FreeRTOS的开发者应该比较清楚，像CM4内核的单片机启动FreeRTOS的第一个任务时，依赖SVC调用从裸机运行切换为RTOS线程运行。而蓝牙协议栈也使用了SVC调用，那是不是意味着该单片机的蓝牙功能不能和FreeRTOS兼容，结果是否定的。因为从官方的例程中可以看到蓝牙+FreeRTOS的例程(examples\ble_peripheral\ble_app_hrs_freertos)。我决定弄清楚Nordic的蓝牙协议栈+FreeRTOS是如何兼容的。

我起初认为FreeRTOS的port实现中对初始任务调用有区别一般单片机的特殊处理，但是仔细看它的port文件后又发现并没有什么特别的。

```c
// external\freertos\portable\ARM\nrf52\port.c

__asm void vPortSVCHandler( void )
{
    PRESERVE8

    /* Get the location of the current TCB. */
    ldr r3, =pxCurrentTCB
    ldr r1, [r3]
    ldr r0, [r1]
    /* Pop the core registers. */
    ldmia r0!, {r4-r11, r14}
    msr psp, r0
    isb
    mov r0, #0
    msr basepri, r0
    bx r14
}

__asm void vPortStartFirstTask( void )
{
    PRESERVE8
    EXTERN __Vectors

    /* Use the NVIC offset register to locate the stack. */
    ldr r0, =__Vectors
    ldr r0, [r0]
    /* Set the msp back to the start of the stack. */
    msr msp, r0
    /* Globally enable interrupts. */
    cpsie i
    cpsie f
    dsb
    isb
#ifdef SOFTDEVICE_PRESENT
    /* Block kernel interrupts only (PendSV) before calling SVC */
    mov r0, #(configKERNEL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))
    msr basepri, r0
#endif
    /* Call SVC to start the first task. */
    svc 0

    ALIGN
}
```

而在FreeRTOSConfig.h文件中也将该函数映射到了中断向量表中

```c
// examples\ble_peripheral\ble_app_hrs_freertos\config\FreeRTOSConfig.h

/* Definitions that map the FreeRTOS port interrupt handlers to their CMSIS
standard names - or at least those used in the unmodified vector table. */

#define vPortSVCHandler                                       SVC_Handler
#define xPortPendSVHandler                                    PendSV_Handler
```

到这里我转头分析nRF的SDK代码，例如sd_ble_enable()函数就是一个SVC调用在ble.h文件中使用一个SVCALL宏定义扩展为SVC调用代码，到这里我怀疑SVCALL针对有无RTOS有两种不同的设计，后来发现是多虑了。

# 检查中断向量表
实在不行上代码吧，在调试环境中看看，我运行了一个仅仅包含蓝牙协议栈的程序，运行起来后发现SCB->VTOR=0

很奇怪~

蓝牙协议栈在ROM中的地址范围为0-0x27000，我们编写的APP地址范围为0x27000-END。因为单片机从地址0处开始运行，肯定是运行蓝牙协议栈，在协议栈内跳转到APP中，按照一般BOOT+APP的设计思想，BOOT跳转是应把中断向量表寄存器改为APP的中断向量表。这里为什么还是BOOT的中断向量表呢？难道是Bug。看来需要看看协议栈固件中发生了什么。

其实到这里我就大概猜到这里面的门道了，我准备逆向分析蓝牙协议栈的二进制包，看看是否符合我的预期。我认为整个系统任然依赖Boot(蓝牙协议栈程序，我把它当作boot的角色)的向量表，而在Boot的中断处理函数中通过适当的判断决定执行Boot中断处理逻辑还是跳转引导到APP中的中断处理函数中来。

# 分析SoftDevice固件
官方提供的boot为一个HEX文件，我先转为bin文件，方便查看二进制数据，从前面的调试结果能够看到中断向量表在起始位置，那么很容易就找到SVC的入口地址了，在0x2c偏移处得到0x00000AA5，忽略最低bit1，转到0x00000AA4，开始分析SVC的函数处理流程，这里不讨论如何逆向固件，直接看结果，我把其中最关键的部分截取了出来。

这里分析的固件从nRF5_SDK_17.0.2_d674dde中获得。不同版本的SoftDevice可以在汇编数据上有差别，大体流程应该是一致的。

```asm
0x00000AA4 F01E0F04  TST           lr,#0x04       ;; 解析SVCid的经典代码，很多地方都看过
0x00000AA8 BF0C      ITE           EQ
0x00000AAA F3EF8108  MRS           r1,MSP
0x00000AAE F3EF8109  MRS           r1,PSP
0x00000AB2 6988      LDR           r0,[r1,#0x18]  ;; 得到执行SVC指令后的PC指针值. why offset = 0x18, see ARM manual
0x00000AB4 3802      SUBS          r0,r0,#0x02    ;; 向后回退一个指令的长度，即2字节，就是SVC指令的位置
0x00000AB6 7800      LDRB          r0,[r0,#0x00]  ;; 按字节取值，从低位得到SVC的id，id是作为低位包含在二进制编码中的
0x00000AB8 2818      CMP           r0,#0x18
0x00000ABA D103      BNE           0x00000AC4   ;; SVC的ID不等于0x18跳转到0x00000AC4，这就是我们需要流程，FreeRTOS的SVCid为0
0x00000ABC E000      B             0x00000AC0   ;; SVC的ID等于0x18跳转到0x00000AC4
0x00000ABE 0000      MOVS          r0,r0
0x00000AC0 4A07      LDR           r2,[pc,#28]  ; @0x00000AE0
0x00000AC2 4710      BX            r2
0x00000AC4 4A07      LDR           r2,[pc,#28]  ; @0x00000AE4
0x00000AC6 6812      LDR           r2,[r2,#0x00] ;; 检测内存发现@0x20000000的值为0x1000
0x00000AC8 322C      ADDS          r2,r2,#0x2C   ;; 0x2C? 像是SVC的偏移，推测在RAM的0x1000处有一个新的中断向量表
0x00000ACA 6812      LDR           r2,[r2,#0x00] ;; r2=0x0002606D
0x00000ACC 4710      BX            r2
0x00000ACE 0000      MOVS          r0,r0

0x00000AE0: 00000377 .word 0x00000377
0x00000AE4: 20000000 .word 0x20000000

0x0002606C 2004      MOVS          r0,#0x04     ;; 这是另一个SVC处理函数
0x0002606E 4671      MOV           r1,lr
0x00026070 4208      TST           r0,r1        ;; 这是编译器生成的还是手工编写的汇编，太冗余了...
0x00026072 D002      BEQ           0x0002607A
0x00026074 F3EF8109  MRS           r1,PSP
0x00026078 E001      B             0x0002607E
0x0002607A F3EF8108  MRS           r1,MSP
0x0002607E 6988      LDR           r0,[r1,#0x18]
0x00026080 3802      SUBS          r0,r0,#0x02
0x00026082 7800      LDRB          r0,[r0,#0x00] ;; 到这里，函数重新解析了一遍SVC的ID
0x00026084 2810      CMP           r0,#0x10
0x00026086 DB13      BLT           0x000260B0    ;; 如果ID小于0x10则跳转到 0x000260B0
0x00026088 2820      CMP           r0,#0x20
0x0002608A DB0F      BLT           0x000260AC    ;; 如果ID小于0x20则跳转到 0x000260AC
0x0002608C 282C      CMP           r0,#0x2C
0x0002608E DB0B      BLT           0x000260A8    ;; 如果ID小于0x2c则跳转到 0x000260A8
0x00026090 4A0A      LDR           r2,[pc,#40]  ; @0x000260BC
0x00026092 6812      LDR           r2,[r2,#0x00]
0x00026094 4B0A      LDR           r3,[pc,#40]  ; @0x000260C0
0x00026096 429A      CMP           r2,r3
0x00026098 D103      BNE           0x000260A2
0x0002609A 2860      CMP           r0,#0x60
0x0002609C DB04      BLT           0x000260A8
0x0002609E 4A09      LDR           r2,[pc,#36]  ; @0x000260C4
0x000260A0 4710      BX            r2
0x000260A2 2002      MOVS          r0,#0x02
0x000260A4 6008      STR           r0,[r1,#0x00]
0x000260A6 4770      BX            lr
0x000260A8 4A07      LDR           r2,[pc,#28]  ; @0x000260C8
0x000260AA 4710      BX            r2
0x000260AC 4A07      LDR           r2,[pc,#28]  ; @0x000260CC
0x000260AE 4710      BX            r2
0x000260B0 4A07      LDR           r2,[pc,#28]  ; @0x000260D0
0x000260B2 6812      LDR           r2,[r2,#0x00]  ;; 在0x20000004处的值为0x27000,它是app的起始位置
0x000260B4 322C      ADDS          r2,r2,#0x2C    ;; 计算出app中SVC的地址
0x000260B6 6812      LDR           r2,[r2,#0x00]
0x000260B8 4710      BX            r2             ;; 跳转到app中的SVC执行

0x000260BC: 2000005C .word 0x2000005C
0x000260C0: CAFEBABE .word 0xCAFEBABE
0x000260C4: 0000139B .word 0x0000139B
0x000260C8: 00024485 .word 0x00024485
0x000260CC: 00024E9F .word 0x00024E9F
0x000260D0: 20000004 .word 0x20000004
```

分析执行过程猜测在0x20000000和0x20000004处存放了两个中断向量表的起始地址，一个是内存中的向量表，位于0x1000；另一个就是APP的中断向量表，位于0x27000。

# 结论
非常明显了，当SVC的id小于0x10时就会跳转到我们编写的APP固件中的SVC处理函数中，这样就实现了FreeRTOS的port所依赖的功能。而id大于等于0x10时就执行协议栈内部的工作流程，实现了蓝牙协议栈+FreeRTOS功能的兼容。

从这个汇编代码的执行路径来看，从触发BOOT中的SVC到跳转到APP中的SVC处理函数，全程都没有使用栈操作的指令，没有涉及pop、push、bl、bxl等。这样在APP的中断函数中就能直接返回到线程环境代码，这样的设计对APP的中断处理函数来说也更加封闭。

官方的[文档](https://infocenter.nordicsemi.com/topic/sds_s132/SDS/s1xx/sd_resource_reqs/svc_number_range.html)中也说明了0-0xf的id调用分配到了Application，其它的id分配为协议栈。

类似的像PendSV_Handler中断处理函数，在协议栈内部触发后，经过5条指令就能够跳转到APP内部，还算是比较简洁的，应该不会对FreeRTOS的调度性能产生很大的影响。

官方的SDK中id是从0x60开始使用的，而协议栈中对小于0x20、0x2c等区间的id有特殊处理，为什么需要这样处理呢？以后再看吧。

不懂固件中为什么需要两次解析SVC的id，关键是是第二次解析时很不简洁，还多了3条指令，看起来好别扭-_-!
