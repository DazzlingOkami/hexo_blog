---
title: 统计FreeRTOS中各个线程的处理器利用率
---

> 大约是在一年前，我在工作中遇到了需要分析嵌入式系统性能的需求，需要查看系统在关键时间点上，部分任务是否存在执行时间过长导致系统实时性能降低的情况。我在基于FreeRTOS的系统中设计了一段非侵入式的代码，能够获取到各个线程实时的处理器利用率，进而分析系统是否设计合理。最近我在整理以前的代码时又看到当时设计的这个功能，我发现当时仅仅是满足了一个基本的功能需求，部分地方还设计的不是足够的合理，所以最近抽空想把这个功能整理完善，以备以后工作之需。

# 处理器利用率
如何理解处理器利用率？这里摘抄了Linux系统中对这个词的理解：
```
CPU Usage:
The task's share of the elapsed CPU time since the last screen update, expressed as a percentage of total CPU time.
```
翻译过来差不多就是指在一段时间长度T内，线程A的运行时间占用了t，则在这段T时间端内，线程A的处理器利用率为 t/T。如果了解操作系统运作的原理，再来理解处理器利用率是非常简单的。

# 统计每个线程的执行时间
FreeRTOS操作系统内核本身是支持统计任务执行时间的，但是它是统计从任务开始运行时计算的，这种统计方法无法分析实时的处理器利用率，而且长时间的记录会导致内部时间计数器溢出，所以需要重新设计统计线程执行时间的功能。

统计一个任务的执行时间长度需要在任务开始运行和结束运行的时间点上计时，这里的开始运行和结束运行是指任务对处理器的占用情况，而不是指任务真正的开始和结束。结合FreeRTOS内核可以利用内核提前设计好的Hook机制找到任务进入和退出的时机，也就是下面两个宏：
```c
#ifndef traceTASK_SWITCHED_IN

/* Called after a task has been selected to run.  pxCurrentTCB holds a pointer
 * to the task control block of the selected task. */
    #define traceTASK_SWITCHED_IN()
#endif

#ifndef traceTASK_SWITCHED_OUT

/* Called before a task has been selected to run.  pxCurrentTCB holds a pointer
 * to the task control block of the task being switched out. */
    #define traceTASK_SWITCHED_OUT()
#endif
```

这里的注释非常好理解，根据这两个Hook函数和pxCurrentTCB就能够掌握每个线程的切换情况。这里先说明一点，这个功能对FreeRTOS内核是非侵入的，不会到FreeRTOS内核代码做任何改动。

接下来就是需要设计一个高精度的定时器，以我的经验来看定时器精度至少要达到微秒级别以上才行，因为有时候线程的切入和切出非常的块，定时器精度不够的话就无法准确的计算执行时间。

这里我采用的高精度定时器是DWT内核调试模块的计时单元，它的运行频率和处理器一致，所以定时精度是足够的，而且32位的计数器也能够满足一定时间长度的计时需求，这个内核调试模块在ARM Cortex-M处理器中都存在。

这里是我设计的DWT计时功能代码：
```c
/* cpu tick timer config. */
#define TIME_SEC_TO_US (1000000u)
#define CPU_CLK_FREQ() (SystemCoreClock)

#if defined(DWT) && defined(CoreDebug)

static unsigned int fclk_pre_us;

static inline void cpu_ts_time_init(void)
{
    fclk_pre_us = CPU_CLK_FREQ() / TIME_SEC_TO_US;

    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CTRL        |= DWT_CTRL_CYCCNTENA_Msk;
    DWT->CYCCNT       = 0;
}

static inline unsigned int cpu_ts_timrd(void)
{
    //return us
    return (DWT->CYCCNT / fclk_pre_us);
}

static inline void cpu_ts_reset(void)
{
    DWT->CYCCNT = 0;
}
```
提供了计数器初始化、复位、记录时间的功能。

