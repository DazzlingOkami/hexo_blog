---
title: 在RTOS中如何优雅的处理Fault异常
---

### ARM处理器下的Fault异常
ARM处理器当发生异常事件后就会暂停当前程序的运行，处理器进入异常模式，响应一个来自处理器内核或者外设的中断请求。

这里想要处理的Fault异常就是ARM处理器多个异常中的一类，以Cortex-M7内核的ARM处理器为例，主要的Fault异常有HardFault、BusFault、UsageFault、MemManage。当发生这些异常就表示程序出现了比较严重的错误，进而导致“死机”。

### 嵌入式实时操作系统下发生Fault异常后会如何处理
常见的各种嵌入式实时操作系统(FreeRTOS、UCOSII/III等)都没有特别的对Fault进行处理，基本上都是按照默认处理方式来解决，也就是让处理器进入死循环。
```c
void xxxFault_Handler(void)
{
    while (1);
}
```

这种处理方式是无可厚非的，因为程序本身发生了致命的故障，暂停当前程序的运行或许能够避免程序发生更严重的错误。大多数单线程嵌入式软件(NoOS)都是这样处理的，一个地方的错误会导致整个系统崩溃。

但是在多任务环境下，RTOS提供了多线程运行的机制，线程之间是相对独立的运行。基于此，在RTOS中采用这种暂停处理器的方式来处理异常就不是最佳的解决方案了。例如有两个线程在运行(线程A和线程B)，线程B在正常运行，线程A由于地址访问错误，导致触发HardFault异常，Fault异常会暂停整个处理器的运行，不但使发生错误的A线程停止了运行，线程B也被牵连导致运行停止。所以在RTOS中发生异常时想要将出现Fault异常而带来的损失降低到最小，也就是单独停止错误线程的运行而不结束整个系统的运行。

### 发生Fault异常后如何暂停异常线程
首先需要知道的是，异常并不总是由线程运行导致的，还有用户中断处理程序(IRQ)、RTOS内核调度等，在这些地方发生的异常目前看起来是比较难处理的，所以后面主要处理的就是在线程中发生的Fault异常。这里以FreeRTOS操作系统为例，对于其他的操作系统都能实现类似的解决方案，可以结合实际的处理器、操作系统平台进行修改移植，因为核心思想是一致的。
当发生Fault后不能简单的while(1)处理了，更优雅且安全的的实现要完成两步动作：
- 暂停错误线程的运行
- 将CPU的执行权限切换到正常线程中

#### 清理异常线程
这是在FreeRTOS下的一个实现：
```c
// fault_handle.c
#include "FreeRTOS.h"
#include "task.h"

#if (INCLUDE_xTaskGetIdleTaskHandle == 0)
#warning "Unable to switch to a valid task!"
#endif

void* clean_fault_task(void){
    extern void* pxCurrentTCB;
    TaskHandle_t fault_task = pxCurrentTCB;
    if(xTaskGetSchedulerState() == taskSCHEDULER_NOT_STARTED){
        /* RTOS not running. */
        return NULL;
    }
    /* Switch to Idle task and delete current fault task. */
    // log_info("task(%s) fault.\n", pcTaskGetName(task));
    #if defined(INCLUDE_xTaskGetIdleTaskHandle) && (INCLUDE_xTaskGetIdleTaskHandle == 1)
    pxCurrentTCB = xTaskGetIdleTaskHandle();
    #else
    return NULL;
    #endif
    vTaskDelete(fault_task);
    return pxCurrentTCB;
}
```
简单解释一下这段代码的含义。首先需要判断操作系统的任务调度有没有运行，如果操作系统还没有启动，这就说明异常不是在线程中触发的，这就不在解决范围内。然后找到一个可以切入的线程，而且它是处于就绪态的任务，显然最好的选择就是Idle线程了。通过`xTaskGetIdleTaskHandle()`获取到Idle线程的句柄，赋值给pxCurrentTCB。最后调用`vTaskDelete()`删除当前错误的线程，并返回新的当前线程就好了。

