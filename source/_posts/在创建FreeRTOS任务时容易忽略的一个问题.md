---
title: 在创建FreeRTOS任务时容易忽略的一个问题
author: WangGaojie
---

面向对象的编程思想经常出现在嵌入式软件开发过程中，通常将一组相关的业务逻辑及其相关联的数据封装为一个实例对象以方便数据和方法的集中管理，尤其是在该实例对象还涉及一个关联的任务时，为了将相关的数据传入线程，将所有的数据封装起来是一种非常合理的操作。下面是一段这样的代码：
```c
typedef struct {
    TaskHandle_t task;
    const char *str;
    int cnt;
} hello_obj_t;

void hello_task(void *pvParameters){
    hello_obj_t *obj = (hello_obj_t *)pvParameters;
    for(int i = 0; i < obj->cnt; i++){
        printf("%s\n", obj->str);
        vTaskDelay(1000);
    }
    TaskHandle_t task = obj->task;
    free(obj);
    vTaskDelete(task);
}

int main_test(const char *str, int cnt){
    hello_obj_t *obj;
    obj = malloc(sizeof(hello_obj_t));
    obj->str = str;
    obj->cnt = cnt;
    static StackType_t stack[1024];
    static StaticTask_t task;
    obj->task = xTaskCreateStatic(hello_task, "hello", 1024, obj, 10, stack, &task);
    return 0;
}
```

这段代码展示一种常见的编程模式，即将任务线程相关的数据封装在一个结构体中，然后在任务创建时传入到任务函数中。当任务将相关业务数据处理完成后删除相关资源。

上面这段代码看起来没什么太大的问题，对象分配的资源都在线程删除前得到释放。但上面这段代码仍然存在一个致命的缺陷，即可能出现hello_task线程无法被正常删除的情况。

FreeRTOS在创建任务时，如果被创建的任务优先级高于当前上下文的优先级，会在任务创建函数返回前就调度到新创建的任务中执行，这使得使得任务创建函数没有及时的返回，导致obj->task无法取得正确的值。线程hello_task执行太快且没有释放CPU执行权时，会导致在任务删除时obj->task没有取到正确的值，从而导致任务无法被正确删除。

可能大多数业务代码都不会执行的太快，但在一些极端情况还是需要考虑这个问题，例如上述代码中传入cnt为0时的情况。

解决这个问题有两个思路，一是使用xTaskCreate()替代xTaskCreateStatic()，xTaskCreate函数通过传参的方式取得任务句柄，不会出现上述问题，它可以保证在任务执行前就将句柄赋予正确的值。二是在使用xTaskCreateStatic接口前关闭任务调度器，在任务创建完成后在打开任务调度，确保任务创建函数正常返回后再启动任务调度器。

或许可以使用vTaskDelete(NULL)来删除当前线程，这样就能规避该问题。但通过上面这个示例可以看出这是一个时序不严谨的代码可能导致比较严重的问题，在实现其它类似这样的多线程代码时也应当注意类似的问题。
