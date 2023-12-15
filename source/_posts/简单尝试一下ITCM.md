---
title: 简单尝试一下ITCM
author: WangGaojie
---

## ITCM简介
在STM32高性能单片机中存在一种供内核专用的内存TCM-SRAM，称为紧耦合内存。为了区分不同的用途，又存在称为DTCM和ITCM的两种不同内存，这两种内存本质上是一致的，但是其背后默认的MPU策略不同，所以一般将DTCM用于存储关键的堆栈数据，将ITCM用于存储关键代码的指令。

由于TCM内存拥有独立的总线直连Cortex内核，它不经过AXI总线矩阵，所以理论上它应该要块很多才对。那TCM相比于一般的内存到底快多少呢？这里做一些简单的测试来说明一下。

## ITCM运行测试
构造一个简单的函数作为测试的对象：
```c
void runtest(void){
  for(int i = 0; i < 2000 * 1000; i++){
    __NOP();
    __NOP();
    __NOP();
    __NOP();
  }
}
```

测试的芯片选用STM32H7B0芯片，其主频280MHz，理论上主频越高，ITCM的优势越明显。

这里还需要说明一下D-Cache，指令缓存也能显著提高代码的运行速度，所以这里设计了4中不同的测试场景。通过统计被测函数的执行时钟周期来反应代码的执行速度。

*注：测试时未开启编译优化，调用函数在Flash中，运行栈为AXI-SRAM*

- 1.关闭D-Cache，被测函数在Flash中执行
运行周期：26065054
- 2.关闭D-Cache，被测函数在ITCM中执行
运行周期：15088644
- 3.开启D-Cache，被测函数在Flash中执行
运行周期：16019317
- 4.开启D-Cache，被测函数在ITCM中执行
运行周期：14017011

通过以上的数据不难看出ITCM的优势。在ITCM中运行的指令比在Flash中开启了D-Cache的指令运行还要块。测试的第4中情况相比于第1种情况性能更是提升了45%以上。

## 如何使用ITCM
这里简单介绍在gcc开发平台下如何方便快捷的使用ITCM。

首先需要在链接脚本中为函数分配独立的字段，用于将相关函数链接到ITCM中。
```
  /* Initialized ramfunc sections goes into RAM, load LMA copy after code */
  _siramfunc = LOADADDR(.ramfunc);
  .ramfunc :
  {
    . = ALIGN(4);
    _sramfunc = .;     /* create a global symbol at data start */
    *(.RamFunc)        /* .RamFunc sections */
    *(.RamFunc*)       /* .RamFunc* sections */

    . = ALIGN(4);
    _eramfunc = .;     /* define a global symbol at data end */
  } >ITCMRAM AT> FLASH
```
这里需要注意，RamFunc可能已经存在于data字段中，需要将其中data字段中删除。

这段链接脚本将函数的存储地址放到Flash中用于持久固化，将函数运行地址放到ITCM中用于执行。

之后需要代码将Flash中的代码数据搬运到ITCM中，这部分工作最好在汇编代码中完成。
```asm

/* 这部分放到最前面 */
.word _siramfunc
.word _sramfunc
.word _eramfunc

/* 这部分插入到启动汇编代码中 */
/* Copy RamFunc segment from flash to RAM */
  ldr r0, =_sramfunc
  ldr r1, =_eramfunc
  ldr r2, =_siramfunc
  b LoopCopyRamFunc

CopyRamFunc:
  ldr r3, [r2], #4
  str r3, [r0], #4
LoopCopyRamFunc:
  cmp r0, r1
  bcc CopyRamFunc

```
如果认为汇编较复杂，也可以使用C代码替代，作用是一样的：
```c
extern uint8_t _siramfunc;
extern uint8_t _sramfunc;
extern uint8_t _eramfunc;
memcpy(&_sramfunc, &_siramfunc, (uint32_t)(&_eramfunc - &_sramfunc));
```

这样当声明函数时就可以将函数存放到ITCM中，并实现持久化的运行：
```c
void runtest(void) __attribute__((section(".RamFunc")));
void runtest(void){
  for(int i = 0; i < 2000 * 1000; i++){
    __NOP();
    __NOP();
    __NOP();
    __NOP();
  }
}
```

其它的开发平台也是类似的原理。

## ITCM的一些限制
ITCM用起来也不是完美的，当涉及较多的函数间调用时，如果指令地址频繁在Flash、ITCM直接来回切换，会产生许多不必要的开销，因为它们两个区域的地址跨度太大，无法直接跳转，编译器会产生一些虚拟函数来实现这个跳转的过渡，无论是Flash到ITCM，还是ITCM到Flash都会有这样的操作。

ITCM主要是用于执行一些时间敏感的函数，例如中断函数，使得中断能够更快响应。还有就是例如FFT这种纯粹的计算型函数，它不会发生较多的其它函数调用且函数执行时间较长，这也能发挥ITCM的关键优势。