# 记录FreeRTOS各个任务的运行时间
建立一个长度为MAX_TASK_NUMS的数组task_runtime用于存储每个任务的执行时间，数组中的每一项对应一个任务，所以需要为每个任务分配一个唯一的ID，FreeRTOS内核为每个任务分配了独立的ID，但是这些ID是不受控制的，即ID的分配总是递增的，任务删除后不会回收。所以需要为每个任务重新分配ID并在任务删除后做回收的处理。
```c
#define TASK_ID(task) (((StaticTask_t*)(task))->uxDummy10[0])

/* 
 * bit0 used for other task(id >= 32)
 * other bit used for every task.
 */
static uint32_t id_pool = 1u; 

/* hook to task create. */
void alloc_task_id(void* task){
    if(!task)
        return ;
    if(id_pool == 0xffffffff){
        /* The task pool is full. */
        TASK_ID(task) = 0;
        return ;
    }
    uint32_t new_pool = id_pool | (id_pool + 1u);
    TASK_ID(task) = 31u - __CLZ(id_pool ^ new_pool);
    id_pool = new_pool;
}

/* hook to task delete. */
void release_task_id(void* task){
    if(!task)
        return ;
    uint8_t id = TASK_ID(task);
    if(id == 0){
        return ;
    }
    id_pool &= ~(1u << id);
}

```

分配ID的过程是足够简单的，不会对系统的性能产生太多额外的影响。同样的利用Hook机制，把它嵌入到FreeRTOS内核中。
```c
// FreeRTOSConfig.h

#define traceTASK_CREATE(pxTCB)  alloc_task_id(pxTCB)
#define traceTASK_DELETE(pxTCB)  release_task_id(pxTCB)
```

通过`task_runtime[(TASK_ID(pxCurrentTCB))]`就能记录和访问线程的运行时间。

接下来就是最关键的traceTASK_SWITCHED_IN()和traceTASK_SWITCHED_OUT()，它们看起来还是比较简单的，一个最基础的逻辑如下：
```c
static uint32_t switch_in_time = 0;
static uint32_t task_runtime[MAX_TASK_NUMS] = {0};

void task_switched_out(void){
    task_runtime[(TASK_ID(pxCurrentTCB))] += cpu_ts_timrd() - switch_in_time;
}

void task_switched_in(void){
    switch_in_time = cpu_ts_timrd();
}

```

到此就实现了记录任务的运行时间，但是这和FreeRTOS内部的实现是类似的，我们需要在适当的时候清空所有的任务运行时间并重置计数器，保证系统总是记录任务最新的处理器利用率。

# 记录任务实时的处理器利用率
如果只有一个缓存记录处理器的运行时间，就会存在一个这样的情况，当运行时间情况的时候，正好需要查看处理器的利用率，那此时就看不到准确的数据。所以设计两个缓存，一个用于记录当前的数据，另一个用于缓存上一次的数据并给予用户访问。

