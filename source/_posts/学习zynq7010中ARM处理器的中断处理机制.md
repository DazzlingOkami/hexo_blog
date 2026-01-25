---
title: 学习zynq7010中ARM处理器的中断处理机制
author: WangGaojie
---

最近在使用zynq7010的PS模块时，需要通过PS端的ARM处理器读写SD卡上的文件，实际测试发现SDK中的SD卡底层驱动实现为轮询模式，导致数据读写时会长时间占用CPU。因为PS端运行了SDK中经过适配的FreeRTOS系统，所以我觉得需要对SD卡驱动进行适当的修改，通过中断配合信号量机制来避免处理器阻塞的问题。

通过xilinx SDK中的一项外设用法示例学习中断的用法，大概知道了在裸机环境下的使用中断的大概流程。

通过Xil_ExceptionRegisterHandler注册全局异常处理函数，我们将常用的外设中断会集中到一个异常上(XIL_EXCEPTION_ID_IRQ_INT)。

全局中断管理由xscugic模块负载，所以一般需要Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,XScuGic_InterruptHandler,IntcInstancePtr); ，这样外设的中断信号才能到达xscugic模块。

在xscugic模块中再通过XScuGic_Connect绑定sd外设的中断信号，并使用XScuGic_Enable控制每个外设的中断使能。

首先裸机模式下的异常中断向量表定义在asm_vectors.S文件中，中断入口为IRQHandler，在该函数中执行由Xil_ExceptionRegisterHandler注册的实际中断处理函数。在FreeeRTOS环境中，情况会变得不一样。

在FreeRTOS配置下，整个向量表被重定义了，在port_asm_vectors.S文件中可以看到新的向量表：

```asm
.section .vectors
_vector_table:
_freertos_vector_table:
	B	  _boot
	B	  FreeRTOS_Undefined
	ldr   pc, _swi
	B	  FreeRTOS_PrefetchAbortHandler
	B	  FreeRTOS_DataAbortHandler
	NOP	  /* Placeholder for address exception vector*/
	LDR   PC, _irq
	B	  FreeRTOS_FIQHandler

_irq:   .word FreeRTOS_IRQ_Handler
_swi:   .word FreeRTOS_SWI_Handler
```

实际的中断处理函数为FreeRTOS_IRQ_Handler，但是该函数任然没有直接调用到外设中断处理上，在FreeRTOS_IRQ_Handler函数中进行了最关键的中断嵌套计数(ulPortInterruptNesting)。通过进一步执行vApplicationIRQHandler函数来执行外设中断，这里在vApplicationIRQHandler函数中直接获取了xscugic模块的中断向量表。至此，完整的中断调用流程分析完毕。

相比于裸机环境，在FreeRTOS下，可以绕过xil_exception模块的异常管理，直接使用FreeRTOS的xPortInstallInterruptHandler或原始的XScuGic_Connect来绑定外设中断。

相比于Cortex-M单片机来说，显然该处理器的外设中断响应要明显慢于Cortex-M的中断机制，CM单片机基本上每个外设在中断向量表上都有独立的位置，能够快速触发中断响应，而Cortex-A9处理器的外设中断需要层层分发，实时性看起来要弱一些。
