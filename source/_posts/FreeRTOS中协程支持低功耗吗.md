---
title: FreeRTOS中协程支持低功耗吗?
author: WangGaojie
---

## FreeRTOS协程
FreeRTOS除了支持多线程外，还支持另外一种多任务机制-协程(coroutine)。它和线程不一样，每个协程不需要独立的运行空间，它依靠编程语法技巧在逻辑上实现了多任务的机制。它避免的一般RTOS带来的栈切换开销，仅仅依靠极小的开销就能保存任务的中间状态并在适当的时刻快速恢复任务。简单来说就是任务需要切换时先记录当前执行的位置并返回，重新进入该任务时根据保存的位置恢复任务执行，所以它不支持任务间抢占，所以任务的实时性也比较有限，但是它多任务时的低开销特性在许多场景下具有比较明显的优势。

FreeRTOS使用C语言提供的switch-case语法作为实现协程的关键，switch-case能够进行非常灵活的跳转，这个真的超乎一般人对switch用法的理解(点名达夫设备)。此外由于多个协程任务共享同一个栈空间，所以协程中无法使用生命周期比较长的局部变量，即定义的局部变量在使用的整个生命周期内不能进行协程的任务调度，一旦发生任务调度，则局部变量的数据就有可能丢失，所以在设计协程时通常使用全局变量来避免使用栈空间内的局部变量数据。

在设计协程任务时需要根据一定的代码模板来设计，因为协程的核心是由语法来实现的，所以代码无法设计的足够的灵活，存在许多的设计和用法限制。

## 低功耗特性
协程意味着任务过程简单，在现实应用场景下，简单任务和低功耗特性通常一起出现，那使用FreeRTOS的协程支持低功耗吗？根据FreeRTOS的参考文档来看，官方并没有提及协程+低功耗的用法，但是这两个特性正好是FreeRTOS都支持的，这两个特性能够独立正常工作。那合并到一起呢？

## 冲突
官方推荐的协程调度方案是这样的：
```c
void vApplicationIdleHook( void )
{
    for( ;; )
    {
        vCoRoutineSchedule();
    }
}
```
使用IDLE任务的hook来进行协程的调度。那进一步看看该hook在Idle任务中的执行位置：
```c
// ...
#if ( configUSE_IDLE_HOOK == 1 )
{
    extern void vApplicationIdleHook( void );

    /* Call the user defined function from within the idle task.  This
     * allows the application designer to add background functionality
     * without the overhead of a separate task.
     * NOTE: vApplicationIdleHook() MUST NOT, UNDER ANY CIRCUMSTANCES,
     * CALL A FUNCTION THAT MIGHT BLOCK. */
    vApplicationIdleHook();
}
#endif /* configUSE_IDLE_HOOK */

/* This conditional compilation should use inequality to 0, not equality
 * to 1.  This is to ensure portSUPPRESS_TICKS_AND_SLEEP() is called when
 * user defined low power mode  implementations require
 * configUSE_TICKLESS_IDLE to be set to a value other than 1. */
#if ( configUSE_TICKLESS_IDLE != 0 )
{
    TickType_t xExpectedIdleTime;

    /* It is not desirable to suspend then resume the scheduler on
     * each iteration of the idle task.  Therefore, a preliminary
     * test of the expected idle time is performed without the
     * scheduler suspended.  The result here is not necessarily
     * valid. */
    xExpectedIdleTime = prvGetExpectedIdleTime();

    // ...
}
#endif /* configUSE_TICKLESS_IDLE */
```
到这里应该发现问题了吧。vApplicationIdleHook比TICKLESS处理要先执行，而协程的调度实现是一个死循环，这意味着idle任务没有时机来进行TICKLESS处理，导致系统无法进入低功耗状态。

这里解释一下vCoRoutineSchedule()为什么需要放到一个无限循环中，这是该协程框架决定的，该调度函数需要一直执行，该函数并不返回一些调度状态，导致我们无法决定协程调度的时机，为了协程的正常运行，所以协程就必须要一直进行调度。而最终的结果就是使用协程后，FreeRTOS的低功耗特性就失效了。

## 曙光
分析到这里的时候，我突然有一个疑惑，FreeRTOS已经设计了非常完善的配置机制，为什么它没有考虑到这两个功能的用法冲突，进而在配置上进行处理呢。比如说开启协程后就禁止使用低功耗特性。这是不是意味着有什么方法能够突破这个限制，所以官方故意在配置上留下了同时启用两种功能的可能性。

