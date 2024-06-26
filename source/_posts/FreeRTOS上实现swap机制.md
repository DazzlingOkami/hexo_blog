---
title: FreeRTOS上实现swap机制
author: WangGaojie
---

## 为什么需要引入SWAP机制
在小容量SRAM的单片机上使用RTOS是一种很尴尬的事情，因为每个任务都需要分配独立的线程栈，栈空间少则512字节，复杂任务的栈空间则达到了几千字节的大小，这对于仅有8k字节SRAM的单片机来说显得压力太大了，两三个任务就导致内存不足。

使用协程方案在在小容量单片机上是一个不错的方案，但是协程相比于实时操作系统具有很多的缺点。比如任务实时性不足、在用法上不灵活等。

根据现代操作系统的SWAP机制，我在FreeRTOS操作系统上也实现了任务栈的swap机制，通过外部存储设备扩展单片机内的可用栈空间大小，可显著降低片内SRAM空间的消耗。同时该方案对任务来说是无感的，也就是说它和普通任务一样在用法上没有区别，可以按照传统的RTOS任务进行设计，这对于任务的移植来说能够降低复杂性。

## 方案介绍
使用SWAP机制一个重要前提是需要一个大容量的片外存储设备来进行内存交换。比如说片外的PSRAM，通过SPI总线访问的那种。一个基本的设想是先分配一段实际的物理空间，这将作为所有线程的栈空间，线程A在使用完这段空间后，切换到线程B时，先将线程A的栈空间数据存放到外部分配的SWAP空间上，同时将线程B保存在外部SWAP空间上的栈数据恢复到共享的栈空间上。通过预留在FreeRTOS内的HOOK接口很容易就能实现这些功能，且对FreeRTOS的原有代码没有明显的入侵。

## 具体实现的要点
- 1.任务创建
为了指定任务运行时实际的栈空间地址，使得栈空间数据可控，需要使用xTaskCreateStatic()接口创建任务。此外，创建任务时会对栈空间数据进行必要的初始化操作，所有创建任务时的上下文栈空间不能处于共享栈空间上，否则会出现数据冲突。
任务的删除也需要特别设计，任务删除仅仅需要删除分配的TCB块空间，栈空间不会收，但是需要释放外部分配的SWAP空间。

- 2.任务切换时栈空间存储
将当前任务的栈空间存储到SWAP空间时，没有必要将整个栈空间都进行存储，因为栈的使用是线性连续的，通常使用栈底的部分数据，所以仅需保存部分数据到SWAP空间即可，这样可以提升线程的切换速度。
在任务切换时还需要做适当的判断，避免无效的栈空间交换。我们仅仅需要在从一个共享栈任务切换到另一个共享栈任务时需要触发SWAP机制，其它情况都不需要进行内存交换。例如，Tiemr线程的栈时独立分配的，那么从timer切换到共享栈任务A在切换到timer过程中，显然都是不需要进行栈交换的。
可以使用traceTASK_SWITCHED_IN()作为内存交换点。

- 3.SWAP存储
作为栈的交换存储空间，读写速度是非常关键的，将一次栈切换的时间控制在1ms以内时非常有必要的。所有内存交换的机制应当尽量设计的简单可靠，避免使用文件等复杂的存储过程。
存储空间的大小和访问方式也很关键，最好是字符设备，块设备的读写在这里是低效的。存储空间应当能够覆盖所有共享的内存数据以及管理这些数据的额外数据开销。

## 方案测试
我选择了STM32L051C8T6单片机(8k字节SRAM)配合APS6404L片外的PSRAM进行测试，PSRAM使用16Mhz的SPI总线连接。创建了Timer任务、IDLE任务和一个普通的用户任务，这三个任务使用了独立的栈空间分配，分别为512、512、1024字节。此外还分配了3个栈空间共享的任务，栈空间大小均为1024字节。在这种情况下，所有任务运行起来后FreeRTOS的heap剩余空间仍然超过了1k。额外创建一个栈空间共享的任务开销约100字节左右。

## 缺陷
- 无法在栈共享任务上创建一个新的栈共享任务，这个在实现要点1上已经解释了。
- 任务栈内的局部变量数据无法传递到栈外，这是特别关键的。通常栈内的数据通过邮箱等传递到其它线程后，通过阻塞的方式等待响应，所以该栈内的局部变量数据传递到其它线程后依然有效。但是对于栈共享的任务来说是无法实现的。
- 栈共享的任务实时性无法得到保障，因为它在线程唤醒时需要进行栈空间的恢复操作，任务响应时间肯定比栈独立的任务要长。所有对于需要快速响应的任务任然可以使用一般的任务创建方式创建任务，因为从栈共享任务切换到一般任务是不需要触发SWAP机制的。
