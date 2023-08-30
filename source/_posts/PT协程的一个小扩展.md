---
title: PT协程的一个小扩展
author: WangGaojie
---

在之前的文章中介绍了使用链表来更加优雅的使用PT协程。我最近发现了当初实现的一个错误，现在已经修复了。

这里不是为了讨论之前的实现，而是介绍一个更有意思的东西。我在之前的实现基础上，实现了一个更加优雅的函数来利用协程，这应该是一种异步编程的技巧，或者说这是C语言的花活😂。

Show code!
```c
/**
 * Declare a pt thread in a simple way.
 * 
 * Example usage:
 * @code{c}
 * PT_THREAD_DECL(thread1, {
 *     while(1){
 *         printf("hello pt!\r\n");
 *         OS_TASK_DELAY(pt, 100);
 *     }
 * });
 * 
 * // OS_TASK_RUN(thread1);
 * @endcode
 */
#define PT_THREAD_DECL(name, body) \
    PT_THREAD(name(struct pt *pt)){PT_BEGIN(pt);PT_YIELD(pt);body;PT_END(pt);}

/** 
 * Asynchronous execution. Only used in threads, similar to OS_TASK_DELAY() function.
 * 
 * Example usage:
 * @code{c}
 * PT_THREAD_DECL(invok_test, {
 *     static int cnt;
 * 
 *     cnt = 0;
 * 
 *     PT_INVOK({
 *         static int i;
 *         for(i = 0; i < 10; i++){
 *             printf("async invok %d\r\n", i);
 *             cnt += i;
 *             OS_TASK_DELAY(pt, 1000);
 *         }
 *         // Automatically exit this asynchrony at end of PT_INVOK();
 *     });
 * 
 *     PT_INVOK({
 *         while(1){
 *             printf("hello invok, cnt = %d\r\n", cnt);
 *             OS_TASK_DELAY(pt, 300);
 *         }
 *     });
 * });
 * 
 * // OS_TASK_RUN(invok_test);
 * @endcode
 */
#define PT_INVOK(body)                             \
    do                                             \
    {                                              \
        static pt_item_t pt_invok;                 \
        static char pt_anchor;                     \
        pt_invok.task =                            \
            containerof(pt, pt_item_t, pt)->task;  \
        list_add_head(&pt_pool, &(pt_invok.list)); \
        pt_anchor = 0;                             \
        LC_SET(pt_invok.pt.lc);                    \
        if (pt_anchor)                             \
        {                                          \
            body;                                  \
            PT_EXIT(pt);                           \
        }                                          \
        pt_anchor = 1;                             \
    } while (0)
```

PT_INVOK可以在不阻塞当前函数执行流程的情况下，从原地开辟出一个并行的执行流，有意思的一点是多个INVOK执行流之间是可以共享上层局部变量的，这就实现了多个执行流之间的通信或同步。

这个宏扩展看起来相当的有用，它解决了编程中一个常见的问题，即在一个不能被阻塞的执行流程中执行一个耗时的操作。

PT协程的实现本身就是一个很花哨东西，这个宏定义成功的把C语言的宏用法上升到了一个新的高度。我怀疑某些编译器根本不能处理这段代码😅。

这里需要注意，PT_INVOK内的代码不能重入，或者说PT_INVOK内的代码还未执行完成时，不能再次在这里执行同一段PT_INVOK，这是代码的静态性导致的。如果有重入的风险，可以在外部添加一些重入保护的机制。