这里为什么需要先获取新的的线程再删除旧的错误线程呢？这就需要理解线程删除中发生了什么。
```c
// void vTaskDelete( TaskHandle_t xTaskToDelete )
if( pxTCB == pxCurrentTCB )
{
    /* A task is deleting itself.  This cannot complete within the
    * task itself, as a context switch to another task is required.
    * Place the task in the termination list.  The idle task will
    * check the termination list and free up any memory allocated by
    * the scheduler for the TCB and stack of the deleted task. */
    vListInsertEnd( &xTasksWaitingTermination, &( pxTCB->xStateListItem ) );

    /* Increment the ucTasksDeleted variable so the idle task knows
    * there is a task that has been deleted and that it should therefore
    * check the xTasksWaitingTermination list. */
    ++uxDeletedTasksWaitingCleanUp;

    /* Call the delete hook before portPRE_TASK_DELETE_HOOK() as
    * portPRE_TASK_DELETE_HOOK() does not return in the Win32 port. */
    traceTASK_DELETE( pxTCB );

    /* The pre-delete hook is primarily for the Windows simulator,
    * in which Windows specific clean up operations are performed,
    * after which it is not possible to yield away from this task -
    * hence xYieldPending is used to latch that a context switch is
    * required. */
    portPRE_TASK_DELETE_HOOK( pxTCB, &xYieldPending );
}
// ...
if( pxTCB == pxCurrentTCB )
{
    configASSERT( uxSchedulerSuspended == 0 );
    portYIELD_WITHIN_API();
}
// ...
```
这是FreeRTOS线程删除时是进行的一些操作，当判断为删除当前线程(pxCurrentTCB)是会进行的额外操作。FreeRTOS是不能直接删除线程自身的，它是将自己标记为预删除的状态，然后操作系统切换到Idle线程中时去清理这些需要被删除的线程，此外这里还会进行一次主动的线程调度，这是非常危险的，当线程出现错误后，如果继续进行常规调度流程，这可能会涉及到访问错误线程的栈空间，这是不可靠的(导致更严重的错误)。所以线程清理仅仅是让操作系统不再调度这个错误的线程，与原错误线程相关的操作降低到最少。

#### 进行一次特别的线程调度
有何特别？这里的线程调度操作将没有切出线程，只有切入线程，因为本该切入的线程在刚刚前面那段代码中删除了。所以这里的线程调度就不能直接使用系统的API完成调度，这里需要自己实现一段线程调度的代码。这里以一段ARM汇编来实现这段调度程序。
```arm
bic r3, lr, #7
cmn r3, #8
beq .end

bl clean_fault_task
cmp r0, #0
beq .end

ldr r0, [r0]
ldmia r0!, {r4-r11, r14}
tst r14, #0x10
it eq
vldmiaeq r0!, {s16-s31}

msr psp, r0
isb
mov r0, #0
msr basepri, r0
bx r14

.end: b .end
```
这段汇编代码分为4个部分，第一段，这里是通过LR寄存器的值来判断触发异常是否是在普通的线程中，如果不是在线程中触发的异常将不能处理，直接跳转到最后的死循环中。第二段，调用前面实现的任务清理函数，清理掉错误线程并选择一个新的可运行线程(这里就是Idle线程)，函数返回后，判断r0，如果无法清理错误线程或者没有可用的线程，仍然跳转到最后的死循环中。第三段，r0就是前面函数返回时传递过来的新的线程句柄，它存储的第一个字段就是该线程的栈，栈里面存储的内容结构涉及到FreeRTOS线程调度相关的内容。这里就简单说一说FreeRTOS的任务调度时的栈内结构，栈里面存储的是r4-r11寄存器，这些寄存器是ARM异常处理无法自动保存的。此外还有保存r14(lr)寄存器，它是用来判断栈内是否存储了浮点寄存器的，LR寄存器的bit4指示了线程是否在使用浮点寄存器。由于s0-s15以及浮点状态寄存器FPSR是直接由ARM异常自动完成存取了，所以这里还需要根据线程是否使用了浮点寄存器来存储s16-s31寄存器。

