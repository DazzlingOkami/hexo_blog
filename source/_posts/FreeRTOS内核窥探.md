---
title: FreeRTOS内核窥探
author: WangGaojie
---

> 在多任务环境下调试代码需要一定的代码调试经验，尤其是涉及到异步的任务流程时，单步执行可能无法跟踪到需要的执行流程。如果代码全速运行时虽然可以通过日志来梳理执行流程，但是涉及到FreeRTOS内核中的内容时，一般的日志显得比较鸡肋了。比如说我想检查某个时刻所有任务的运行状态，包括任务是运行还是阻塞、阻塞的延时有多久、阻塞在某个信号量上等等，或者更进一步检查任务的调用栈等等。如果能够实现这些功能来窥视FreeRTOS的运行过程，这显然能够帮助我们更加清晰的了解代码的运行过程，方便解决一些棘手的问题。

## 获取当前系统所有任务的详细信息
FreeRTOS提供了uxTaskGetSystemState()方法，可以获取所有任务的一些简单信息，主要包括任务的TCB句柄、任务名称、运行状态、优先级、栈起始地址已经栈空间的最大使用量。这些信息是FreeRTOS直接暴露出来的信息，但是这些信息并不能够方便我们了解任务的运行过程，尤其是任务的细节状态。

我们想要获取的关键详细信息包括任务当前的SP指针，即实时的栈地址。还需要获取任务的阻塞时长，主要是对于处于阻塞状态的任务来说的。最关键的是需要获取任务阻塞的事件是什么，一般在RTOS中会使用很多的信号量、邮箱等同步机制，如果系统假死后能够看到每个任务的阻塞情况，那么可以最大程度的方便我们定位故障点。

非常遗憾的是以上这些信息都不能显式的从某个内核接口中获取，同时FreeRTOS执行非常严格数据隐藏策略，外部的应用代码是无法直接访问FreeRTOS内核的数据以及数据结构。但是FreeRTOS为开发人员预留了一组虚拟数据结构的定义，该虚拟数据结构的形式同FreeRTOS中实际的数据结构是一致的，FreeRTOS的初衷是通过这些虚拟结构来计算内核数据结构的大小，但是我们在这里通过这些数据结构的索引就能够突破内核的数据隐藏策略来间接访问内核数据。

虚拟数据结构的定义都是形如“struct xSTATIC_xxx”的定义，都位于FeeRTOS.h文件中。获取任务的SP指针现在就很容易了，SP指针的值位于TCB数据结构的第一项，虽然不能直接使用TCB的数据结构，但是struct xSTATIC_TCB的结构同它是一致的，所以我们访问任务SP指针的方法是这样的：
```c
void * pvGetTaskMSP(TaskHandle_t xTask){
    return ((StaticTask_t*)(xTask))->pxDummy1;
}
```

## 任务需要阻塞多长的时间？
这个问题稍微复杂一点，首先这个问题仅仅针对处于阻塞状态的任务，尤其是有限时长阻塞状态的任务。看看vTaskDelay()中的实现，当一个任务需要阻塞时，就将其放入到等待任务列表中，通过函数prvAddCurrentTaskToDelayedList()实现的。首先将当前时刻和需要延时的时长进行计算得到唤醒的时刻，将该时间点记录到任务的xStateListItem变量内部，同时该函数将xStateListItem变量作为一个列表项插入到了pxDelayedTaskList链表的中，这是一个有序的插入过程，即根据任务唤醒时间在链表上进行排序，到这里都比较好理解。通过任务TCB的xStateListItem就能够得到任务延时的时间。但是这里最关键的一点来了，如果任务时无限时长阻塞呢？这种情况是将变得比较麻烦，内核的实现是将其放入到了挂起任务列表(xSuspendedTaskList)中了，我们无法直接访问到xSuspendedTaskList，怎么确定任务是否处于无限阻塞状态呢？

到这里我确实没有比较好的办法，我选择直接修改内核代码，将处于挂起任务的xStateListItem值设为最大值以标记该任务没有明确的唤醒时间，这里需要思考有没有更简单的方法。
```c
if( ( xTicksToWait == portMAX_DELAY ) && ( xCanBlockIndefinitely != pdFALSE ) )
{
    /* Add the task to the suspended task list instead of a delayed task
     * list to ensure it is not woken by a timing event.  It will block
     * indefinitely. */
    listINSERT_END( &xSuspendedTaskList, &( pxCurrentTCB->xStateListItem ) );

    listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), 0xffffffff );
}
```