最终完整的设计实现如下：
```c
extern void* pxCurrentTCB;

#define STAT_PERIOD (3000000u)
#define TASK_ID(task) (((StaticTask_t*)(task))->uxDummy10[0])
#define CURRENT_TASK_ID (TASK_ID(pxCurrentTCB))
#define MAX_TASK_NUMS (32)

static uint32_t switch_in_time = 0;
static uint32_t task_runtime_buf1[MAX_TASK_NUMS] = {0};
static uint32_t task_runtime_buf2[MAX_TASK_NUMS] = {0};
static uint32_t task_total_runtime = 0;
static uint32_t *task_current_runtime = task_runtime_buf1;

void task_switched_out(void){
    task_current_runtime[CURRENT_TASK_ID] += cpu_ts_timrd() - switch_in_time;
}

void task_switched_in(void){
    switch_in_time = cpu_ts_timrd();
    if(switch_in_time == 0){
        cpu_ts_time_init();
        switch_in_time = cpu_ts_timrd();
    }
    if(switch_in_time > STAT_PERIOD){
        cpu_ts_reset();
        task_total_runtime = switch_in_time;
        switch_in_time = 0;
        if(task_current_runtime == task_runtime_buf1)
            task_current_runtime = task_runtime_buf2;
        else
            task_current_runtime = task_runtime_buf1;
        for(int i = 0; i < MAX_TASK_NUMS; i++) task_current_runtime[i] = 0;
    }
}

/* 
 * bit0 used for other task(id >= 32)
 * other bit used for every task.
 */
static uint32_t id_pool = 1u; 

/* hook to task create. */
void alloc_task_id(void* task){
    if(!task)
        return ;
    if(id_pool == 0xffffffff){
        /* The task pool is full. */
        TASK_ID(task) = 0;
        return ;
    }
    uint32_t new_pool = id_pool | (id_pool + 1u);
    TASK_ID(task) = 31u - __CLZ(id_pool ^ new_pool);
    id_pool = new_pool;
}

/* hook to task delete. */
void release_task_id(void* task){
    if(!task)
        return ;
    uint8_t id = TASK_ID(task);
    if(id == 0){
        return ;
    }
    id_pool &= ~(1u << id);
}

static int task_id_exist(int id){
    return (id_pool & (1 << id));
}

static const uint32_t *get_task_runtime(void){
    if(task_current_runtime == task_runtime_buf1)
        return task_runtime_buf2;
    else
        return task_runtime_buf1;
}

int show_cpu_usage(int argc, char **argv){
    char buf[1024] = {0};
    int buf_len = 0;

    // os_enter_critical();

    const uint32_t *tick_buf = get_task_runtime();
    float r = 0;
    uint32_t irq_runtime = task_total_runtime;

    for(int i = 0; i < MAX_TASK_NUMS; i++){
        if(!task_id_exist(i)){
            continue;
        }
        if(i == 0){
            if(tick_buf[i] > 0)
                buf_len += sprintf(buf + buf_len, ">32 --  ");
            else
                continue;
        }else{
            buf_len += sprintf(buf + buf_len, "%2d  --  ", i);
        }
        if(tick_buf[i] == 0){
            buf_len += sprintf(buf + buf_len, "0\n");
        }else{
            r = 100.0f * (float)tick_buf[i] / task_total_runtime;
            if(r < 0.01f)
                buf_len += sprintf(buf + buf_len, "<0.01%%\n");
            else
                buf_len += sprintf(buf + buf_len, "%.2f%%\n", r);
        }
        irq_runtime -= tick_buf[i];
    }
    r = 100.0f * (float)irq_runtime / task_total_runtime;
    buf_len += sprintf(buf + buf_len, "IS  --  %.2f%%\n", r);
    // os_exit_critical(0);

    puts(buf);

    return 0;
}
```
记录周期为3000ms，这个时间需要结合定时器的溢出时间、系统性能要求做调整，时间太短会影响系统调度性能，太长会导致处理器利用率统计的实时性降低，且计数器存在溢出的风险。

最后这里写了一段简单的代码来输出各个任务的CPU利用率，经过我的测试，它能够很好的工作。访问数据时建议进入到临界区处理，这样更加安全。在实际使用时，每个线程使用ID表示的，ID和任务名称的关系可以通过FreeRTOS内核函数vTaskList()得到，一一对应即可。

# 总结
内容我写的比较仓促，其中部分细节我没有太多的说明，但我认为这些都是比较好理解。关于任务ID的分配，还有些细节没有说明，我将超过31个任务后面的任务ID统一分配为0，它们的处理器利用率将一并计算，这是我的ID分配器决定的，它只能分配31个ID。

当时我实现这个功能时忽略了FreeRTOS内核对任务ID的分配策略，当时我直接引用系统分配的ID，如果不对任务进行删除操作，他还是能够可靠的工作。但是如果需要进行任务删除，那么就会出现一些奇怪的问题。我也是最近才发现这个问题。主要是当时对FreeRTOS内核内的一些细节还是不够了解。

关于中断的执行时间，我这里只是粗略的估算了一个irq_runtime ，它包含了大部分任务调度的时间，并不能够保证把所有的中断时间统计在内。

这里涉及到了高精度定时器、快速ID的分配、双缓冲机制、内核调度、非侵入式设计等内容，每个都能够拿出来细讲，限于时间和篇幅就点到为止。如果你有幸看到这篇文章，对其中的内容有疑问或者建议，欢迎一起沟通学习。