我重新开始梳理了FreeRTOS的协程和低功耗实现方案，发现现有的设计是无法实现的。将协程调度放入到单独的线程中是最接近的一个实现方案：
```c
void CoRoutineTask(void *p){
    for(;;){
        vCoRoutineSchedule();
        // vTaskDelay(?);
    }
}
```
该任务的优先级如何确定呢？协程中的延时是否需要？延时多长？只要优先级比idle任务高且存在任意时间的延时，低功耗的tickless就可执行。但是难点是无法确定延时时间。

协程没有实现计算下一次协程调度的时机函数。对于任务来说，prvGetExpectedIdleTime()函数能够计算出下一次进行任务调度的期望时间。所以我们也需要一个这样的函数，计算下一次进行协程调度的期望时间。实现了这个函数将是在协程下实现低功耗的希望。

## 下一次协程调度到底需要多长时间
检索这些数据就能知道答案：
- 1.等待任务列表(pxDelayedCoRoutineList)
- 2.就绪任务列表(pxReadyCoRoutineLists)
- 3.即将就绪任务列表(xPendingReadyCoRoutineList)

最后一个是很容易忽视的，在进行协程间通信时，任务并不是直接切换到就绪任务列表中，而是添加到了待就绪任务列表(这是为了保证就绪任务列表不在中断中进行修改)。所以在检索任务时该列表中的任务可以看作时任务已经就绪了。

当就绪任务列表非空或者即将就绪任务列表非空意味着协程需要立即调度。当等待任务列表中的任务已经超时了也要立即调度。否则就根据等待任务列表中等待时间最少的任务来计算下一次协程调度的时间。此外，还需要注意所有列表为空的情况，这可以认为下一次调度的时间为无限长。

```c
TickType_t xGetExpectedIdleTime( void )
{
    CRCB_t * pxCRCB;
    UBaseType_t uxPriority;
    TickType_t xIdleTime;

    if( listLIST_IS_EMPTY( &xPendingReadyCoRoutineList ) == pdFALSE )
    {
        xIdleTime = 0u;
    }
    else
    {
        xIdleTime = portMAX_DELAY;

        for( uxPriority = 0; uxPriority < configMAX_CO_ROUTINE_PRIORITIES; uxPriority++ )
        {
            if( listLIST_IS_EMPTY( &( pxReadyCoRoutineLists[ uxPriority ] ) ) == pdFALSE )
            {
                xIdleTime = 0u;
                break;
            }
        }

        if(xIdleTime > 0u)
        {
            if( listLIST_IS_EMPTY( pxDelayedCoRoutineList ) == pdFALSE )
            {
                pxCRCB = ( CRCB_t * ) listGET_OWNER_OF_HEAD_ENTRY( pxDelayedCoRoutineList );

                if(xCoRoutineTickCount < listGET_LIST_ITEM_VALUE( &( pxCRCB->xGenericListItem )))
                {
                    xIdleTime = listGET_LIST_ITEM_VALUE( &( pxCRCB->xGenericListItem )) - xCoRoutineTickCount;
                }
                else
                {
                    xIdleTime = 0u;
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
    }

    return xIdleTime;
}
```

在代码风格上参考了FreeRTOS相类似的代码风格。这里检查了所有优先级的就绪任务列表(从高优先级到低优先级应该会更快)，可以考虑仅仅检查uxTopCoRoutineReadyPriority优先级的列表，这样在优先级较多的时候可以执行的更快。

## 协程运行在可休眠的线程中
能够计算协程的调度时机后，我们就能将协程放入到一个可休眠的线程中，而不是在阻塞的无限循环中执行。

```c
void vCoScheduleTask( void )
{
    // ...
    for( ;; )
    {
        vCoRoutineSchedule();
        uint32_t idle_time = xGetExpectedIdleTime();
        if(idle_time > 0)
        {
            vTaskDelay(idle_time);
        }
    }
}
```
该线程的优先级比Idle线程略高。这样在保证正常的协程调度的情况下，系统也能够正常的运行idle线程，保证tickless中的prvGetExpectedIdleTime()函数能够计算出合理的系统线程调度休眠时间，进而实现了协程和低功耗特性的共存。

## 最后
xGetExpectedIdleTime()函数需要直接实现在croutine.c和.h文件中，所以会对FreeRTOS代码源文件进行修改，但是这没有对现存的代码进行进行逻辑上的更改，所以这样的修改是可控的。

上面还有一点未提及，如果在中断中同协程进行通信，协程还能够响应吗？需要如何修改才能实现在中断中同协程通信且不影响现有的低功耗特性？其实只需要做简单的修改即可，欢迎讨论。
