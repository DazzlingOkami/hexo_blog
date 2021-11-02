---
title: SVC系统调用的编程使用方法
---

SVC称为系统服务调用(SuperVisorCall)，异常类型为11，通过svc指令可用触发异常，SVC在触发异常后必须立即得到相应(触发异常后在执行异常前不能执行其它代码)，除非有更高优先级的异常在执行。

对于可靠系统而言，可用使用SVC异常实现资源的特权访问，使系统更加安全。它通过SVC系统调用编号能够实现参数传递，从而实现不同功能的系统服务。使用系统服务可用不需要知道具体服务函数的地址，这样可用隐藏更多的细节。

在ARM开发工具链下，有比较优雅的方式直接使用SVC异常，在GCC下需要配合汇编来使用。

这里不讨论ARM处理器对于异常的处理机制，它在触发SVC指令后响应的异常行为与系统其它异常一致。

### SVC异常处理

```arm
  .syntax unified
  .thumb

  .global SVC_Handler
  .global SVC_VIRTUAL_CALL_0
  .global SVC_VIRTUAL_CALL_1
  .global SVC_VIRTUAL_CALL_2
  .global SVC_VIRTUAL_CALL_3

  .type SVC_Handler, %function
  .type SVC_VIRTUAL_CALL_0, %function
  .type SVC_VIRTUAL_CALL_1, %function
  .type SVC_VIRTUAL_CALL_2, %function
  .type SVC_VIRTUAL_CALL_3, %function

SVC_Handler:
  tst lr, #4
  ite eq
  mrseq r0, msp
  mrsne r0, psp

  push {r6, r7, lr}
  mov r7, r0

  ldr r0, [r7, #0]     //原始R0
  ldr r1, [r7, #4]
  ldr r2, [r7, #8]
  ldr r3, [r7, #12]
  ldr r6, [r7, #24]

  ldr r6, [r6, #-2]
  and r6, #0xFF
  ldr r12, =g_svc_vector
  ldr r6, [r12, r6, lsl #2]

  blx r6

  str r0, [r7, #0]

  pop {r6, r7, pc}

Default_SVC_Handler:
  bx lr

  .align 2
g_svc_vector:
  .word SVC_HANDLER_0
  .word SVC_HANDLER_1
  .word SVC_HANDLER_2
  .word SVC_HANDLER_3
  
  .weak      SVC_HANDLER_0
  .thumb_set SVC_HANDLER_0, Default_SVC_Handler
  .weak      SVC_HANDLER_1
  .thumb_set SVC_HANDLER_1, Default_SVC_Handler
  .weak      SVC_HANDLER_2
  .thumb_set SVC_HANDLER_2, Default_SVC_Handler
  .weak      SVC_HANDLER_3
  .thumb_set SVC_HANDLER_3, Default_SVC_Handler

SVC_VIRTUAL_CALL_0:
    svc 0
    bx lr

SVC_VIRTUAL_CALL_1:
    svc 1
    bx lr

SVC_VIRTUAL_CALL_2:
    svc 2
    bx lr

SVC_VIRTUAL_CALL_3:
    svc 3
    bx lr

```

SVC_Handler函数是异常处理函数，进入函数，判断LR寄存器来分析进入异常前的环境，获取到对应的sp地址。通过sp寄存器就能访问到栈空间，在栈中可以获取到r0-r3，这个就是执行svc时传进来的参数，通常这4个寄存器不会被修改，除非发生了更高优先级的异常。通过在栈中获取到执行svc的pc指针，得到svc指令的二进制码，低8位存储了指令的异常id，我们使用该id在内部通过查表得方式定位具体应该执行的函数入口，执行函数后，我们将r0也就是返回值存储到栈空间中，这样返回到用户态后，用户就能获得执行的返回结果。

为了方便定义用户系统调用函数，设计了这样的一个宏。

```c
#define SVC_CALL_DEF(id, func, ret_t, ...) \
extern ret_t SVC_VIRTUAL_CALL_##id(__VA_ARGS__); \
ret_t SVC_HANDLER_##id(__VA_ARGS__)
```
其中id表示该函数绑定到svc的那一个系统调用上，func表示函数名称，ret_t和后面的参数分别表示函数的返回值和形参列表，如果无形参则填入void。通过 SVC_VIRTUAL_CALL_id()来执行系统调用。

这是一个简单的例子：
```c
SVC_CALL_DEF(1, test_svc_add, int, int a, int b){
    return a + b;
}

int main(void){
    int ret = 0;
    ret = SVC_VIRTUAL_CALL_1(2, 3);

    printf("2+3 = %d\r\n", ret);
    return 0;
}
```

### 进阶写法
前面的定义方法和调用方法与传统的C语言函数定义和函数调用存在区别，所以在使用上存在不方便的地方，按照下面的写法更加合理，且函数的定义与调用与C语言类似，减低了代码移植上的一致性问题。
```c
#ifndef __SVC_HANDER_H
#define __SVC_HANDER_H
#if defined(__GNUC__)
#define DEF_SVC_FUNC(id, ret_t, name, ...) \
    ret_t name(__VA_ARGS__); \
    __asm__(        \
        ".thumb\n"  \
        ".global " #name "\n" \
        ".type " #name ", %function\n" \
        #name ":\n" \
        "svc " #id "\n" \
        "bx lr\n"); \
    ret_t SVC_HANDLER_##id(__VA_ARGS__)
#elif defined(__CC_ARM)
#define DEF_SVC_FUNC(id, ret_t, name, ...) \
    ret_t __svc(id) name(__VA_ARGS__)
#else
#define DEF_SVC_FUNC(id, ret_t, name, ...) \
    ret_t name(__VA_ARGS__)
    // #warning "not support SVC function!"
#endif
#endif
```

有了这个新的头文件后，定义一个svc函数以及函数的写法将变为：
```c
DEF_SVC_FUNC(2, int, add_func, int a, int b){
    return a + b;
}

int main(void){
    int ret = 0;
    ret = add_func(2, 3);
    printf("2+3 = %d\r\n", ret);
    return 0;
}
```
这样的写法能够兼容不同的编译器，且不用提前定义SVC_VIRTUAL_CALL_0函数。

