---
title: 更优雅的使用Protothreads协程框架
author: WangGaojie
---
## PT协程框架
PT([Protothreads](https://github.com/gburd/pt))是一个轻量级的多任务框架，区别于一般基于栈的RTOS，它实现多任务的基本原理是通过语句间的任意跳转来实现任务切换。所以实现多任务的开销比较小。

我们看一段示例代码来了解其用法。
```c
/**
 * This is a very small example that shows how to use
 * protothreads. The program consists of two protothreads that wait
 * for each other to toggle a variable.
 */

/* We must always include pt.h in our protothreads code. */
#include "pt.h"

#include <stdio.h> /* For printf(). */

/* Two flags that the two protothread functions use. */
static int protothread1_flag, protothread2_flag;

/**
 * The first protothread function. A protothread function must always
 * return an integer, but must never explicitly return - returning is
 * performed inside the protothread statements.
 *
 * The protothread function is driven by the main loop further down in
 * the code.
 */
static int
protothread1(struct pt *pt)
{
  /* A protothread function must begin with PT_BEGIN() which takes a
     pointer to a struct pt. */
  PT_BEGIN(pt);

  /* We loop forever here. */
  while(1) {
    /* Wait until the other protothread has set its flag. */
    PT_WAIT_UNTIL(pt, protothread2_flag != 0);
    printf("Protothread 1 running\n");

    /* We then reset the other protothread's flag, and set our own
       flag so that the other protothread can run. */
    protothread2_flag = 0;
    protothread1_flag = 1;

    /* And we loop. */
  }

  /* All protothread functions must end with PT_END() which takes a
     pointer to a struct pt. */
  PT_END(pt);
}

/**
 * The second protothread function. This is almost the same as the
 * first one.
 */
static int
protothread2(struct pt *pt)
{
  PT_BEGIN(pt);

  while(1) {
    /* Let the other protothread run. */
    protothread2_flag = 1;

    /* Wait until the other protothread has set its flag. */
    PT_WAIT_UNTIL(pt, protothread1_flag != 0);
    printf("Protothread 2 running\n");
    
    /* We then reset the other protothread's flag. */
    protothread1_flag = 0;

    /* And we loop. */
  }
  PT_END(pt);
}

/**
 * Finally, we have the main loop. Here is where the protothreads are
 * initialized and scheduled. First, however, we define the
 * protothread state variables pt1 and pt2, which hold the state of
 * the two protothreads.
 */
static struct pt pt1, pt2;
int
main(void)
{
  /* Initialize the protothread state variables with PT_INIT(). */
  PT_INIT(&pt1);
  PT_INIT(&pt2);
  
  /*
   * Then we schedule the two protothreads by repeatedly calling their
   * protothread functions and passing a pointer to the protothread
   * state variables as arguments.
   */
  while(1) {
    protothread1(&pt1);
    protothread2(&pt2);
  }
}
```
一个任务就是一个单独的函数和一个记录任务状态的句柄。通过在一个循环中一直执行所有任务相关的函数就能够实现多任务的效果，因为它是非抢占式的，如果某个任务函数如果不需要执行的时机就退出该函数并将执行时机转移到下一个任务。由于句柄记录了函数内部执行的位置，所以下次进入函数能够恢复任务的执行。
相比于传统的RTOS，PT协程的用法不是足够的灵活，任务不能动态的创建，任务的调度是提前分配好的，如果需要实时的创建或者删除一个任务，上面的这种写法就不是很方便。

## 基于PT实现一套更优雅的任务框架
主要是要解决PT协程框架不能动态创建任务的问题，可以使用链表将所有的任务管理起来，将创建好的任务就放入到链表中，如果任务结束或删除就将其从链表中删除。实现一个调度器自动调度链表中的所有任务。为了完全保留原始PT的用法，这里只新增任务创建和调度的实现，其余用法保持不变。

这里链表的引用参考Linux内核链表，其实现和使用都很简单。

首先需要定义一个全局的链表，用于管理所有的任务。
```c
// pt_os.c
#include "list.h"

LIST_HEAD(pt_pool);
```

剩余的内容就只需要在头文件中实现即可。
```c
// pt_os.h
#ifndef _PT_OS_H
#define _PT_OS_H
#include "pt.h"
#include "list.h"

typedef struct
{
    struct list_node list;
    PT_THREAD((*task)(struct pt *pt));
    struct pt pt;
} pt_item_t;

extern struct list_node pt_pool;

/**
 * Run pt os.
 * OS_SCHEDULE() executes an infinite loop.
 * 
 * Example usage:
 * @code{c}
 * int main(void){
 *     //...
 *     for(;;){
 *         OS_SCHEDULE();
 *     }
 *     return 0;
 * }
 * @endcode
 */
#define OS_SCHEDULE()                                           \
    do                                                          \
    {                                                           \
        pt_item_t *pt_item;                                     \
        list_for_each_entry(&pt_pool, pt_item, pt_item_t, list) \
        {                                                       \
            if (pt_item->task(&(pt_item->pt)) >= PT_EXITED)     \
            {                                                   \
                list_delete(&(pt_item->list));                  \
            }                                                   \
        }                                                       \
    } while (0)

/** 
 * Create a new pt task.
 * 
 * Example usage:
 * @code{c}
 * PT_THREAD(iwdg_task(struct pt *pt)){
 *      static struct timer periodic_timer;
 *      PT_BEGIN(pt);
 *      timer_set(&periodic_timer, 100);
 *      while(1){
 *          HAL_IWDG_Refresh(&hiwdg);
 *          PT_WAIT_UNTIL(pt, timer_expired(&periodic_timer));
 *          timer_reset(&periodic_timer);
 *      }
 *      PT_END(pt);
 * }
 * 
 * int main(void){
 *     OS_TASK_RUN(iwdg_task);
 *     //...
 *     for(;;){
 *         OS_SCHEDULE();
 *     }
 *     return 0;
 * }
 * @endcode
 */
#define OS_TASK_RUN(func)                            \
    do                                               \
    {                                                \
        static pt_item_t pt_##_func;                 \
        pt_##_func.task = func;                      \
        PT_INIT(&(pt_##_func.pt));                   \
        list_add_head(&pt_pool, &(pt_##_func.list)); \
    } while (0)

#endif
```

这里定义任务结构pt_item_t，相比与原生PT，每个任务会多占用12字节的RAM，此外就没有多余的开销了。使用OS_TASK_RUN()方法可以创建一个新的任务，它可以将PT任务添加到任务列表中。OS_SCHEDULE()方法用于调度所有的PT任务，它需要放入到一个循环中一直执行，当某个任务执行退出后可以将它从任务列表中删除。

这里存在一些缺陷，任务的入口不能传递参数，如果一定要实现这个功能那也是很容易的，对pt_item_t中的任务签名做调整即可。一个PT任务函数不能同时创建两个与之相关联的任务，这主要是OS_TASK_RUN()的实现导致的，我想大部分PT任务函数都不具有重入的特性，所以看起来也无伤大雅。

通过这段短小的代码(大约20行)就拓展了原生PT的实现，达到任务的自动调度，任务的动态创建和删除，这样在资源受限的单片机内能够实现更加灵活的异步编程。