## 任务等待的事件是什么？
不是所有在阻塞的任务都在等待事件，如果任务只是一般的延时，这就表明任务没有等待任何事件。任务的事件机制是依靠TCB中xEventListItem来实现的，比如说等一个任务等待一个队列(queue)，任务就将该TCB中的xEventListItem加入到该队列内部相关的一个链表上。FreeRTOS中的队列(queue)用来实现消息队列、邮箱、信号量等，queue内部有两个链表，一个等待接收链表、一个等待发送链表，这是FreeRTOS内核允许发送数据阻塞导致的，所以队列内部需要两个链表来维护任务的阻塞关系，但实际上这两个队列大多数情况是仅使用其中一个。

通过xEventListItem中的pxContainer可以反向定位到任务所关联的链表，但是问题来了，queue上有两个链表，我们怎么确定任务所属的链表到底是哪一个呢？因为不能确定的话就不能反向索引到queue的指针，如果假设任务所述链表是queue内的第一个链表，通过containerof方法就能够定位到queue的地址，但万一是另一个链表呢？

队列中预留了一个有意思的成员uxQueueNumber，它并没有实际的功能，但通过它我们就能够将队列打上特殊标记，进而检索出队列的地址，找到了任务等待的队列地址，这样就确定了任务到达在等待什么事件了。
```c
#define QUEUE_TRACK_FLAG (0xab123456)
void vQueueFlagSet(void* queue){
    vQueueSetQueueNumber((QueueHandle_t)queue, QUEUE_TRACK_FLAG);
}
void *pvGetTaskEvent(TaskHandle_t xTask, int *type){
    List_t * const pxList = ((StaticTask_t*)(xTask))->xDummy3[1].pvDummy3[3];
    if(pxList == NULL){
        *type = 0;  // no event
        return NULL;
    }
    StaticQueue_t *queue = containerof(pxList, StaticQueue_t, xDummy3[0]);
    if(uxQueueGetQueueNumber((QueueHandle_t)queue) == QUEUE_TRACK_FLAG){
        *type = 1;
        return queue; // wait tx
    }
    queue = containerof(pxList, StaticQueue_t, xDummy3[1]);
    if(uxQueueGetQueueNumber((QueueHandle_t)queue) == QUEUE_TRACK_FLAG){
        *type = 2;
        return queue; // wait rx
    }
    *type = 3; // unknow
    return NULL;
}
```
vQueueFlagSet()方法需要使用queue的创建HOOK进行调用，这样内核上创建的所有queue都将被打上特殊标记以方便我们跟踪。

## 分析队列事件
上面得到的仅仅是队列的地址，队列的详细信息其实也能够获取到，包括该队列的类型(一般队列、互斥量、计数信号量、二进制信号量)、等待该队列事件的其它所以任务列表以及该队列内部数据量、数据尺寸的信息等待。方法任然是通过内核上的虚拟数据结构来间接访问。

## 利用SP指针回溯调用栈
最前面我们获取到了任务的SP地址，这是任务在丢失CPU执行权限后由内核任务调度器更新的最新的SP地址，所以它不是简单的SP指针，因为栈内还有任务调度时保存的数据。

分析调用栈除了需要实际的sp指针外，还需要最近一次的PC指针、LR指针。内核调度器在最后时刻将R4-R11寄存器、R14寄存器存入了栈内，如果还有浮点数，这还保存了S16-S31这16个浮点数寄存器。

简单来说，可以简化为两个结构体
```c
typedef struct{
    uint32_t r4[8];
    uint32_t r14;
    uint32_t s16[16];
    uint32_t r0[4];
    uint32_t r12;
    uint32_t lr;
    uint32_t pc;
    uint32_t xPSR;
    uint32_t s0[16];
    uint32_t fpscr;
} rtos_cm7_msp_fp_t;

typedef struct{
    uint32_t r4[8];
    uint32_t r14;
    uint32_t r0[4];
    uint32_t r12;
    uint32_t lr;
    uint32_t pc;
    uint32_t xPSR;
} rtos_cm7_msp_t;
```

如果存在浮点数则使用rtos_cm7_msp_fp_t，如果不存在浮点数则使用rtos_cm7_msp_t，是否使用浮点数可以使用r14进行判断，这里的r14就是FreeRTOS进行到任务调度时的异常中断中的LR寄存器。
有了这个数据结构，再配合栈分析机制就能够非常方便的得到任务的调用栈了。

## 总结
通过以上的方法，我们就能够非常方便的分析出FreeRTOS运行时的细节信息了，配合日志系统或者shell功能在适当的时候检索出这些信息对分析实时运行过程比较方便。此外，UCOSII等操作系统也能实现类似的操作。