将待切入运行的线程相关的寄存器都恢复完毕后，就可以退出异常并转入到新的线程去执行了，也就是第四段代码的内容。

#### 封装异常处理
将前面汇编代码进行封装，这样就能够处理多个Fault异常了。
```c
// fault_handle.h
#ifndef _FAULT_HANDLE_H
#define _FAULT_HANDLE_H

#if defined(__CC_ARM)
#define FAULT_HANDLER()
#elif defined(__GNUC__)

#define FAULT_HANDLER() \
    ({ \
    __asm volatile ( \
        "   push {r0, r1, r2, r3, r12, lr}  \n" \
        /* Exception not returned to Handle mode */ \
        "   bic r3, lr, #7                  \n" \
        "   cmn r3, #8                      \n" \
        "   beq .end" END_LINE "            \n" \
        "                                   \n" \
        /* Clean fault task and get the pxCurrentTCB address. */ \
        "   bl clean_fault_task             \n" \
        "   cmp r0, #0                      \n" \
        "   beq .end" END_LINE "            \n" \
        "   add sp, #24                     \n" \
        "                                   \n" \
        /* The first item in pxCurrentTCB is the task top of stack. */ \
        "   ldr r0, [r0]                    \n" \
        /* Pop the registers that are not automatically saved on \
           exception entry and the critical nesting count. */ \
        "   ldmia r0!, {r4-r11, r14}        \n" \
        /* Is the task using the FPU context?  If so, pop the high vfp registers too. */ \
        "   tst r14, #0x10                  \n" \
        "   it eq                           \n" \
        "   vldmiaeq r0!, {s16-s31}         \n" \
        /* Restore the task stack pointer. */   \
        "   msr psp, r0                     \n" \
        "   isb                             \n" \
        "   mov r0, #0                      \n" \
        "   msr basepri, r0                 \n" \
        "   bx r14                          \n" \
        "                                   \n" \
        ".end" END_LINE ":                  \n" \
        "   pop {r0, r1, r2, r3, r12, lr}   \n" \
        ); \
    })

#define _STR2(x) #x
#define _STR(x) _STR2(x)
#define END_LINE _STR(__LINE__)

#endif

#endif
```
这段代码实现比前面的汇编多了一些内容，主要是要考虑堆栈平衡以及异常线程清理失败后的恢复线程的问题。前面处理失败是进入一个死循环中，这里没有进入死循环。处理失败后所有的寄存器会恢复到处理前的状态，这样能够为其它错误分析类的程序提供帮助，比如我前面有篇文章讲解了如何进行栈回溯，那段程序就能够衔接在这段程序前面或者后面，帮助分析异常问题。

这里需要特别提醒，栈平衡是非常重要的一点，尤其是使用汇编来操作栈空间时。要知道，`bx lr`后后面的代码是无法继续执行的，所以需要保证进入Fault时到执行`bx lr`时msp栈是平衡的。如果栈不平衡就会导致栈内存空间异常减小。
这段宏定义是内联汇编，当它嵌入到C函数后，函数头是否还有栈操作我不确定。可能不同的编译器情况不一样，但我用GCC时没有发现问题。如果你想要把这段代码移植到你的系统中，最好仔细核对栈平衡相关的内容。

相比于最开始的while(1)，新的异常处理程序可以写为：
```c
void xxxFault_Handler(void)
{
    FAULT_HANDLER()；
    while (1)
    {
    }
}
```
这样当一个线程执行出现异常后能够自动停止并继续进行系统调度，保证了系统的持续运行。

## 总结
通过这种处理方式能够解决线程错误带来的死机问题，但是在实际的生产运行过程中，还需要其它辅助性质的程序来帮助程序更加稳固的运行，例如可以设计一个基于定时器的守护任务，当它监视到线程崩溃结束后自动重启它。这样才更加具有实际意义。

这段程序任然有不完美的地方，比如它不能处理非线程Fault异常，还有线程清理时是暂停它还是删除它？删除线程后相关资源是否及时回收？这些问题都需要结合实际项目进一步完善。
