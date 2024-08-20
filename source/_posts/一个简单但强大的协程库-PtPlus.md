---
title: 一个简单但强大的协程库-PtPlus
author: WangGaojie
---

去年我基于Protothreads协程库实现了一些有意思的扩展功能以方便使用该协程库。相关的两篇文章可以翻看去年的博客。
- {% post_link '更优雅的使用Protothreads协程框架' %}
- {% post_link 'PT协程的一个小扩展' %}

最近我继续完善了相关的功能，并增加了一些新的功能。实现了低功耗支持、可超时的信号量等。

将所有相关代码进行整理后我为其命名为[PtPlus](https://github.com/DazzlingOkami/PtPlus)。现已经发布在GitHub上，欢迎大家使用。

它使用标准C语言编写，几乎可以运行在所有的嵌入式平台上。

相关的用法以及示例可以参考[PtPlus](https://github.com/DazzlingOkami/PtPlus)。

这里仅展示一段代码来展示该协程库的使用。
```c
#include <stdio.h>
#include "pt_plus.h"
#ifdef _WIN32
#include <windows.h>
#define sleep(ms) Sleep(ms)
#endif

struct pt_sem sem;

PT_THREAD(test_task_A(struct pt *pt)){
    PT_BEGIN(pt);
    static int i;
    for(i = 0; i < 10; i++){
        PT_TASK_DELAY(pt, 1000);
        printf("send semaphore - %d\r\n", clock_time());
        PT_SEM_SIGNAL(pt, &sem);
    }
    PT_END(pt);
}

PT_THREAD(test_task_B(struct pt *pt)){
    PT_BEGIN(pt);
    static int err_cnt = 0;
    while(1){
        int ret;
        PT_SEM_WAIT_TIMEOUT(pt, &sem, 2000, &ret);
        if(ret == 0){
            printf("Obtained semaphore! - %d\r\n", clock_time());
        }else{
            printf("Obtain semaphore timeout - %d\r\n", clock_time());
            err_cnt++;
            if(err_cnt >= 3){
                PT_EXIT(pt);
            }
        }
    }
    PT_END(pt);
}

int main(void){
    PT_SEM_INIT(&sem, 0);
    PT_TASK_RUN(test_task_A);
    PT_TASK_RUN(test_task_B);

    while(PT_TASK_NUMS() > 0){
        PT_TASK_SCHEDULE();
        clock_time_t idle_time = PT_TASK_IDLE_TIME();
        if(idle_time > 0){
            // Calling system delay functions
            sleep(idle_time);
        }
    }
    return 0;
}
```

在main函数中创建了两个任务后开始进行调度。任务A每间隔1秒发出一个信号量，任务B每隔2秒等待信号量，如果等待超时则打印错误信息。任务B连续3次超时后，任务B将退出。当所有的协程任务都结束后，程序结束。
