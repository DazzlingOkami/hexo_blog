---
title: TensorFlow Lite 移植步骤
author: WangGaojie
---

> 无论你使用CMake还是makefile，亦或是使用某些IDE，基本上都可以按照这样的步骤来操作。

## 1. 获取源码

从官方地址获取最新的 TFLite 源码：

```
https://github.com/tensorflow/tflite-micro
```

将其克隆到 Linux 主机上进行交叉编译，得到嵌入式平台的静态库。

## 2. 编译静态库

参考 `tensorflow/lite/micro/cortex_m_generic/README.md` 文件能够得到编译说明。

### 编译指令

```bash
make -f tensorflow/lite/micro/tools/make/Makefile TARGET=cortex_m_generic TARGET_ARCH=cortex-m7+fp OPTIMIZED_KERNEL_DIR=cmsis_nn BUILD_TYPE=debug microlite
```

### 参数说明

- `TARGET=cortex_m_generic`: 指定嵌入式平台为 Cortex-M7
- `TARGET_ARCH=cortex-m7+fp`: 开启硬件浮点单元
- `OPTIMIZED_KERNEL_DIR=cmsis_nn`: 使用 CMSIS 的神经网络库来优化计算
- `BUILD_TYPE=debug`: 开启调试信息方便调试代码

### 编译结果

执行指令后会在 `gen/cortex_m_generic_cortex-m7+fp_debug_cmsis_nn_gcc/lib` 目录中得到 `libtensorflow-microlite.a` 文件，这个文件就是需要添加到工程中的静态库。

## 3. 工程配置

### 3.1 转换为 C++ 工程

TFLite 为 C++ 工程，需要将我们的工程也转为 C++ 工程。

### 3.2 添加源码文件

除了静态库文件外，还需要将整个源码也加入到工程中。这些源码不参与编译，仅仅使用里面的头文件。

### 3.3 添加依赖库

在 Linux 主机端编译时，编译工具还额外下载了一些库，它们位于 `/tensorflow/lite/micro/tools/make/downloads/` 目录中。根据需要将 `gemmlowp` 和 `flatbuffers` 也拷贝到工程中。

## 4. 工程设置

在工程中需要做 3 个关键动作：

### 4.1 设置头文件路径

包括：
- TFLite 源码根目录
- `downloads/flatbuffers/include`
- `downloads/gemmlowp`

### 4.2 设置静态库和静态库路径

```bash
-ltensorflow-microlite -L"/tflite-micro"
```

### 4.3 设置编译宏

设置编译宏 `TF_LITE_STATIC_MEMORY`，这是为了和静态库编译脚本中保持一致。

## 5. 集成示例代码

保证编译通过后，可以尝试将代码示例中的 hello_world 工程加入进来。

### 5.1 添加示例代码

文件路径：`/tensorflow/lite/micro/examples/hello_world/hello_world_test.cc`

### 5.2 添加模型文件

模型文件在 `gen/cortex_m_generic_cortex-m7+fp_debug_cmsis_nn_gcc/genfiles` 目录中，包含四个文件：

- `hello_world_float_model_data.cc`
- `hello_world_float_model_data.h`
- `hello_world_int8_model_data.cc`
- `hello_world_int8_model_data.h`

### 5.3 修改 main 函数

由于工程中已经有 main 函数了，需要将 `hello_world_test.cc` 中的 main 函数名称修改一下，然后在 main 函数中调用它即可。

需要注意的是 变量tflite::MicroProfiler profiler;是一个较大的对象，会分配较大的栈空间，这里需要根据实际情况对代码进行调整。

### 5.4 调整头文件引用路径

两个 `model_data.cc` 文件中的头文件引用路径可能和实际情况不一致，按照实际情况进行调整即可。

## 6. 日志输出配置

为了观察代码运行情况，需要向 TFLite 注入一个日志函数接口。

### 6.1 实现日志函数

```c
#include "tensorflow/lite/micro/cortex_m_generic/debug_log_callback.h"
#include "usart/usart.h"

static void tflite_log(const char *s){
    uart_puts(s);
}
```

### 6.2 注册日志回调

```c
int main(void){
    // ...
    RegisterDebugLogCallback(tflite_log);
    // ...
}
```

主要通过 `RegisterDebugLogCallback` 注册一个输出接口。


