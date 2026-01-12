---
title: 在单片机上试试最新的Javascript引擎
author: WangGaojie
---

## 简介

前段时间，Fabrice Bellard 大佬发布了他的又一力作 [mquickjs](https://github.com/bellard/mquickjs)。我得知后第一时间就进行了测试，它是一款面向嵌入式软件平台的 JavaScript 引擎，能够适配嵌入式平台有限的资源环境。

该引擎对 JS 特性的支持有限，功能接近 ES5。为了更好地适配嵌入式环境，牺牲部分复杂的语法特性是一种合理的设计思路。

几年前我曾移植过 QuickJS，其运行时（runtime）结构复杂，移植过程颇为艰辛。而此次的 mquickjs 则简洁许多，只需对三个文件中的部分代码进行平台适配，就能顺利运行起来。

## 移植过程

下面简单说说将其移植到单片机的过程及注意事项。

首先，将代码克隆到本地，通过 Makefile 编译一遍。这么做主要是为了生成两个关键头文件——mqjs_stdlib.h 和 mquickjs_atom.h。这两个文件由主机平台上的 mqjs_stdlib 程序生成，内容主要是 JS 内部类相关的常量信息。

接下来，将生成的头文件及源码中的相关文件添加到单片机工程中。其中，example_stdlib.c、example.c、mqjs_stdlib.c、mquickjs_build.c 及其关联的头文件无需使用，需剔除出工程。

移植的核心工作是修改两个文件以适配目标平台的运行时，具体如下：

### 1. readline_tty.c 文件移植

该文件负责 REPL（交互式解释器）模式下的输入输出交互，若无需启用 REPL 模式，此文件可省略。若保留 REPL 模式，需做如下修改：

首先是 readline_tty_init 函数，该函数用于返回交互式终端的宽度。以 PC 端串口终端为例，可直接让该函数返回 80（大多数终端的默认宽度）。

term_printf 是格式化输出函数，适配串口输出的实现示例如下：

```c
void term_printf(const char *fmt, ...)
{
    static char buf[1024];
    int buf_len;
    va_list ap;
    va_start(ap, fmt);
    buf_len = vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);
    HAL_UART_Transmit(&huart4, buf, buf_len, 100);
}
```

term_flush 函数用于输出刷新。若 term_printf 采用中断或 DMA 方式输出数据，可在该函数中进行数据同步，确保数据传输完成。

readline_tty 是该文件的核心函数，可删除其中针对 Win32 平台的代码，但需保留 Ctrl+C 信号的处理逻辑。函数内部需重点关注 `len = read(0, buf, sizeof(buf));` 这行代码——它用于阻塞式读取数据，无数据时会阻塞等待，读取到数据后返回实际读取长度。若改用串口读取，需实现对应的非阻塞读取逻辑，示例如下：

```c
// 需要配合缓存队列
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart){
  if(huart == &huart4){
    char c = recv_ch;
    HAL_UART_Receive_IT(&huart4, &recv_ch, 1);
    ringbuf_put(&recv_buf, c);
  }
}
int uart_read(uint8_t *buf, int len){
  int rlen = ringbuf_elements(&recv_buf);
  if(rlen == 0){
    if((HAL_UART_GetState(&huart4) & 2) == 0){
      HAL_UART_Receive_IT(&huart4, &recv_ch, 1);
    }
    return 0;
  }
  if(len > rlen){
    len = rlen;
  }
  ringbuf_gets(&recv_buf, (char *)buf, len);
  return len;
}
// 替换原有 read 调用：
// len = read(0, buf, sizeof(buf));
// 改为：
// while(1){
//     len = uart_read(buf, sizeof(buf));
//     if(len > 0) break;
//     HAL_Delay(1);
// }
```

我使用 MobaXterm 的串口终端进行测试，可正常发送 Ctrl+C 信号，因此保留了原有中断信号处理逻辑。至此，readline_tty.c 文件移植完成。

### 2. mqjs.c 文件移植

将任何 C 代码移植到嵌入式平台，都需重点关注运行时依赖——明确代码所需的底层支持，才能有针对性地完成移植。mqjs.c 的核心依赖包括：动态内存管理（malloc、free）、文件读写、日期时间、延时休眠、标准输出（printf）、系统调用（exit）、随机数生成器。

其中，日期时间、延时、标准输出的适配相对简单，只需替换 fwrite(..., stdout)、get_time_ms、js_date_now、nanosleep 等接口的实现即可，这些功能在嵌入式平台上均容易实现。

JS_SetRandomSeed 需传入随机数种子，可利用单片机的硬件随机数发生器生成。

文件读写相关逻辑集中在 compile_file、load_file 函数中，可根据需求选择性实现：若仅需运行 REPL 模式，可移除这两个函数相关的所有功能；若需通过单片机内部 Flash 实现简单文件存储，可实现 load_file 函数以支持基准测试等功能。

内存管理是 mqjs 移植到单片机的关键难点：其默认需要 16MB 内存，这对于大多数单片机的内部 SRAM 而言并不现实，因此需依赖扩展 SDRAM 才能正常运行。若将内存限制在 128KB 左右，仅能在 REPL 模式下执行简单测试——执行 test_builtin.js 测试需约 500KB 内存，执行 microbench.js 基准测试则需约 8MB 内存。

确定内存资源后，还需选择合适的内存管理器：C 库内存管理器、FreeRTOS 内存管理器或其他开源内存管理器均可，但需注意移植适配性。前段时间，我利用午休时间出于兴趣写了一个简单的内存分配器，虽仅作学习娱乐之用，但非常适合嵌入式裸机环境，代码如下：

```c
#ifndef _MINI_MALLOC_H_
#define _MINI_MALLOC_H_
#include <stddef.h>
#ifndef DEFAULT_HEAP
#define DEFAULT_HEAP (0x20000000)
#define DEFAULT_SIZE (4096 * 1024)
#endif
#define HEAP_ALIGN (4)
#define ALLOC_MAGIC ((heap_node_t *)0xDEADBEEF)
struct heap_node {
    unsigned int size;
    struct heap_node *next;
    // bytes[size];
};
typedef struct heap_node heap_node_t;
static inline void mini_heap_init(void){
    heap_node_t *node = (heap_node_t *)DEFAULT_HEAP;
    node->size = DEFAULT_SIZE - sizeof(heap_node_t);
    node->next = NULL;
}
static inline void* mini_malloc(unsigned int size){
    if(size == 0) return NULL;
    size = (size + (HEAP_ALIGN - 1)) & (~(HEAP_ALIGN - 1));
    heap_node_t *node = (heap_node_t *)DEFAULT_HEAP;
    heap_node_t *last = NULL;
    if(node->size >= size + sizeof(heap_node_t)){
        node->size -= (size + sizeof(heap_node_t));
        node = (heap_node_t *)(((char*)(node + 1)) + node->size);
        node->size = size;
        node->next = ALLOC_MAGIC;
        return node + 1;
    }
    last = node;
    node = node->next;
    while(node){
        if(node->size >= size){
            if(node->size <= size + sizeof(heap_node_t)){
                last->next = node->next;
                node->next = ALLOC_MAGIC;
                return node + 1;
            }
            node->size -= (size + sizeof(heap_node_t));
            node = (heap_node_t *)(((char*)(node + 1)) + node->size);
            node->size = size;
            node->next = ALLOC_MAGIC;
            return node + 1;
        }
        last = node;
        node = node->next;
    }
    return NULL;
}
static inline void mini_free(void *ptr){
    if(ptr == NULL) return;
    heap_node_t *node = (heap_node_t *)DEFAULT_HEAP;
    heap_node_t *pnode = (heap_node_t *)ptr;
    pnode -= 1;
    if(pnode->next != ALLOC_MAGIC){
        return ;
    }
    while(node){
        if(pnode > node && (pnode < node->next || node->next == NULL)){
            pnode->next = node->next;
            node->next = pnode;
            if((char*)node + sizeof(heap_node_t) + node->size == (char*)pnode){
                node->size += sizeof(heap_node_t) + pnode->size;
                node->next = pnode->next;
            }
            if((char*)node + sizeof(heap_node_t) + node->size == (char*)(node->next)){
                node->size += sizeof(heap_node_t) + node->next->size;
                node->next = node->next->next;
            }
            return ;
        }
        node = node->next;
    }
}
#endif
```

PS：该内存分配器无线程安全机制，不建议用于生产环境，仅作学习参考。

回到 mqjs 移植，最后需处理的是 exit 系统调用——这是移植过程中的一个“大坑”，原因有二：一是大多数单片机运行环境不支持标准 exit 调用；二是通过 exit 进行大跨度上下文跳转会导致资源泄露（如文件句柄、动态内存未释放）。

暂不考虑资源泄露问题，可通过以下方式在单片机中模拟 exit 机制：

```c
static jmp_buf exit_jmp;
static void mqjs_exit(int state){
    longjmp(exit_jmp, state);
}
#define exit(s) mqjs_exit(s)
int mqjs_main(int argc, const char **argv){
    if(setjmp(exit_jmp))
        goto exit;            
    // ...
    buf = malloc(100);
    if(buf == NULL){
        exit(1);
    }
exit:
    return 0;
}
```

此外，mquickjs.c 中还有一个易被忽略的系统调用——abort()，该函数仅用于处理极端严重的错误流程，可暂时忽略，或直接用 exit 替换。

通过上述修改，可让 mqjs 快速运行起来，资源泄露问题可后续优化解决。

## 基准测试

我在嵌入式平台上完成了完整移植（包括文件读写功能），并运行了所有测试用例，均顺利通过！下面重点说说测试过程中的关键发现：

```js
assert((25).toExponential(), "2.5e+1");
assert((25).toExponential(0), "3e+1");
assert((-25).toExponential(0), "-3e+1");
assert((2.5).toPrecision(1), "3");
assert((-2.5).toPrecision(1), "-3");
assert((25).toPrecision(1), "3e+1");
assert((1.125).toFixed(2), "1.13");
assert((-1.125).toFixed(2), "-1.13");
assert((-1e-10).toFixed(0), "-0");
```

上述代码涉及数字转字符串功能，看似简单，却是旧版 QuickJS 的“重灾区”：QuickJS 的浮点数转字符串四舍五入精度机制依赖 C 库转换函数，导致不同编译器编译的版本在执行四舍五入时结果不一致，进而导致测试用例失败。

而 mquickjs 中，数字到字符串的转换已不再依赖 C 库——dtoa.c 文件专门负责此功能。正因为不依赖 C 库，其在不同平台下的运行结果完全一致。

最后对比 Windows 平台与 STM32H750 单片机执行 microbench.js 基准测试的结果：

| Platform                                                                                   | i3-10100 |          | STM32H750 |          | SCORE(%) |
| ------------------------------------------------------------------------------------------ | -------- | -------- | --------- | -------- | -------- |
| TEST                                                                                       | N        | TIME(ns) | N         | TIME(ns) | SCORE(%) |
| empty_loop                                                                                 | 10000000 | 13       | 500000    | 340      | 26.15    |
| date_now                                                                                   | 2000000  | 80       | 20000     | 5000     | 62.50    |
| prop_read                                                                                  | 2000000  | 13.5     | 100000    | 350      | 25.93    |
| prop_write                                                                                 | 5000000  | 11.5     | 100000    | 275      | 23.91    |
| prop_update                                                                                | 1000000  | 32.5     | 50000     | 475      | 14.62    |
| prop_create                                                                                | 100000   | 72.5     | 2000      | 2500     | 34.48    |
| prop_delete                                                                                | 1000     | 110      | 50        | 3200     | 29.09    |
| array_read                                                                                 | 1000000  | 11       | 50000     | 280      | 25.45    |
| array_write                                                                                | 2000000  | 9        | 50000     | 200      | 22.22    |
| array_update                                                                               | 500000   | 31       | 50000     | 360      | 11.61    |
| array_prop_create                                                                          | 5000     | 20       | 200       | 650      | 32.50    |
| array_length_read                                                                          | 2000000  | 17.5     | 100000    | 275      | 15.71    |
| array_length_decr                                                                          | 2000     | 80       | 50        | 3100     | 38.75    |
| array_push                                                                                 | 5000     | 48       | 200       | 1700     | 35.42    |
| array_pop                                                                                  | 5000     | 48       | 200       | 1400     | 29.17    |
| typed_array_read                                                                           | 500000   | 30       | 10000     | 950      | 31.67    |
| typed_array_write                                                                          | 500000   | 22       | 20000     | 600      | 27.27    |
| closure_read                                                                               | 2000000  | 13.75    | 100000    | 250      | 18.18    |
| closure_write                                                                              | 5000000  | 7.25     | 200000    | 175      | 24.14    |
| global_read                                                                                | 2000000  | 15       | 100000    | 250      | 16.67    |
| global_write_strict                                                                        | 5000000  | 6.75     | 200000    | 175      | 25.93    |
| func_call                                                                                  | 1000000  | 35       | 50000     | 650      | 18.57    |
| closure_var                                                                                | 1000000  | 40       | 50000     | 700      | 17.50    |
| int_arith                                                                                  | 5000     | 22       | 200       | 550      | 25.00    |
| float_arith                                                                                | 1000     | 100      | 20        | 5500     | 55.00    |
| array_for                                                                                  | 50000    | 24       | 2000      | 600      | 25.00    |
| array_for_in                                                                               | 20000    | 94       | 500       | 2400     | 25.53    |
| array_for_of                                                                               | 100000   | 17       | 5000      | 420      | 24.71    |
| math_min                                                                                   | 2000     | 55       | 100       | 1100     | 20.00    |
| regexp_ascii                                                                               | 500      | 240      | 10        | 12000    | 50.00    |
| regexp_utf16                                                                               | 500      | 240      | 10        | 15000    | 62.50    |
| regexp_replace                                                                             | 100      | 1200     | 2         | 66000    | 55.00    |
| string_length                                                                              | 2000000  | 16.25    | 100000    | 275      | 16.92    |
| string_build1                                                                              | 500      | 800      | 50        | 13000    | 16.25    |
| string_build2                                                                              | 500      | 800      | 50        | 13000    | 16.25    |
| sort_bench                                                                                 | 1        | 6        | 1         | 162.2    | 27.03    |
| int_to_string                                                                              | 1000000  | 130      | 20000     | 5400     | 41.54    |
| float_to_string                                                                            | 200000   | 460      | 2000      | 50000    | 108.70   |
| string_to_int                                                                              | 1000000  | 100      | 50000     | 3400     | 34.00    |
| string_to_float                                                                            | 1000000  | 135      | 20000     | 5200     | 38.52    |
| total                                                                                      |          | 5206.5   |           | 217862.2 | 41.84    |

测试结果显示，STM32H750 上的执行时间约为桌面 PC（i3-10100）的 40 倍。其中，浮点运算、浮点数转字符串、正则表达式等测试项耗时较长，符合嵌入式平台的性能特点。

mquickjs还支持编译后运行。通过运行test_builtin.js实测发现，先编译为字节码再运行比直接运行js文件在运行时间上节省了48%。此外，运行字节码文件时heap资源消耗也大幅度减小，执行字节文件时heap使用了323kb，而直接运行js文件所使用的heap空间为489.6kb。

## 添加一个自定义模块

基础功能移植完成后，仅能体验 JavaScript 的基本语法。为适配单片机平台，下面尝试添加一个 GPIO 控制模块，实现 IO 读写、中断检测等功能。

官方代码库中的 example_stdlib.c 和 example.c 提供了模块开发的参考范式，实现自定义模块的核心步骤如下：

### 1. 定义模块结构

首先在 mqjs_stdlib.c 文件中定义 GPIO 类的结构，包括类名称、属性、方法等，代码如下：

```c
// 类实例的属性及方法
static const JSPropDef js_gpio_proto[] = {
    JS_CGETSET_DEF("value", js_gpio_read, js_gpio_write), // 可读写属性
    JS_CFUNC_DEF("write", 1, js_gpio_write),     // 修改 IO 电平（1 个参数）
    JS_CFUNC_DEF("read", 0, js_gpio_read),       // 读取 IO 电平（无参数）
    JS_CFUNC_DEF("watch", 1, js_gpio_watch),     // 设置 IO 中断回调函数
    JS_PROP_END
};
// 类静态属性及方法（此处无需定义）
static const JSPropDef js_gpio[] = {
    JS_PROP_END,
};
// 自定义类 ID
#define JS_CLASS_GPIO (JS_CLASS_USER + 0)
// 类定义：类名称、构造函数参数个数、构造函数、类 ID、静态方法、实例属性、父类、析构函数
static const JSClassDef js_gpio_obj = JS_CLASS_DEF("GPIO", 3, js_gpio_constructor, JS_CLASS_GPIO, js_gpio, js_gpio_proto, NULL, js_gpio_finalizer);
```

### 2. 注册模块到全局对象

将定义的 js_gpio_obj 类添加到全局对象列表中，使 JavaScript 代码可直接访问，代码如下：

```c
static const JSPropDef js_global_object[] = {
    // ... 其他全局对象 ...
    JS_CFUNC_DEF("print", 1, js_print),
    JS_PROP_CLASS_DEF("GPIO", &js_gpio_obj), // 注册 GPIO 类
    // ... 其他全局对象 ...
};
```

修改完成后，重新运行 mqjs_build 工具，生成新的 mqjs_stdlib.h 头文件。

### 3. 实现模块核心逻辑

接下来需用 C 语言实现 GPIO 类的构造函数、析构函数、属性读写、方法等核心逻辑——这是模块开发的核心难点。

#### （1）构造函数实现

GPIO 构造函数接收 3 个参数：引脚编号（如 "PA0"）、IO 方向（"in" 输入 / "out" 输出）、上下拉状态（"up" 上拉 / "down" 下拉 / "none" 无）。

```c
// 保存 GPIO 硬件信息
typedef struct {
    GPIO_TypeDef *GPIOx;
    uint16_t GPIO_Pin;
} js_gpio_data_t;
/* 
JavaScript 调用示例：
var pa0;
pa0 = new GPIO("PA0", "in", "up");
 */
JSValue js_gpio_constructor(JSContext *ctx, JSValue *this_val, int argc, JSValue *argv){
    JSValue obj;
    // 检查是否通过 new 关键字调用
    if (!(argc & FRAME_CF_CTOR))
        return JS_ThrowTypeError(ctx, "must be called with new");
    argc &= ~FRAME_CF_CTOR;
    // 检查参数个数
    if(argc < 3){
        return JS_ThrowTypeError(ctx, "invalid arguments");
    }
    // 分配 GPIO 数据内存
    js_gpio_data_t *gpio = malloc(sizeof(js_gpio_data_t));
    if(gpio == NULL){
        return JS_ThrowInternalError(ctx, "out of memory");
    }
    // 创建类实例
    obj = JS_NewObjectClassUser(ctx, JS_CLASS_GPIO);
    if (JS_IsException(obj)){
        free(gpio);
        return obj;
    }
    // 绑定私有数据到类实例
    JS_SetOpaque(ctx, obj, gpio);
    // 解析参数
    JSCStringBuf buf;
    const char *str;
    // 解析引脚编号（如 "PA0"）
    str = JS_ToCString(ctx, argv[0], &buf);
    js_gpio_init(gpio, str);
    // 解析 IO 方向
    str = JS_ToCString(ctx, argv[1], &buf);
    int mode = strcmp(str, "in") == 0 ? GPIO_MODE_INPUT : GPIO_MODE_OUTPUT_PP;
    // 解析上下拉状态
    str = JS_ToCString(ctx, argv[2], &buf);
    int pull = GPIO_NOPULL;
    if(strcmp(str, "up") == 0){
        pull = GPIO_PULLUP;
    }else if(strcmp(str, "down") == 0){
        pull = GPIO_PULLDOWN;
    }
    // 使能 GPIO 时钟
    if(gpio->GPIOx == GPIOA){
        __HAL_RCC_GPIOA_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOB){
        __HAL_RCC_GPIOB_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOC){
        __HAL_RCC_GPIOC_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOD){
        __HAL_RCC_GPIOD_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOE){
        __HAL_RCC_GPIOE_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOF){
        __HAL_RCC_GPIOF_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOG){
        __HAL_RCC_GPIOG_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOH){
        __HAL_RCC_GPIOH_CLK_ENABLE();
    }else if(gpio->GPIOx == GPIOI){
        __HAL_RCC_GPIOI_CLK_ENABLE();
    }
    // 初始化 GPIO 配置
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = gpio->GPIO_Pin;
    GPIO_InitStruct.Mode = mode;
    GPIO_InitStruct.Pull = pull;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(gpio->GPIOx, &GPIO_InitStruct);
    return obj;
}
// 解析引脚编号到硬件 GPIO 结构体
static void js_gpio_init(js_gpio_data_t *gpio, const char *name){
    // 解析端口（如 "PA0" 中的 "A"）
    if(name[1] == 'A'){
        gpio->GPIOx = GPIOA;
    }else if(name[1] == 'B'){
        gpio->GPIOx = GPIOB;
    }else if(name[1] == 'C'){
        gpio->GPIOx = GPIOC;
    }else if(name[1] == 'D'){
        gpio->GPIOx = GPIOD;
    }else if(name[1] == 'E'){
        gpio->GPIOx = GPIOE;
    }else if(name[1] == 'F'){
        gpio->GPIOx = GPIOF;
    }else if(name[1] == 'G'){
        gpio->GPIOx = GPIOG;
    }else if(name[1] == 'H'){
        gpio->GPIOx = GPIOH;
    }else if(name[1] == 'I'){
        gpio->GPIOx = GPIOI;
    }
    // 解析引脚号（如 "PA0" 中的 "0"）
    gpio->GPIO_Pin = (1 << (name[2] - '0')); // todo：可扩展支持两位引脚号（如 PA10）
}
```

关键说明：通过 `JS_NewObjectClassUser` 创建自定义类实例，通过 `JS_SetOpaque` 绑定私有数据（GPIO 硬件信息）；通过 `JS_ToCString` 将 JavaScript 参数转为 C 字符串，需注意 `JSCStringBuf` 缓存的长度限制（5 字节），避免传入过长字符串。

#### （2）IO 读写方法实现

读写方法直接调用 HAL 库接口操作底层 IO，逻辑简单，代码如下：

```c
// 读取 IO 电平
JSValue js_gpio_read(JSContext *ctx, JSValue *this_val, int argc, JSValue *argv){
    js_gpio_data_t *gpio = JS_GetOpaque(ctx, *this_val);
    int v = HAL_GPIO_ReadPin(gpio->GPIOx, gpio->GPIO_Pin);
    return JS_NewInt32(ctx, v);
}
// 写入 IO 电平
JSValue js_gpio_write(JSContext *ctx, JSValue *this_val, int argc, JSValue *argv){
    if (JS_IsInt(argv[0])){
        js_gpio_data_t *gpio = JS_GetOpaque(ctx, *this_val);
        int v = JS_VALUE_GET_INT(argv[0]);
        v = v ? GPIO_PIN_SET : GPIO_PIN_RESET;
        HAL_GPIO_WritePin(gpio->GPIOx, gpio->GPIO_Pin, v);
        return JS_UNDEFINED;
    }else{
        return JS_ThrowTypeError(ctx, "must be write integer");
    }
}
```

#### （3）中断检测功能实现

中断检测是模块开发的难点：JavaScript 引擎为单线程模型，无类似 MicroPython 的全局锁机制，中断处理不能脱离 JS 线程独立运行。mquickjs 提供了`JS_SetInterruptHandler` 接口，用于绑定全局中断轮询函数，可在该函数中处理多个中断事件。

##### 步骤 1：绑定全局中断处理函数

官方代码仅在 REPL 模式下绑定中断处理函数，为支持脚本运行时的中断响应，需在创建 JS 上下文后全局绑定，代码如下：

```c
// 全局中断处理函数
static int js_interrupt_handler(JSContext *ctx, void *opaque)
{
    js_gpio_interrupt_handler(ctx); // GPIO 中断处理
    return readline_is_interrupted(); // 保留 Ctrl+C 中断检测
}

// 在创建 JS 上下文后绑定
ctx = JS_NewContext(mem); // mem 为内存分配器相关指针，前文内存管理部分已定义
if (!ctx) {
    return 1;
}
// 绑定全局中断处理函数，实现中断轮询
JS_SetInterruptHandler(ctx, js_interrupt_handler, NULL);
```

##### 步骤 2：GPIO 中断绑定处理机制

```c
// 首先定义了一个结构体，保存gpio对象和回调函数
typedef struct {
    JSGCRef func;
    JSGCRef gpio_ref;
    void *opaque;
    BOOL allocated;
    uint8_t last_value;
} js_gpio_watcher_t;

// 这里使用静态数组保存gpio对象和回调函数，如果需要无线绑定watch，需要改为链表模式
#define JS_GPIO_WATCH_COUNT (10)
static js_gpio_watcher_t js_gpio_watch_list[JS_GPIO_WATCH_COUNT] = {0};

static js_gpio_watcher_t* get_watcher_by_val(JSValue val){
    for(int i = 0; i < JS_GPIO_WATCH_COUNT; i++){
        if (js_gpio_watch_list[i].allocated && 
            js_gpio_watch_list[i].gpio_ref.val == val)
            return &js_gpio_watch_list[i];
    }
    return NULL;
}

static js_gpio_watcher_t* get_watcher_by_opaque(void *opaque){
    for(int i = 0; i < JS_GPIO_WATCH_COUNT; i++){
        if (js_gpio_watch_list[i].allocated && 
            js_gpio_watch_list[i].opaque == opaque)
            return &js_gpio_watch_list[i];
    }
    return NULL;
}

static js_gpio_watcher_t* get_idle_watcher(void){
    for(int i = 0; i < JS_GPIO_WATCH_COUNT; i++){
        if (!js_gpio_watch_list[i].allocated)
            return &js_gpio_watch_list[i];
    }
    return NULL;
}

/* 
function io_watch(v){
    console.log("irq");
    console.log(this.value);
}
pa0.watch(io_watch);
 */
JSValue js_gpio_watch(JSContext *ctx, JSValue *this_val, int argc, JSValue *argv){
    js_gpio_watcher_t *watcher = get_watcher_by_val(*this_val);
    if(watcher){
        JS_DeleteGCRef(ctx, &watcher->func);
        JS_DeleteGCRef(ctx, &watcher->gpio_ref);
        watcher->allocated = FALSE;
        return JS_UNDEFINED;
    }

    if(JS_IsNull(argv[0])){
        return JS_UNDEFINED;
    }

    if (!JS_IsFunction(ctx, argv[0]))
        return JS_ThrowTypeError(ctx, "not a function");

    watcher = get_idle_watcher();
    if(watcher == NULL){
        return JS_ThrowInternalError(ctx, "too many watchers");
    }
    JSValue *pfunc = JS_AddGCRef(ctx, &watcher->func);
    *pfunc = argv[0];
    JSValue *gpio_obj = JS_AddGCRef(ctx, &watcher->gpio_ref);
    *gpio_obj = *this_val;
    watcher->opaque = JS_GetOpaque(ctx, *this_val);
    watcher->allocated = TRUE;

    js_gpio_data_t *gpio = watcher->opaque;
    watcher->last_value = HAL_GPIO_ReadPin(gpio->GPIOx, gpio->GPIO_Pin);
    return JS_UNDEFINED;
}
```

在watch处理函数中，首先需要查询当前GPIO是否已经绑定了中断处理函数，如果已经绑定了则需要进行解绑操作。如果传入的是null而不是一个函数，则说明仅进行解绑而不需要绑定新的函数。
绑定新的函数时，需要保存gpio对象和回调函数。不能脱离js线程保存JSValue类型的变量，需要使用JSGCRef来间接引用保存数据。这是因为JS的内存管理方式和GC机制导致引用的变量不能脱离js线程保存。mquickj的内存管理依靠一个free指针来哪些空间被使用和释放，所以在进行非连续空间的释放时，垃圾回收机制需要将不连续的内存进行合并，所以JS内部对象的实际存储地址会随着垃圾回收移动而改变。所以不能直接保存JSValue类型的变量。相关的逻辑可以看看函数`js_malloc`、`js_free`、`gc_compact_heap`。
起初我没有意识到JSValue的存储问题导致，导致频繁的内存存储，跟踪了很久的代码。最后发现原作者已经在readme文件中强调了该问题。

##### 步骤 3：GPIO 中断处理函数

```c
void js_gpio_interrupt_handler(JSContext *ctx){
    for(int i = 0; i < countof(js_gpio_watch_list); i++){
        js_gpio_watcher_t *watcher = &js_gpio_watch_list[i];
        if(watcher->allocated == FALSE){
            continue;
        }
        JSValue gpio_obj = watcher->gpio_ref.val;
        js_gpio_data_t *gpio = JS_GetOpaque(ctx, gpio_obj);
        uint8_t value = HAL_GPIO_ReadPin(gpio->GPIOx, gpio->GPIO_Pin);
        if(value != watcher->last_value){
            watcher->last_value = value;
            if (JS_StackCheck(ctx, 3)){
                return ;
            }
            JS_PushArg(ctx, JS_NewInt32(ctx, value));   // arg[0]
            JS_PushArg(ctx, watcher->func.val);         // func
            JS_PushArg(ctx, gpio_obj);                  // this
            JSValue ret = JS_Call(ctx, 1);
            if (JS_IsException(ret)){
                return ;
            }
        }
    }
}
```

通过watch列表检查哪些GPIO需要监控IO的状态。当IO的电平状态发生变化时通过JS_Call执行对应的回调函数。

#### （4）析构函数别忘了

```c
void js_gpio_finalizer(JSContext *ctx, void *opaque){
    js_gpio_data_t *gpio = opaque;
    HAL_GPIO_DeInit(gpio->GPIOx, gpio->GPIO_Pin);

    js_gpio_watcher_t *watcher = get_watcher_by_opaque(opaque);
    if(watcher){
        JS_DeleteGCRef(ctx, &watcher->func);
        JS_DeleteGCRef(ctx, &watcher->gpio_ref);
        watcher->allocated = FALSE;
    }
    free(opaque);
}
```

这里需要强调一点，由于在watch调用中引用了GPIO自身，所以这里出现了内存的循环依赖链，对GC垃圾回收来说，即使GPIO对象的变量引用丢失，其内存也不会被释放。需要确保对象释放前主动调用watch(null)来解除绑定关系，以确保垃圾回收能够正常释放相关内存。

### 3. GPIO模块测试

测试代码如下：

1. 周期性设置GPIO的状态
   
   ```js
   function sleep(delayMs) {
    if (typeof delayMs !== 'number' || isNaN(delayMs) || delayMs < 0) {
        throw new Error('delayMs type error'); 
    }
    var startTime = Date.now();
    while (Date.now() - startTime < delayMs) { }
   }
   var ph9;
   ph9 = new GPIO("PH9", "out", "none");
   ph9.value = 1
   for(var i = 0; i < 10; i++) {
    ph9.write(1 - ph9.value); // toggle
    console.log(ph9.value);
    sleep(800);
   }
   ```
   
   这里先定义了一个sleep函数，用于延时等待。使用GPIO类创建一个gpio对象来操作ph9引脚，每0.8秒反转GPIO的状态。

2. 读取GPIO
   
   ```js
   load("sleep.js")
   var ph4;
   ph4 = new GPIO("PH4", "in", "up"); // PH4 input pull up
   console.log(ph4.value);
   for(var i = 0; i < 10; i++) {
    console.log(ph4.value);
    sleep(800);
   }
   ```
   
   将前面的sleep函数单独保存为一个文件，并加载进来。然后每0.8秒读取并输出p4引脚的状态，通实际控制引脚的电平变化确认功能正常。

3. 监听GPIO
   
   ```js
   function ph4_irq(value){
    console.log("PH4 interrupt: " + value);
   }
   var ph4;
   ph4 = new GPIO("PH4", "in", "up");
   console.log(ph4.value);
   ph4.watch(ph4_irq);
   sleep(8000);
   ```
   
   执行该代码后，点击对应引脚的按钮，能够看到输出的对应中断信息。

## 总结

后续考虑扩展一些更加复杂的模块，比如将网络协议栈集成进来等，实现方法可以参考这里的GPIO模块实现方法。现在基础的框架已经做好了，后面的功能慢慢添加。总的来说mqjs移植要比micropython要简单很多，但micropython的功能更多，资源消耗也要少一些。相比于lua这类脚本语言来说，js又能体现语法上的灵活性优势。所以mquickjs是一个值得学习和研究的一个嵌入式软件项目。
