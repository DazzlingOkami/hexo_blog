---
title: PIO参考手册
author: WangGaojie
---

> 本文原始文档来自于[RP2040官方文档](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)，该译文内容源自互联网 https://github.com/charlee/rp2040-pio-zhcn
> 本文的编写重新参考了最新的RP2350官方文档，更新并补充了许多新的内容，包括新版本PIO的特性和新加入的一些指令等。

## PIO概要

RP2040有两个完全相同的PIO模块，RP2350有3个PIO模块。每个PIO块都独立连接到总线、GPIO和中断控制器。一个 PIO 块的结构图如下图所示。

| ![图38](figures/figure-38.png) |
|:--:|
| 图38. PIO块结构图。RP2040有两个PIO块，每个块有四个状态机。四个状态机可以从一片共享指令内存中同时执行指令。<br>FIFO数据队列负责在PIO和系统之间缓冲传输的数据。每个状态机可以通过GPIO映射逻辑监视并操作最多30个GPIO。|

可编程输入输出块（PIO）是一个非常灵活的硬件接口。它能支持许多IO标准，例如：

- 8080 和 6800 并行总线
- I2C
- 3针I2S
- SDIO
- SPI、DSPI、QSPI
- UART
- DPI 或 VGA（通过电阻式DAC）

PIO的编程方式与处理器类似。每个PIO有四个状态机，每个状态机可以独立执行顺序代码，操作 GPIO 并传输数据。但不同于通用处理器，PIO 状态机是专门为输入输出设计的，因此具有确定性、精确的时间，并与固定功能的硬件紧密结合。

每个状态机都拥有以下内容：
- 两个 32 位移位寄存器 - 可以向任意方向位移任意比特
- 两个 32 位可擦写寄存器（scratch registor）
- 每个方向上（TX/RX）都有 4x32 位总线 FIFO，可以重新配置为单向 8x32 FIFO
- 支持小数的时钟分频器（16 位整数，8 位小数）
- 灵活的 GPIO 映射
- DMA 接口，每个时钟周期最多可传输来自系统 DMA 的 1 个字
- IRQ 标志设置/清除/状态查询

每个状态机及其相关的硬件所占用的芯片面积大约相当于标准的串行接口，如 SPI 或 I2C 控制器等。但是，PIO 状态机可以动态配置或重新配置，实现多种不同的接口。

状态机能用软件的形式编程，相比于像 CPLD 那样采用完全可配置的逻辑芯片，PIO能在同样的成本和功耗下提供更多的硬件接口。它还带来了人们更为熟悉的编程模型，和更简单的工具流程，因此人们可以利用 PIO 的灵活性直接进行编程，而不需要使用 PIO 库中事先定义好的接口。

PIO 不仅非常灵活，性能还非常高，这都要归功于每个状态机中的固定功能的硬件。在输出 DPI 时，PIO 能够在 48MHz 系统时钟的支持下，在活跃的扫描线期间提供 360Mb/s 的吞吐量。在这个例子中，一个状态机负责处理帧和扫描线的时间，并生成像素时钟，另一个负责处理像素数据，并解包行程编码的扫描线。

状态机的输入和输出可以映射到最多 32 个 GPIO 上（在 RP2040 中限制为 30 个 GPIO），所有状态机都能独立、同时访问任何 GPIO。例如，标准的 UART 代码允许 TX、RX、CTS 和 RTS 使用任意四个 GPIO，而 I2C 也允许将 SDA 和 SCL 映射到任意 GPIO 上。具体的自由度取决于 PIO 程序使用 PIO 针脚映射资源的方式，一个接口可以在一定数量的 GPIO 范围内自由移动。

## 3.2. PIO程序

四个状态机执行共享指令内存的程序。系统软件将内存加载至该区域，配置状态机和 IO 映射，然后将状态机设置为运行状态。PIO 程序可以来自多个地方：可以由用户直接汇编，可以来自 PIO 库，或者由用户的软件生成。

之后，状态机就会自动运行，系统软件通过 DMA、中断和控制寄存器与之交互，就像操作 RP2040 上的其他外设一样。对于更复杂的接口，PIO 提供了一个短小灵活的指令集，可以让系统软件更深入地操作状态机的控制流。

| ![图39](figures/figure-39.png) |
|:--:|
| 图39. 状态机概览。数据通过一对 FIFO 输入输出。状态机可以执行一段程序，让这些 FIFO、<br>一系列内部寄存器和管脚之间传输数据。时钟分频器可以降低状态机的执行速度。 |

### 3.2.1. PIO程序

PIO 状态机执行短小的二进制程序。

PIO 库提供了像 UART、SPI 或 I2C 等通用接口的程序，因此许多情况下不需要自己编写 PIO 程序。但是，直接对 PIO 编程可以带来更大的灵活性，支持许多连设计者都没有考虑供的接口。

PIO 一共有九条指令：`JMP`、`WAIT`、`IN`、`OUT`、`PUSH`、`PULL`、`MOV`、`IRQ` 和 `SET`。关于每条指令的详细信息请参见[3.4节](#34-指令集)。

尽管 PIO 只有九条指令，手工编写二进制 PIO 程序也非常困难。PIO 汇编是用文本形式描述的 PIO 程序，每条命令对应于二进制程序中的一条指令。
下面是一个 PIO 汇编程序的例子：

<caption>Pico 示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.pio 第7-12行 </caption>

```
 7 .program squarewave
 8     set pindirs, 1 ; Set pin to output
 9 again:
10     set pins, 1 [1] ; Drive pin high and then delay for one cycle
11     set pins, 0 ; Drive pin low
12     jmp again ; Set PC to label `again`
```

SDK 自带了 PIO 汇编器，名为 `pioasm`。该程序接受一个 PIO 汇编程序的文本文件，其中可以包含多个程序，并输出汇编后的程序。这些汇编后的程序以 C 头文件的形式输出，头文件中包含了常量数组。更多信息请参考[3.3节](#33-pio-汇编器pioasm)。

### 3.2.2. 控制流

在每个时钟周期，每个状态机获取、解码并执行一条指令。每个指令精确地占用一个时钟周期，除非它显式地暂停执行（如 `WAIT` 指令）。每条指令还可以带有最多 31 个周期的延时，推迟下一条指令的执行，用于编写精确时钟周期的程序。

程序计数器 `PC` 指向当前周期正在执行的指令的内存地址。一般而言，`PC` 每个周期加一，到达指令内存边界时自动返回开头。跳转指令是一个例外，它显式提供了 `PC` 的下一个值。

上一节的示例汇编程序（开头为 `.program squarewave`）演示了这两个概念。它在一个 GPIO 引脚上产生占空比为 50% 的方波，每个周期占用四个时钟周期。通过其他手段（如 side-set）可以将周期缩短至两个时钟周期。

**注意**：Side-set 可以让状态机在执行一条指令时，顺便设置少量 GPIO 的状态。详细描述请参见[3.5.1节](#351-side-set)。

系统对指令内存拥有只写的权限，用于加载程序：

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 第34-38行</caption>

```c
34     // Load the assembled program directly into the PIO's instruction memory.
35     // Each PIO instance has a 32-slot instruction memory, which all 4 state
36     // machines can see. The system has write-only access.
37     for (int i = 0; i < count_of(squarewave_program_instructions); ++i)
38         pio->instr_mem[i] = squarewave_program_instructions[i];
```

时钟分频器可以按照固定的比例降低状态机的执行速度，该比例用一个 16.8 的定点分数表示。在上述示例中，如果采用的时钟分割因子为 2.5，那么方波的周期就是 4 x 2.5 = 10 个时钟周期。这在需要设置 UART 等串行接口的精确波特率时非常有用。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 第42-47行</caption>

```c
42     // Configure state machine 0 to run at sysclk/2.5. The state machines can
43     // run as fast as one instruction per clock cycle, but we can scale their
44     // speed down uniformly to meet some precise frequency target, e.g. for a
45     // UART baud rate. This register has 16 integer divisor bits and 8
46     // fractional divisor bits.
47     pio->sm[0].clkdiv = (uint32_t) (2.5f * (1 << 16));
```

上述代码片段所在的整个程序可以在 GPIO 0 （或任何管脚）上产生一个 12.5MHz 的方波。我们还可以使用 `WAIT PIN` 指令根据管脚状态等待一定时间，或使用 `JMP PIN` 指令根据管脚状态跳转，实现根据管脚状态改变控制流。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 第51-59行 </caption>

```c
51     // There are five pin mapping groups (out, in, set, side-set, jmp pin)
52     // which are used by different instructions or in different circumstances.
53     // Here we're just using SET instructions. Configure state machine 0 SETs
54     // to affect GPIO 0 only; then configure GPIO0 to be controlled by PIO0,
55     // as opposed to e.g. the processors.
56     pio->sm[0].pinctrl =
57         (1 << PIO_SM0_PINCTRL_SET_COUNT_LSB) |
58         (0 << PIO_SM0_PINCTRL_SET_BASE_LSB);
59     gpio_set_function(0, GPIO_FUNC_PIO0);

```

系统可以通过 CTRL 寄存器在任意时间启动或停止任意状态机。多个状态机可以同时启动，而 PIO 的确定性能保证它们之间的完美同步。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 第63-67行</caption>

```c
63     // Set the state machine running. The PIO CTRL register is global within a
64     // PIO instance, so you can start/stop multiple state machines
65     // simultaneously. We're using the register's hardware atomic set alias to
66     // make one bit high without doing a read-modify-write on the register.
67     hw_set_bits(&pio->ctrl, 1 << (PIO_CTRL_SM_ENABLE_LSB + 0));
```

大多数指令都来自指令内存，但指令也可以来自其他地方，并且可以混合使用：

- 写入特殊配置寄存器（`SMx INSTR`）的指令会立即执行，中断其他指令的执行。例如，写入 `SMx INSTR` 的一条 `JMP` 指令会让状态机立即从另一个位置开始执行。
- 使用 `MOV EXEC` 指令，可以从寄存器执行指令。
- 使用 `OUT EXEC` 指令，可以从输出移位寄存器中执行指令。

后面几种方式非常灵活：指令可以嵌入到传递给 FIFO 的数据流中。I2C 的示例就在正常的数据中嵌入了 `STOP` 和 `RESTART` 等的条件。对于 `MOV` 和 `OUT EXEC` 来说，`MOV` / `OUT` 本身需要一个时钟周期，然后再执行指定的指令。

### 3.2.3. 寄存器

每个状态机都拥有几个内部寄存器。这些寄存器用于保存输入或输出数据，以及循环变量等临时数据。

#### 3.2.3.1. 输出移位寄存器（OSR）

| ![图40](figures/figure-40.png) |
|:--:|
| 图40. 输出移位寄存器（OSR）。数据每次可以并行输出 1 ~ 32 比特，未使用的数据由一个双方向移位器负责回收。<br>当 OSR 寄存器为空时，它会从 TX FIFO 中加载数据。

输出移位寄存器（OSR）负责存储在 TX FIFO 和管脚（或其他目的地，如可擦写寄存器）之间移位输出数据。

- `PULL` 指令：从 TX FIFO 中移除一个 32 位字并置于 OSR 中。
- `OUT` 指令将 OSR 中的数据移位至其他目的地，一次可移位 1 ~ 32 比特。
- 当数据被移位出后，OSR 会填充为零。
- 如果启用自动加载（autopull），那么在达到某个移位阈值时，状态机会在执行 `OUT` 指令时，自动从 FIFO 加载数据到 OSR。
- 移位方向可以是左或右，由处理器通过配置寄存器进行设置。

例如，以每两个时钟周期一个字节的速率，将数据通过 FIFO 传输到管脚：

```
1 .program pull_example1
2 loop:
3     out pins, 8
4 public entry_point:
5     pull
6     out pins, 8 [1]
7     out pins, 8 [1]
8     out pins, 8
9     jmp loop

```

在绝大部分情况下，可以通过自动加载（autopull，参见[3.5.4节](#354-自动推出和自动加载)）功能，当状态机试图在空的 OSR 上执行 `OUT` 指令时，由硬件自动填充 OSR。这样做有两个好处：

- 节省一条从 FIFO 加载数据的指令
- 实现更高的吞吐量，只要 FIFO 有数据，就能以每时钟周期最高 32 比特的数据输出

配置好自动加载后，上述程序可以简化如下，其行为完全相同：

```
1 .program pull_example2
2 
3 loop:
4     out pins, 8
5 public entry_point:
6     jmp loop
```

通过程序折返功能（program wrapping，参见[3.5.2节](#352-程序折返)）还可以进一步简化程序，实现每个系统时钟周期输出 1 字节。

```
1 .program pull_example3
2 
3 public entry_point:
4 .wrap_target
5     out pins, 8 [1]
6 .wrap
```

#### 3.2.3.2. 输入移位寄存器（ISR）

| ![图41](figures/figure-41.png) |
|:--:|
| 图41. 输入移位寄存器（ISR）。数据每次进入 1 ~ 32 比特，当前内容会左移或右移，以腾出空间。<br>当寄存器满时，内容会被写入 RX FIFO。 |

- `IN` 指令每次将 1 ~ 32 比特的数据移位进寄存器。
- `PUSH` 指令将 ISR 的内容写入 RX FIFO。
- 执行推出（push）时，ISR 的内容被清零。
- 如果设置了自动推出（autopush），那么当达到某个移位阈值时，状态机会在执行 `IN` 指令时自动将 ISR 的内容推出。
- 移位方向可以由处理器通过配置寄存器进行设置。

#### 3.2.3.3. 移位计数器

状态机会记录总共有多少比特通过 `OUT` 指令移出了 OSR 寄存器，以及通过 `IN` 指令移入了 ISR。这个信息由一对硬件计数器负责跟踪，即输出移位计数器和输入移位计数器，每个计数器可以保存数字 0 到 32（包含 0 和 32）。在每次移位操作后，相应的计数器会增加移位数量，最多增加 32（等于移位寄存器的宽度）。可以对状态机进行配置，在某个计数器达到一定阈值后自动执行下列某个动作：

- 当特定数量的比特被移出后，自动加载 OSR。参见[3.5.4节](#354-自动推出和自动加载)
- 当特定数量的比特被移入后，自动清空 ISR。参见[3.5.4节](#354-自动推出和自动加载)
- `PUSH` 或 `PULL` 指令根据输入或输出移位计数器的条件自动执行

在 PIO 复位时，或在 `CTRL_SM_RESTART` 时，输入一位计数器被清为 0（表示没有任何比特被移入），输出移位寄存器被初始化为 32（表示全空，没有任何等待移出的比特）。影响移位计数器的其他指令包括：

- 成功的 `PULL` 会将输出移位计数器清为 0
- 成功的 `PUSH` 会将输入移位计数器清为 0
- `MOV OSR, ...` （即任何写入 `OSR` 的 `MOV` 指令）会将输出移位计数器清为 0
- `MOV ISR, ...` （即任何写入 `ISR` 的 `MOV` 指令）会将输入移位计数器清为 0
- `OUT ISR, count` 将输入移位计数器设置为 `count`

#### 3.2.3.4. 可擦写寄存器

每个状态机有两个 32 位内部可擦写计数器（scratch registor），名为 `X` 和 `Y`。

它们可以用于：

- `IN`/`OUT`/`SET`/`MOV`指令的源或目的地
- 分支条件的源

例如，假设我们要为比特 "1" 产生一个长脉冲，为比特 "0" 产生一个短脉冲：

```
 1 .program ws2812_led
 2 
 3 public entry_point:
 4     pull
 5     set x, 23       ; Loop over 24 bits
 6 bitloop:
 7     set pins, 1     ; Drive pin high
 8     out y, 1 [5]    ; Shift 1 bit out, and write it to y
 9     jmp !y skip     ; Skip the extra delay if the bit was 0
10     nop [5]
11 skip:
12     set pins, 0 [5]
13     jmp x-- bitloop ; Jump if x nonzero, and decrement x
14     jmp entry_point
```

这里 `X` 是循环计数器，`Y` 是临时变量，根据来自 OSR 的一个比特进行分支。该程序可以用来驱动 WS2812 LED 接口，不过该程序还可以写得更紧凑（最少只需要三条指令）。

通过 `MOV` 指令，可擦写寄存器可以用来保存或恢复移位寄存器，可以用于重复移出同样的序列等情况。

**注意**：更紧凑的 WS2812 示例（共四条指令）参见[3.6.2节](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)。

#### 3.2.3.5. FIFO

每个状态机都拥有一对 4 字深的 FIFO，一个用于将数据从系统传输到状态机（TX），另一个用于将数据从状态机传输至系统（RX）。TX FIFO 由总线管理者（如处理器或 DMA 控制器）负责写入，而 RX FIFO 由状态机写入。FIFO 解耦合了 PIO 状态机和系统总线之间的时序，让状态机在没有处理器介入的情况下工作更长时间。

FIFO 还会生成数据请求信号（DREQ），系统的 DMA 控制器可以借助此信号，根据 RX FIFO 中的数据情况，或 TX FIFO 中的空闲空间情况，采取适当的读写节奏。因此，处理器可以设置更长的事务，比如允许在没有处理器介入的情况下传输几 K 字节的数据。

通常，一个状态机只需要单方向传递数据。此时，`SHIFTCTRL_FJOIN` 选项可以将两个 FIFO合并成一个 8 字深的单向 FIFO。这个特性可以用于高带宽的接口，如 DPI。

### 3.2.4. 等待状态

状态机可能由于多种原因暂停执行：

- `WAIT` 指令的条件未满足
- 阻塞的 `PULL` 指令遇到 TX FIFO 为空的情况，或阻塞的 `PUSH` 指令遇到 RX FIFO 为满的情况
- `IRQ WAIT` 指令设置了 IRQ 标志，等待其清空
- `OUT` 指令遇到启用了自动加载，OSR 达到移位阈值，且 TX FIFO 为空的情况
- `IN` 指令遇到启用了自动推出，ISR 达到以为预制，且 RX FIFO 为满的情况

此时，程序计数器不会增加，状态机会在下一个周期继续执行当前指令。如果指令带有一定数量的延时，那么在暂停执行状态解除之前，延时不会被执行。

**注意**：Side-set（参见[3.5.1节](#351-side-set)）不受暂停执行的影响，永远发生在所属指令的第一个时钟周期。

### 3.2.5. 管脚映射

PIO 可以控制最多 32 个 GPIO 的输出电平和方向，以及观察它们的输入电平。在每个系统时钟周期，每个状态机可以进行零个、一个或两个以下操作：

- 通过一条 `OUT` 或 `SET` 指令改变某些 GPIO 的电平或方向，或通过一条 `IN` 指令读取某些 GPIO
- 通过一个 side-set 操作改变某些 GPIO 的电平或方向

每个操作都可以指定连续的一段 GPIO 管脚，管脚数量和起始管脚由每个状态机的 `PINCTRL` 寄存器控制。共可以设定四个范围，分别用于 `OUT`、`SET`、`IN` 和 side-set 操作。每个范围可以覆盖给定 PIO 块内的任意数量的 GPIO （在 RP2040 上为 30 个用户定义 GPIO），而且这些范围之间可以重叠。

对于每次 GPIO 输出（电平和方向分别考虑），PIO 会考虑该时钟周期内发生的所有 8 个写入操作，并从编号最大的状态机开始应用写入操作。如果同一个状态机在同一个 GPIO 上同时执行 `SET`/`OUT` 和 side-set 操作，则采用 side-set。如果没有任何状态机写入某个 GPIO 输出，则其值保持前一个周期的值不变。

一般而言，每个状态机的输出被映射到不同的 GPIO 组上，从而实现某种并行接口。

### 3.2.6 IRQ 标志

IRQ 标志是可以由状态机或系统进行设置或清除的状态位。共有 8 个标志，全部对所有状态机可见，而且还可以通过 `IRQ0_INTE` 和 `IRQ1_INTE` 控制寄存器，将 IRQ 标志的低 4 比特遮盖成某个 PIO 的中断请求线，

IRQ 标志的用途主要有两个：

- 在状态机程序中，判断系统级别的中断，然后据此等待某个中断的应答
- 同步两个状态机的执行

状态机可以通过 `IRQ` 和 `WAIT` 指令来处理 IRQ 标志。

### 3.2.7. 状态机之间的交互

指令内存是一个 1-写 4-读的寄存器文件，所以四个状态机可以在同一个周期读取指令，而不需要相互等待。

使用多个状态机有三种方式：

- 将多个状态机指向同一个程序
- 将多个状态机指向不同的程序
- 用多个状态机运行同一个接口的不同部分，例如 UART 的 TX 端和 RX 端，或 DPI 显示的时钟/水平同步和像素数据

状态机之间无法进行数据通信，但可以通过 IRQ 标志互相同步。共有 8 个 IRQ 标志（低 4 比特可以遮盖，用于系统 IRQ），每个状态机可以通过 `IRQ` 指令设置或清除任意标志，也可以通过 `WAIT IRQ` 指令等待某个标志被设置或清除。这样可以实现状态机之间的时钟周期级别的同步。

## 3.3. PIO 汇编器（pioasm）

PIO 汇编器能够解析一段 PIO 源文件，并输出汇编后的代码，该代码可以包含到某个 RP2040 应用程序中，可以是使用 SDK 构建的 C 或 C++ 程序，也可以是 RP2040 MicroPython 上运行的 Python 程序。

本节简要介绍 `pioasm` 中可以使用的标识符（directives）和指令（instructions）。有关怎样使用 `pioasm`、怎样将其集成到 SDK 构建系统中、怎样扩展其特性以及它能生成何种输出格式等的深入讨论，请参考 [Raspberry Pi Pico C/C++ SDK](https://datasheets.raspberrypi.org/pico/raspberry-pi-pico-c-sdk.pdf) 一书。

### 3.3.1. 标识符

PIO 程序中的汇编语言可以使用以下标识符：
```
.define ( PUBLIC ) <symbol> <value>
```
定义一个名为 &lt;symbol&gt; 的整数符号，其值为 &lt;value&gt; （参见[3.3.2节](#332-值)）。如果 .define 出现在输入文件中的第一个程序之前，那么定义就是全局的，对所有程序生效；否则，定义就是局部的，仅对其所在的程序生效。如果指定了 PUBLIC，那么符号将输出到汇编中，可以由用户代码使用。对于 SDK 而言，程序符号的定义形式为 `#define <program_name>_<symbol> value`，而全局符号的定义形式为`#define <symbol> value`。

```
.program <name>
```
开始一个名为 &lt;name&gt; 的新程序。注意名称会在代码中使用，所以应当由字母、数字或下划线组成，并且不以数字开始。程序直到出现下一个 .program 标识或源文件末尾时结束。PIO 指令只能在程序内使用。

```
.clock_div <divider>
```
如果存在该指令，则divider是程序的状态机时钟分频器。需要注意的是，分频值可以是浮点数，但目前无法使用算数表达式来定义。该指令影响程序的默认状态机配置，此指令仅在第一条程序指令之前有效。

```
.pio_version <version>
```
如果存在该指令，则要求硬件的PIO版本至少需要达到指定的值。rp2040的PIO版本为0，RP2350的PIO版本为1。

```
.fifo <fifo_config>
```
如果存在该指令，则它用来指定程序的FIFO配置。支持以下FIFO配置：
 - txrx: TX和RX的FIFO各自分配4字，这也是默认配置
 - tx: TX FIFO独占8字
 - rx: RX FIFO独占8字
 - txput: 4字FIFO用于TX，4字FIFO用于 mov rxfifo[index], isr (该配置不支持PIO version 0)
 - txget: 4字FIFO用于RX，4字FIFO用于 mov osr, rxfifo[index] (该配置不支持PIO version 0)
 - putget: 4字FIFO用于 mov rxfifo[index], isr  4字FIFO用于 mov osr, rxfifo[index] (该配置不支持PIO version 0)

```
.in <count> (left|right) (auto) (<threshold>)
```
如果存在该指令，count 指示要使用的bit数，left和right指示ISR寄存器位移的方向，auto指示使能自动push，如果存在threshold，则其指示执行自动push的阈值。该指令影响程序的默认状态机配置，此指令仅在第一条程序指令之前有效。(对于PIO version 0来说，count固定为32)

```
.out <count> (left|right) (auto) (<threshold>)
```
如果存在该指令，count 指示要使用的bit数，left和right指示OSR寄存器位移的方向，auto指示使能自动pull，如果存在threshold，则其指示执行自动pull的阈值。该指令影响程序的默认状态机配置，此指令仅在第一条程序指令之前有效。(对于PIO version 0来说，count固定为32)

```
.mov_status rxfifo < <n>
.mov_status txfifo < <n>
.mov_status irq <(next|prev)> set <n>
```
该指令用于配置指令 mov x, status 的源。可从三种情况之中选择一种进行配置，包括根据RXFIFO数量低于n、TXFIFO数量低于n、irq的第n个标志被设置(或下一个更高编号或上一个更低编号的PIO)。irq选项不支持PIO version 0。

```
.origin <offset>
```
可选的标识，指示 PIO 指令内存的偏移量，程序必须加载到此偏移量处。绝大多数情况下，该标识用于指定程序必须加载到偏移量 0，这样这些程序才能使用基于数据的 JMP 指令，并且只用几个比特来存储跳转的（绝对）地址。在程序外部，该标识无效。

```
.side_set <count> (opt) (pindirs)
```
如果出现该标识，则 &lt;count&gt; 指示使用的 side-set 比特数。还可以通过 opt 以指定 side &lt;value&gt; 对于指令是可选的（注意这样做会使 side-set 在原本需要占用的 &lt;count&gt; 个比特之外，再额外占用一个比特。这些被占用的比特均来自指令的延时比特）。最后，pindirs 可以用来指示 side-set 值应当应用于 PINDIR，而不是 PIN 上。该标识只能出现在程序的第一条指令之前。

```
.wrap_target
```
该标识置于某条指令之前，用于指示在程序折返时应当从哪一条指令继续执行。该标识只能在程序内部使用，且每个程序只能使用一次。如果不指定，折返目的地为程序的开头。

```
.wrap
```
该标识置于某条指令之后，指示在正常的控制流中（即 jmp 条件为假，或没有 jmp 的情况）执行完该指令后，程序应当折返（至 .warp_target 标识的指令）。该标识只能在程序内部使用，且每个程序只能使用一次。如果不指定，则折返点为程序的最后一条指令之后。

```
.lang_opt <lang> <name> <option>
```
为程序指定与特定语言生成器有关的选项（参见[SDK文档](https://datasheets.raspberrypi.org/pico/raspberry-pi-pico-c-sdk.pdf)中的"Language generators"一节）。该标识只能在程序内使用。

```
.word <value>
```
将一个 16 位值作为一条指令存储在程序中。该标识只能在程序内部使用。

### 3.3.2. 值

下述值类型可以用于定义整数，或分支目的地。

| 格式 | 说明 |
|-----|------|
| integer | 一个整数值，如 3 或 -7 |
| hex     | 一个十六进制值，如 0xf |
| binary  | 一个二进制值，如 0b1001 |
| symbol  | 由 .define 定义的一个值（参见[3.3.1节](#331-标识符)） |
| &lt;label&gt; | 程序中的标签所对应的指令偏移量。通常在 JMP 指令中使用（参见[3.4.2节](#342-jmp) |
| ( &lt;expression&gt; ) | 一个可求值的表达式；参见[3.3.3节](#333-表达式)。注意括号是必须的。 |

### 3.3.3. 表达式

表达式可以与 pioasm 值一同使用。

| 格式 | 说明 |
|-----|------|
| `<expression> + <expression>` | 两个表达式的和 |
| `<expression> - <expression>` | 两个表达式的差 |
| `<expression> * <expression>` | 两个表达式的积 |
| `<expression> / <expression>` | 两个表达式的整数除法商 |
| `- <expression>` | 表达式的负值 |
| `:: <expression>` | 表达式的按位取反 |
| `<value>` | 任意的值（参见[3.3.2节](#332-值) |

### 3.3.4. 注释

行注释以 `//` 或 `;` 开头。

C 语言风格的注释放在 `/*` 和 `*/` 之间。

### 3.3.5. 标签

标签的形式如下：

`<symbol>:`

或

`PUBLIC <symbol>:`

标签必须从行首开始。

**提示**：标签实际上只是自动的 `.define`，其值设置为当前程序指令的偏移量。`PUBLIC` 的标签可以通过与 `PUBLIC` 的 `.define` 同样的方式从用户代码访问。

### 3.3.6. 指令

所有的 pioasm 指令都遵循以下格式：

`<instruction> (side <side_set_value>) ([<delay_value])`

其中：

`<instruction>` 是下一节介绍的汇编指令。（参见[3.4节](#34-指令集)）

`<side_set_valie>` 是一个值（参见[3.3.2节](#332-值)），在指令开始时，该值将应用到 side_set 管脚。注意 `side <side_set_value>` 中的 side-set 值的规则依赖于该程序的 `.side_set` 指示（参见[3.3.1节](#331-标识符)）。

如果没有指定 `.side_set`，则 `side <side_set_value>` 就是无效的。如果指定了可选数量的 side-set 管脚，则允许使用 `side <side_set_value>`。如果指定了必须数量的 side-set 管脚，则 `side <side_set_value>` 是必须的。

`<side_set_value>` 的值必须匹配 `.side_set` 标识中指定的 side-set 的比特数。

`<delay_value>` 指定在指令执行完成后需要延时的时钟周期数。`delay_value` 是一个值（参见[3.3.2节](#332-值)），通常在 0 ~ 31 之间（包含 0 和 31，是一个 5 比特的值），但是如果通过 `.side_set` 指示（参见[3.3.1节](#331-标识符)）启用了 side-set，那么可用的比特数就会减少。如果不指定 `<delay_value>`，则指令没有延时。


**注意** pioasm 指令名称、关键字和标识不区分大小写。遵循 SDK 的风格，下面的“汇编语法”一节统一使用小写。

**注意** 某些汇编语法中会出现都好，但逗号完全是可选的。例如 `out pins, 3` 可以写成 `out pins 3`，`jmp x-- label` 可以写成 `jmp x--, label`。“汇编语法”一节使用前一种格式。

### 3.3.7 伪指令

目前，pioasm 提供一条伪指令（pseudoinstruction），以方便编程：

`nop`：汇编成 `mov y, y`。表示“无操作”。没有任何副作用，用于 side-set 操作，或额外的延时。

## 3.4. 指令集
### 3.4.1. 概要
PIO 指令为 16 位，编码方式如下：

![表376](figures/table-376.png)

所有 PIO 指令的执行时间都是一个时钟周期。

5 比特的 `Delay/side-set` 字段的含义依赖于状态机的 `SIDESET_COUNT` 配置：

- 最多 5 个 LSB 比特（5 减 `SIDESET_COUNT`）为当前指令和下一条指令之间插入的空闲周期数。
- 最多 5 个 MSB 比特（由 `SIDESET_COUNT` 设置）为 side-set（参见[3.5.1节](#351-side-set)），可以在该指令执行的同时，将某些 GPIO 管脚设置为某个常量。

### 3.4.2. JMP

#### 3.4.2.1. 编码

![JMP](figures/instruction-jmp.png)

#### 3.4.2.2. 操作

如果 `Condition` 为真，则将程序计数器设置为 `Address`，否则无操作。

`JMP` 上的延时周期不论 `Condition` 是否为真都会生效。延时在 `Condition` 被求值、程序计数器被更新后进行。

- Condition：
  - 000：（无条件）：永远跳转
  - 001：`!X`：当寄存器 X 为零时
  - 010：`X--`：当 X 非零时。判断后进行减一
  - 011：`!Y`：当寄存器 Y 为零时
  - 100：`Y--`：当 Y 非零时。判断后进行减一
  - 101：`X!=Y`：当 X 不等于 Y 时
  - 110：`PIN`：根据输入管脚跳转
  - 111：`!OSRE`：当输出移位寄存器非空时
- Address：要跳转到的指令地址。在指令编码中，该值为 PIO 指令内存中的绝对地址。

`JMP PIN` 会根据 `EXECCTRL_JMP_PIN` 选择的 GPIO 管脚进行跳转。`EXECCTRL_JMP_PIN` 是一个可配置选项，它从状态机可以使用的最多 32 个 GPIO 输入管脚中选择其中之一，供 JMP 使用。不依赖于状态机的其他输入映射。如果该 GPIO 为高电平，则跳转。

`!OSRE` 将自上一次 `PULL` 以来移出的比特数，与移位计数阈值进行比较。移位计数阈值由 `SHIFTCTRL_PULL_THRESH` 配置。自动加载（autopull，参见[3.5.4节](#354-自动推出和自动加载)）也通过该配置项配置。

`JMP X--` 和 `JMP Y--` 总是会将寄存器 X 或 Y 减一。减一操作与可擦写寄存器的当前值无关。跳转条件是寄存器的*初始值*，即减一操作发生之前的值。如果寄存器最初为非零值，则发生跳转。

#### 3.4.2.3. 汇编语法

`jmp (<cond>) <target>`

其中：

`<cond>` 是上节列出的可选的条件（例如，`!x` 表示可擦写寄存器 X 为零）。如果未指定条件，则总是跳转。

`<target>` 是一个程序标签或值（参见[3.3.2节](#332-值)），表示程序内部的指令偏移量（第一条指令的偏移量为 0）。注意，由于 PIO JMP 指令使用 PIO 指令内存之内的绝对地址，JMP 需要在运行时根据程序的加载偏移量进行调整。这一步在加载程序时由 SDK 负责，但在编码 JMP 指令供 `OUT EXEC` 使用时需要留意这一点。

### 3.4.3. WAIT

#### 3.4.3.1. 编码

![WAIT](figures/instruction-wait.png)

#### 3.4.3.2. 操作

等待，直到条件满足。

与所有等待指令一样（参见[3.2.4节](#324-等待状态)），延时周期在指令**完成**之后发生。也就是说，如果指定了延时周期，那么延时要直到等待条件满足**之后**才会开始。

- Polarity
  - 1：等待 1。
  - 0：等待 0。
- Source：指定要等待什么。可能的值有：
  - 00:`GPIO`：等待由 `Index` 选择的系统 GPIO 输入。该项为绝对 GPIO 索引，不受状态机的输入 IO 映射影响。
  - 01：`PIN`：由 `Index` 指定的输入管脚。首先会应用当前状态机的输入 IO 映射，然后利用 `Index` 选择要等待哪个输入比特。换句话说，选择的管脚就是 `PINCTRL_IN_BASE` 加上 `Index`，再对 32 取余。
  - 10：`IRQ`：等待由 `Index` 选择的 PIO IRQ 标志。
  - 11：`JMPPIN`：等待由PINCTRL_JMP_PIN配置的GPIO，可以设置一个0-3的偏移。加上偏移后不得超过32。(该模式不支持PIO version 0)
- Index：要检查哪个管脚或比特。

`WAIT x IRQ` 的行为与其他 `WAIT` 源略有不同：

- 如果 `Polarity` 为 1，则在等待条件满足后，当前状态机会清除选择的 IRQ 标志。
- 标志位索引的解码方式与 `IRQ` 的索引字段相同，从两个MSB进行解码：
  - 00: 最低三位用于要等待的当前PIO的IRQ标志编号。
  - 01(PREV): 最低三位指示的IRQ编号为上一个PIO的IRQ标志。
  - 10(REL):  最低三位的值加上状态机的编号再模4作为IRQ的编号进行等待。例如状态机2的最低3位的值为1，则将等待IRQ3。状态机3的最低三位为3，则将等待IRQ2。这样，允许同一个程序的不同状态机就可以相互同步。
  - 11(NEXT): 最低三位指示的IRQ编号为上一个PIO的IRQ标志。
  如果 MSB 被设置，则将状态机的 ID（0..3）加到 IRQ 索引上，加法采用两个 LSB 的模 4 加法。例如，状态机 2、标志值为 '0x11'，则等待标志 3，而标志值 '0x13' 将等待标志 1。这样，运行同一个程序的多个状态机就可以互相同步。

**注意** `WAIT 1 IRQ x` 不应当使用供中断控制器使用的 IRQ 标志，以避免与系统中断处理函数的竞合冲突。

#### 3.4.3.3. 汇编语法

`wait <polarity> gpio <gpio_num>`

`wait <polarity> pin <pin_num>`

`wait <polarity> irq <irq_num> (rel, next, prev)`

`wait <polarity> jmppin (+ <pin_offset>)`

其中：

`<polarity>` 是一个值（参见[3.3.2节](#332-值)），指定极性（0 或 1）

`<pin_num>` 是一个值（参见[3.3.2节](#332-值)），指定输入管脚的编号（在状态机输入管教映射中的编号）

`<gpio_num>` 是一个值（参见[3.3.2节](#332-值)），指定实际的 GPIO 管脚编号

`<irq_num> (rel)` 是一个值（参见[3.3.2节](#332-值)），指定要等待的 IRQ 编号（0 ~ 7）。如果设置了 `rel`，则实际的 IRQ 编号的计算方式为，将 IRQ 编号的最低两个比特（`irq_num`<sub>10</sub>）替换成和的最低两个比特（`irq_num`<sub>10</sub> + `sm_num`<sub>10</sub>），其中 `sm_num`<sub>10</sub> 为状态机编号

`<pin_offset>` 是一个值，用来添加到JMP_PIN上来获得实际需要等待的引脚编号

### 3.4.4. IN

#### 3.4.4.1. 编码

![IN](figures/instruction-in.png)

#### 3.4.4.2. 操作

从 `Source` 移位 `Bit count` 个比特到输入移位寄存器（ISR）中。移位方向由各个状态机的 `SHIFTCTRL_IN_SHIFTDIR` 决定。此外，增加输入移位计数器 `Bit count`，最大到 32。

- Source：
  - 000：`PINS`
  - 001：`X`（可擦写寄存器 X）
  - 010：`Y`（可擦写寄存器 Y）
  - 011：`NULL`（全零）
  - 100：保留
  - 101：保留
  - 110：`ISR`
  - 111：`OSR`
- Bit count：要将多少比特移位到 ISR。值为 1 ~ 32 比特，32 编码为 `00000`。

如果启用了自动推出（autopush），那么当达到推出阈值（`SHIFTCTRL_PUSH_THRES`）时，`IN` 还会将 ISR 的内容推出到 RX FIFO。不论自动推出是否发生，`IN` 的执行时间都是一个时钟周期。当自动推出发生，但 RX FIFO 满时，状态机会进入等待状态。自动推出会清空 ISR 的内容为全零，并且会清除输入移位计数器。参见[3.5.4节](#354-自动推出和自动加载)。

`IN` 永远使用源数据的最低 `Bit count` 个比特。例如，如果 `PINCTRL_IN_BASE` 设置为 5，那么指令 `IN 3, PINS` 会读取管脚 5、6 和 7，并将这些值移入 ISR。首先，ISR 会左移或右移，为新的输入数据腾出空间，然后将输入数据复制到腾出的空间中。输入数据的比特顺序与移位方向无关。

使用 `NULL` 可以移位 ISR 的内容。例如，UART 会首先接受 LSB，因此必须右移。在执行 8 次 `IN PINS, 1` 指令后，输入的串行数据占据了 ISR 的 31..24 比特。此时，`IN NULL, 24` 指令将会移入 24 个零比特，将输入数据移到 ISR 的 7..0 比特。另一种方式是让处理器或 DMA 从 FIFO 地址 +3 的位置读取一个字节，这样可以读取 FIFO 内容的 31..24 比特。

#### 3.4.4.3. 汇编语法

`in <source>, <bit_count>`

其中：

`<source>` 是上述源之一。

`<bit_count>` 是一个值（参见[3.3.2节](#332-值)，指定了要移位的比特数（有效值为 1 ~ 32）。

### 3.4.5. OUT

#### 3.4.5.1. 编码

![OUT](figures/instruction-out.png)

#### 3.4.5.2. 操作

将 `Bit count` 个比特移出输入移位寄存器（OSR），并写入 `Destination`。此外，还会增加输出移位计数器 `Bit count`，直到最大值 32。

- Destination：
  - 000：`PINS`
  - 001：`X`（可擦写寄存器 X）
  - 010：`Y`（可擦写寄存器 Y）
  - 011：`NULL`（全零）
  - 100：`PINDIRS`
  - 101：`PC`
  - 110：`ISR`（同时设置 ISR 移位计数器为 `Bit count`）
  - 111：`EXEC`（将 OSR 的移位数据作为指令执行）
- Bit count：要从 OSR 移出多少比特。值为 1..32 比特，32 编码为 `00000`。


将一个 32 位值写入 `Destination`：低 `Bit count` 个比特来自 OSR，其余为零。如果 `SHIFTCTRL_OUT_SHIFTDIR` 为右，则该值为 OSR 的最低 `Bit count` 比特，否则为最高 `Bit count` 比特。

`PINS` 和 `PINDIRS` 使用 `OUT` 的管脚映射，参见[3.5.6节](#356-gpio-映射)。

如果启用自动加载（autopull），那么如果达到了加载阈值 `SHIFTCTRL_PULL_THRESH`，则自动从 TX FIFO 填充 OSR。同时输出移位计数器清零。在这种情况下，如果 TX FIFO 为空，则 `OUT` 会进入等待状态，但执行时间仍然为一个时钟周期。详情参考[3.5.4节](#354-自动推出和自动加载)。


有了 `OUT EXEC`，指令就可以放在 FIFO 数据流中。`OUT` 本身的执行需要一个时钟周期，然后下一个周期执行来自 OSR 的指令。该机制能执行的指令类型没有限制。最初的 `OUT` 指令的延时周期将被忽略，但之后执行的指令可以正常插入延时周期。

`OUT PC` 相当于无条件跳转到 OSR 移出的值对应的地址。

#### 3.4.5.3. 汇编语法

`out <destination>, <bit_count>`

其中：

`<destination>` 是上述目的地之一。

`<bit_count>` 是一个值（参见[3.3.2节](#332-值)，指定要移位的比特数（有效值为 1 ~ 32）。

### 3.4.6. PUSH

#### 3.4.6.1. 编码

![PUSH](figures/instruction-push.png)

#### 3.4.6.2. 操作

将 ISR 的内容作为一个 32 位字推出到 RX FIFO。同时将 ISR 清除为全零。

- `IfFull`：如果设置为 1，那么在输入移位计数器未达到阈值（`SHIFTCTRL_PUSH_THRESH`，与自动推出用的是同一个选项；参见[3.5.4节](#354-自动推出和自动加载)），则什么都不做。
- `Block`：如果设置为 1，那么在 RX FIFO 为满时，暂停执行。

`PUSH IFFULL` 也能像自动推出（autopush）那样让程序更紧凑。如果启用了自动推出，会导致 `IN` 在不恰当的时候陷入等待状态，比如状态机需要在此时判断某种外部控制信号，那么 `PUSH IFFULL` 就能派上用场了。

PIO 汇编器会默认设置 `Block` 位。如果 `Block` 位没有设置，则 `PUSH` 不会在 RX FIFO 满的情况下进入等待，而是会继续立即执行下一条指令。此时，FIFO 的状态和内容不变。ISR 仍被清空为全零，同时设置 `FDEBUG_RXSTALL` 标志（与 RX FIFO 满时进行阻塞 `PUSH` 或自动推出相同），表示数据已经丢失。

#### 3.4.6.3. 汇编语法

`push (iffull)`

`push (iffull) block`

`push (iffull) nonblock`

其中：

`iffull` 相当于上述 `IfFull == 1`。也就是说，不指定的情况下，默认值为 `IfFull == 0`

`block` 相当于上述 `Block == 1`。如果不指定 `block` 或 `noblock`，则此值为默认值

`noblock` 相当于上述 `Block == 0`。

### 3.4.7. PULL

#### 3.4.7.1. 编码

![PULL](figures/instruction-pull.png)

#### 3.4.7.2. 操作

从 TX FIFO 加载一个 32 位字到 OSR 中。

- `IfEmpty`：如果为 1，那么在输出移位计数器达到阈值（`SHIFTCTRL_PULL_THRESH`，与自动加载用的是同一个选项；参见[3.5.4节](#354-自动推出和自动加载)），则什么都不做。
- `Block`：如果设置为 1，那么在 TX FIFO 为空时，暂停执行。如果为 0，那么从空的 FIFO 中加载，将会把可擦写寄存器 X 的内容复制到 OSR。

一些外设（UART、SPI等）应当在无数据时等待，有数据时进行加载；而另一些（I2S）则应当继续执行，而且最好是继续输出占位数据，或重复数据。这个行为可以通过 `Block` 参数实现。

在空的 FIFO 上执行不阻塞的 `PULL` 相当于执行 `MOV OSR, X`。程序可以事先在寄存器 X 中加载适当的默认值，或在每次 `PULL NOBLOCK` 之后执行一次 `MOV X, OSR`，从而重复上一个有效的 FIFO 字，直到新的数据出现。

如果在启用自动加载时，当 TX FIFO 为空时的 `OUT` 会导致程序在不恰当的地方等待，就能用 `PULL IFEMPTY` 解决问题。`IfEmpty` 可以像自动加载一样简化一些程序，例如可以去掉外层循环计数器，但是它能够控制暂停的发生位置。

**注意** 当启用自动加载时，在 OSR 满的情况下任何 `PULL` 指令都是误操作，所以 `PULL` 指令可以作为一种屏障。`OUT NULL, 32` 可以显式地抛弃 OSR 的内容。更多细节参见[3.5.4.2节](#354-自动推出和自动加载)。

#### 3.4.7.3. 汇编语法

`pull (ifempty)`

`pull (ifempty) block`

`pull (ifempty) noblock`

其中：

`ifempty` 相当于上述 `IfEmpty == 1`，即如果没有指定，则默认值为 `IfEmpty == 0`

`block` 相当于上述 `Block == 1`。如果不指定 `block` 或 `noblock`，则此值为默认值

`noblock` 相当于上述 `Block == 0`。

### 3.4.8. MOV

#### 3.4.8.1. 编码

![MOV](figures/instruction-mov.png)

#### 3.4.8.2. 操作

从 `Source` 复制数据到 `Destination`。

- Destination：
  - 000：`PINS`（使用与 `OUT` 相同的管脚映射）
  - 001：`X`（可擦写寄存器 X）
  - 010：`Y`（可擦写寄存器 Y）
  - 011：`PINDIRS`（使用与 `OUT` 相同的管脚映射）
  - 100：`EXEC`（将数据作为指令执行）
  - 101：`PC`
  - 110：`ISR`（该操作会重置输入移位计数器为零，意为空）
  - 111：`OSR`（该操作会重置输出移位寄存器为零，意为满）
- Operation：
  - 00：无
  - 01：求反（按位求补）
  - 10：反转比特顺序
  - 11：保留
- Source：
  - 000：`PINS`（使用与 `IN` 相同的管脚映射）
  - 001：`X`
  - 010：`Y`
  - 011：`NULL`
  - 100：保留
  - 101：`STATUS`
  - 110：`ISR`
  - 111：`OSR`

`MOV PC` 会导致无条件跳转。`MOV EXEC` 的效果和 `OUT EXEC`（参见[3.4.5节](#345-out)）相同，可以将寄存器的内容作为指令执行。`MOV` 指令自身需要一个时钟周期，`Source` 中的指令在下一个周期执行。`MOV EXEC` 上的延时周期会被忽略，但被执行的指令可以带有延时周期。

源 `STATUS` 的值为全 1 或全 0，取决于某些状态机的状态，如 FIFO 满/空等。可以通过 `EXECCTRL_STATUS_SEL` 配置。

`MOV` 可以通过有限的几种方式操作传输的数据，由 `Operation` 参数指定。求反操作会将 `Destination` 中的每个比特设置为 `Source` 中对应比特的 NOT，也就是说，1 变成 0，0 变成 1。至于反转比特顺序，如果将比特编号为 0 ~ 31 的话，该操作可以将 `Destination` 中的每个比特 n 设置为 `Source` 中的比特 31 - n。

`MOV dst, PINS` 使用IN引脚映射读取引脚的值，掩码为 `SHIFTCTRL_IN_COUNT`，读取到的最低位来自于 `PINCTRL_IN_BASE`，其余的依次高位就是对应更高位的pin，最高到达31。结果中超过 `SHIFTCTRL_IN_COUNT` 的位被设置为0。

`MOV PINDIRS, src` 不支持PIO version 0。

#### 3.4.8.3. 汇编语法

`mov <destination>, (op) <source>`

其中：

`<destination>` 是上述目的地之一。

`<op>` 如果存在的话，为下列值之一：

- `!` 或 `~` 表示 NOT （注意：永远是按位求 NOT）
- `::` 表示反转比特顺序

`<source>` 是上述源之一。

### 3.4.9. IRQ

#### 3.4.9.1. 编码

![IRQ](figures/instruction-irq.png)

#### 3.4.9.2. 操作

设置或清除由 `Index` 参数指定的 IRQ 标志。

- Clear：如果为 1，则清除 `Index` 指定的标志，否则设置标志。如果指定了 `Clear`，则忽略 `Wait` 比特。
- Wait：等待指定标志再次变成低，即，等待某个系统中断处理函数响应该标志。
- Index：最低三个bit指定来IRQ的索引，值为0 ～ 7。
- Index Mode：Index的高2为编号偏移模式，有4种情况：
  - 00: 默认值，最低三位的索引用于当前PIO的IRQ标志。
  - 01(PREV): 索引值用于前一个PIO的对应IRQ标志。
  - 10(REL): 最低三位值加上状态机编号再模4的值用于当前PIO的IRQ标志。
  - 11(NEXT): 索引值用于下一个PIO的对应IRQ标志。

IRQ 标志 4 ~ 7 只能被状态机使用，而 IRQ 标志 0 ~ 3 可以作为系统级别的中断，在 PIO 的两个外部中断请求线之一上使用，具体采用哪个外部中断请求线，由 `IRQ0_INTE` 和 `IRQ1_INTE` 配置。

NEXT/PREV模式可以再不同PIO之间进行同步，如果这些待同步的PIO状态机设置了时钟分频，则其分频配置必须相同，并且需要通过写入CTRL.NEXTPREV_CLKDIV_RESTART及相关的NEXT_PIO_MASK/PREV_PIO_MASK位来同步时钟。需要注意的是，跨PIO同步在非安全代码中具有不同可访问性的PIO之间将被阻断。（NEXT/PREEV模式不适用PIO version 0）

模 4 加法可以实现 IRQ 和 WAIT 指令的“相对寻址”，用于运行同一个程序的不同状态机之间的同步。

如果设置了 `Wait`，则 `Delay` 周期会等到等待期间结束后再开始。

#### 3.4.9.3. 汇编语法

`irq <irq_num> (prev, rel, next)`

`irq set <irq_num> (prev, rel, next)`

`irq nowait <irq_num> (prev, rel, next)`

`irq wait <irq_num> (prev, rel, next)`

`irq clear <irq_num> (prev, rel, next)`

其中：

`<irq_num> (rel)` 是一个值（参见[3.3.2节](#332-值)），指定要等待的 IRQ 编号（0 ~ 7）。如果存在 `rel`，则实际的 IRQ 编号的计算方式为，将 IRQ 编号（`irq_num`<sub>10</sub>）的最低两比特替换为（`<irq_num`<sub>10</sub> + `sm_num`<sub>10</sub>）的最低两比特，其中 `sm_num`<sub>10</sub> 为状态机编号

`irq` 表示设置 IRQ，无需等待

`irq set` 也表示设置 IRQ，无需等待

`irq nowait` 还是表示设置 IRQ，无需等待

`irq wait` 表示设置 IRQ，然后等待 IRQ 被清除后再继续

`irq clear` 表示清除 IRQ

### 3.4.10. SET

#### 3.4.10.1. 编码

![SET](figures/instruction-set.png)

#### 3.4.10.2. 操作

将一个数 `Data` 写入到 `Destination`。

- Destination：
  - 000：`PINS`
  - 001：`X`（可擦写寄存器 X）5 个 LSB 比特写入 `Data`，其他比特清为 0。
  - 010：`Y`（可擦写寄存器 Y）5 个 LSB 比特写入 `Data`，其他比特清为 0。
  - 011：保留
  - 100：`PINDIRS`
  - 101：保留
  - 110：保留
  - 111：保留
- Data: 设置到管脚或寄存器的 5 比特的立即数。

该指令用于设置控制信号，如时钟或芯片选择，或初始化循环计数器。由于 `Data` 只有 5 比特，因此可擦写寄存器可以 `SET` 的值为 0 ~ 31，最大可循环 32 次。

`SET` 和 `OUT` 的管脚映射是独立配置的。它们可以映射到不同的位置，例如，一个管脚作为时钟信号使用，另一个作为数据使用。它们的管脚范围也可以重叠：UART 传输程序可以使用 `SET` 来设置开始和终止比特，用 `OUT` 指令向同一个管脚上移出 FIFO 数据。`SET`指令最多可以控制5个管脚的输出和方向。

#### 3.4.10.3. 汇编格式

`set <destination>, <value>`

其中：

`<destination>` 为上述目的地之一。

`<value>` 是要设置的值（参见[3.3.2节](#332-值)），有效值为 0 ~ 31。

### 3.4.11. MOV(to RX)

#### 3.4.11.1. 编码
未添加。

#### 3.4.11.2. 操作
将ISR寄存器按照选定的位置写入RXFIFO。状态机可以按照任意顺序写入RXFIFO，由Y寄存器或立即数进行索引。前提是需要对SHIFTCTRL_FJOIN_RX_PUT字段进行配置，否则该操作未定义。

**注意**，RXFIFO存储寄存器只有一个读端口和写端口，任何时候每个端口的访问都值分配给其中一个（系统或状态机）。

#### 3.4.11.3. 汇编格式

`mov rxfifo[y], isr`
`mov rxfifo[<index>], isr`

其中：

`y` 表示RXFIFO由y寄存器来索引
`<index>` 使用一个立即数对RXFIFO进行索引，有效值是0～3。

### 3.4.12. MOV(from RX)

#### 3.4.12.1. 编码
未添加。

#### 3.4.12.2. 操作
将所选的RXFIFO条目保存到OSR寄存器。PIO可以按照任意顺序读取RXFIFO，由Y寄存器或立即数进行索引。前提是需要对SHIFTCTRL_FJOIN_RX_GET字段进行配置，否则该操作未定义。

#### 3.4.12.3. 汇编格式

`mov osr, rxfifo[y]`

`mov osr, rxfifo[<index>]`

其中：

`y` 表示RXFIFO由y寄存器来索引
`<index>` 使用一个立即数对RXFIFO进行索引，有效值是0～3。


## 3.5. 功能详解

### 3.5.1. Side-set

Side-set 特性可以让状态机在执行主指令的同时，改变最多 5 个管脚的电平或方向。

需要使用 side-set 的例子之一就是高速 SPI 接口，其数据比特从 OSR 推出到 GPIO，而时钟信号转换（从 1 到 0， 或从 0 到 1）必须与此同时发生。

此时，一个带有 side-set 的 `OUT` 就能同时完成两项任务。

这样可以保证接口的时序更精确，减小整体的程序大小（因为不需要独立的 `SET` 指令设置时钟管脚），而且提高了 SPI 能支持的最大频率。

Side-set 还可以让 GPIO 映射更灵活，因为它的映射独立于 `SET` 的映射。示例 I2C 代码在禁止时钟拉伸（clock streching）的情况下，可以将 `SDA` 和 `SCL` 映射到任意管脚上。

正常情况下，SCL 反转用于同步数据传输，而 SDA 包含移出的数据比特。但是，一些特定的 I2C 序列，如 `Start` 和 `Stop` 线条件，需要在 SDA 和 SCL 上产生特定的脉冲。
I2C 通过如下映射来实现这一点：

- Side-set → SCL
- `OUT` → SDA
- `SET` → SDA

这样，状态机就可以在 SDA 上提供两种情况的数据，在 SCL 上提供时钟，或者同时在 SDA 和 SCL 上提供固定的转换，同时依然允许 SDA 和 SCL 被映射到任意两个 GPIO 上。

Side-set 数据编码到每条指令的 `Delay/side-set` 字段。任何指令都可以与 side-set 组合，包括 `OUT PINS`、`SET PINS` 等写入管脚的指令。

Side-set 的管脚映射独立于 `OUT` 和 `SET` 的映射，但它们可以重叠。如果 side-set 和 `OUT` 或 `SET` 同时写入同一个管脚，则采用 side-set 的数据。

**注意** 如果指令进入等待状态，side-set 依然会立即生效。

```
1 .program spi_tx_fast
2 .side_set 1
3 
4 loop:
5     out pins, 1   side 0
6     jmp loop      side 1
```

`spi_tx_fast` 示例展示了 side-set 的两个优势：数据和时钟可以更精确地对齐，并且程序整体上可以更快，本例中每两个系统时钟周期就能输出一比特。程序也可以更小。

使用 side-set 时需要配置四个选项：

1. `Delay/side-set` 字段用于 side-set 而不是延时的 MSB 比特数。通过 `PINCTRL_SIDESET_COUNT` 设置。如果设置为 5，则无法使用延时周期。如果设置为 0，则无法使用 side-set。
2. 是否使用最高比特作为有效位。Side-set 只有在设置了有效位的指令上生效。如果不使用有效位，则当 `SIDESET_COUNT` 不为零时，该状态机上的**所有**指令都会执行 side-set。该选项由 `EXECCTRL_SIDE_EN` 设置。
3. 映射到 side-set 最低比特的 GPIO 编号。由 `PINCTRL_SIDESET_BASE` 设置。
4. side-set 写入到 GPIO 电平还是 GPIO 方向。由 `EXECCTRL_SIDE_PINDIR` 设置。

上述示例中，我们只设置了一个 side-set 比特，每条指令都执行 side-set，因此不需要有效位。`SIDESET_COUNT` 为 1，`SIDE_EN` 为假。`SIDE_PINDIR` 为假，因为我们要驱动时钟信号的高低电平，而不是高低阻抗。`SIDESET_BASE` 为时钟信号对应的 GPIO 管脚编号。

### 3.5.2. 程序折返

PIO 程序通常有一个“外层循环”：在 FIFO 和外部之间传输数据时，它们需要反复执行同样的步骤。最小的示例就是前面的方波示例：

<caption>Pico 示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.pio 第7-12行</caption>

```
 7 .program squarewave
 8     set pindirs, 1 ; Set pin to output
 9 again:
10     set pins, 1 [1] ; Drive pin high and then delay for one cycle
11     set pins, 0 ; Drive pin low
12     jmp again ; Set PC to label `again`
```

程序的主题部分设置管脚为高，然后设置为低，从而产生方波的一个周期。然后整个程序进行循环，以产生周期性的输出。跳转指令本身需要一个周期，每个 `set` 指令也需要一个周期，所以为了保证高低部分的长度相同，`set pins, 1` 指令增加了一个周期的延时，使状态机在执行 `set pins, 0` 指令之前等待一个周期。每次循环总共需要 4 个周期。这里有两个问题：

1. `JMP` 占用了本可用于其他程序的指令内存
2. 执行 `JMP` 所需的额外周期导致最大输出速度降低了*一半*

由于程序计数器（`PC`）超过 31 时会自动折返为 0，因此将整个指令内存填充为 `set pins, 1` 和 `set pins, 0`，就能解决第二个问题，但这样很浪费。状态机有一个硬件特性，通过配置 `EXECCTRL` 控制寄存器，就能解决大部分情况下的问题。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave_wrap.pio 第11-19行</caption>

```
11 .program squarewave_wrap
12 ; Like squarewave, but use the state machine's .wrap hardware instead of an
13 ; explicit jmp. This is a free (0-cycle) unconditional jump.
14 
15     set pindirs, 1    ; Set pin to output
16 .wrap_target
17     set pins, 1 [1]   ; Drive pin high and then delay for one cycle
18     set pins, 0 [1]   ; Drive pin low and then delay for one cycle
19 .wrap
```

在执行完程序内存中的一条指令后，状态机会使用以下逻辑更新 `PC`：

1. 如果当前指令为 `JMP`，且 `Condition` 为真，则设置 `PC` 为 `Target`
2. 否则，如果 `PC` 等于 `EXECCTRL_WRAP_TOP`，则设置 `PC` 为 `EXECCTRL_WRAP_BOTTOM`
3. 否则，`PC` 加一，但如果当前值为 31，则设置为 0。

`pioasm` 中的 `.wrap_target` 和 `.wrap` 汇编标识实际上是标签。它们会导出常量，以写入到 `WRAP_BOTTOM` 和 `WRAP_TOP`中。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/generated/squarewave_wrap.pio.h 第1-37行</caption>

```c
// -------------------------------------------------- //
// This file is autogenerated by pioasm; do not edit! //
// -------------------------------------------------- //

#pragma once

#if !PICO_NO_HARDWARE
#include "hardware/pio.h"
#endif

// --------------- //
// squarewave_wrap //
// --------------- //

#define squarewave_wrap_wrap_target 1
#define squarewave_wrap_wrap 2

static const uint16_t squarewave_wrap_program_instructions[] = {
    0xe081, //  0: set    pindirs, 1                 
            //     .wrap_target
    0xe101, //  1: set    pins, 1                [1] 
    0xe100, //  2: set    pins, 0                [1] 
            //     .wrap
};

#if !PICO_NO_HARDWARE
static const struct pio_program squarewave_wrap_program = {
    .instructions = squarewave_wrap_program_instructions,
    .length = 3,
    .origin = -1,
};

static inline pio_sm_config squarewave_wrap_program_get_default_config(uint offset) {
    pio_sm_config c = pio_get_default_sm_config();
    sm_config_set_wrap(&c, offset + squarewave_wrap_wrap_target, offset + squarewave_wrap_wrap);
    return c;
}
#endif
```

这是 PIO 汇编器 `pioasm` 产生的原始输出，它创建了一个默认的 `pio_sm_config` 对象，其中包含了程序代码中的 `WRAP` 寄存器值。也可以直接初始化控制寄存器字段。


**注意** `WRAP_BOTTOM` 和 `WRAP_TOP` 是 PIO 指令内存中的绝对地址。如果程序加载到某个偏移量，那么必须相应地调整折返地址。

`squarewave_wrap` 示例中插入了延时周期，所以它的行为与原版 `squarewave` 程序完全相同。但由于使用了程序折返，我们现在可以去掉延时，
这样输出的速度就是以前的两倍，同时还能维持高电平和低电平的时间相等。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave_fast.pio 第12-18行</caption>

```
12 .program squarewave_fast
13 ; Like squarewave_wrap, but remove the delay cycles so we can run twice as fast.
14     set pindirs, 1    ; Set pin to output
15 .wrap_target
16     set pins, 1       ; Drive pin high
17     set pins, 0       ; Drive pin low
18 .wrap
```

### 3.5.3. FIFO 合并

默认情况下，每个状态机拥有两个 4 字长的 FIFO：一个用于将数据从系统传送到状态机（TX），一个用于反方向（RX）。但是，很多程序并不需要系统和状态机之间的双向数据传输，而更长的 FIFO 却很有用，特别是在 DPI 这种高带宽的接口中。在这些情况下，可以通过 `SHIFTCTRL_FJOIN` 选项将两个 4 字长的 FIFO 合并为一个 8 字长的 FIFO。

| ![图42](figures/figure-42.png) |
|:--:|
| 图42. 可合并的双 FIFO。每个 FIFO 为 4 字长，包括 4 个数据寄存器、一个 1：4 解码器和一个 4:1 多路复用器。<br>多路复用可以在 TX 和 RX 通道之间进行写数据和读数据操作，这样所有 8 个字都可以从两个端口进行访问。 |

另一个用例是 UART。由于 UART 的 TX/CTS 和 RX/RTS 部分是异步的，因此它们是在两个独立的状态机上实现的。而让每个状态机的一半 FIFO 资源空闲，是一种浪费。由于我们可以将两个 FIFO 合并成一个 TX FIFO 供 TX/CTS 状态机使用，或者合并成一个 RX FIFO 供 RX/RTS 状态机使用，这样就能利用全部资源。而拥有 8 字深 FIFO 的 UART 能够处理的中断数量是 4 字深 FIFO 的 UART 的两倍。

增加 FIFO 的深度（从 4 增加到 8）后，同一个状态机中的另一个 FIFO 的大小就变为零。例如，如果将 FIFO 合并供 TX 使用，那么就无法使用 RX FIFO，任何 `PUSH` 指令都会陷入等待状态。在 `FSTAT` 寄存器中，RX FIFO 同时表现为 `RXFULL` 和 `RXEMPTY` 状态。而合并到 RX 的情况正相反：TX FIFO 不可用，该状态机的 `FSTAT` 中 `TXFULL` 和 `TXEMPTY` 比特均为设置状态。

只要 DMA 没有因为其他竞合状态而变慢，那么 8 字 FIFO 足够通过 RP2040 的 DMA 实现每个周期 1 字的速率。

**注意** 改变 `FJOIN` 会抛弃当前状态机的 FIFO 中的一切数据。如果数据不能恢复，那么必须事先清空 FIFO 队列。

### 3.5.4. 自动推出和自动加载

随着 `OUT` 指令的执行，数据移出，OSR 会逐渐清空。OSR 为空后必须加载数据，比如通过 `PULL` 指令从 TX FIFO 传输一个字的数据到 OSR。类似地，ISR 为满后也必须清空。一种方法是，在移位一定数量的数据之后，通过循环执行一次 `PULL`：

```
 1 .program manual_pull
 2 .side_set 1 opt
 3 
 4 .wrap_target
 5     set x, 2                     ; X = bit count - 2
 6     pull            side 1 [1]   ; Stall here if no TX data
 7 bitloop:
 8     out pins, 1     side 0 [1]   ; Shift out data bit and toggle clock low
 9     jmp x-- bitloop side 1 [1]   ; Loop runs 3 times
10     out pins, 1     side 0       ; Shift out last bit before reloading X
11 .wrap
```

该程序按照每 4 个时钟周期 1 比特的速率，从每个 FIFO 字中移出 4 比特，同时输出时钟信号。当 TX FIFO 为空时，程序在时钟高电平处暂停（注意在指令暂停的周期上 side-set 依然会生效）。
图43演示了状态机执行该程序的过程。

| ![图43](figures/figure-43.png) |
|:--:|
| 图43. manual_pull 程序的执行过程。X 是循环计数器。每次循环会移出一比特数据，然后将时钟拉低，再拉高。<br>每条指令都有一个周期的延时，因此整个循环需占用四个时钟周期。在第三次循环后，移出第四比特，<br>然后状态机立即返回程序开头，重置循环计数器，并加载新的数据，同时维持每比特 4 个时钟周期的节奏。 |

该程序有一些限制：

- 它占用了 5 个指令槽，但只有 2 个是有用的（`out pins, 1 set 0` 和 `... set 1`），用于输出串行数据和时钟信号。
- 它的吞吐量的上限为系统时钟频率除以 4，因为它需要额外的周期来加载新数据并重新设置循环计数器。

这是非常常见的一种 PIO 程序，所以没个状态机都有一些额外的硬件来处理这种情况。状态机会跟踪从 OSR `OUT` 出的比特数，以及 `IN` 进 ISR 的比特数，在这些计数器达到某个可配置的阈值后，执行特定的动作。

- 在执行 `OUT` 指令时，如果达到或超过加载的阈值，且 TX FIFO 中有数据的话，状态机就会同时将 TX FIFO 中的数据加载到 OSR 中。
- 在执行 `IN` 指令时，如果达到或超过推出的阈值，且 RX FIFO 中有数据的话，状态机可以直接将移位结果写入 RX FIFO 中，并清除 ISR。

利用自动加载（autopull）功能，`manual_pull` 示例可以重写如下：

```
1 .program autopull
2 .side_set 1
3 
4 .wrap_target
5     out pins, 1    side 0      [1]
6     nop            side 1      [1]
7 .wrap
```

这个程序比原版本更短、更简单，而且如果去掉延时的话，运行速度是原版本的*两倍*，因为通过硬件加载 OSR 是“免费”的。注意程序并不知道下一次加载之前需要移位多少比特；硬件会在达到配置好的阈值（`SHIFTCTRL_PULL_THRESH`）后自动加载，所以示例程序也可以从每个 FIFO 字中移出 16 比特或 32 比特。

最后，注意上述程序与原本并非**完全**相同，因为它会在时钟信号低时暂停，而不是高的时候。我们可以通过 `PULL IFEMPTY` 指令改变暂停位置，该指令使用与自动加载（autopull）同样的可配置阈值：

```
1 .program somewhat_manual_pull
2 .side_set 1
3 
4 .wrap_target
5     out pins, 1    side 0      [1]
6     pull ifempty   side 1      [1]
7 .wrap
```

下面是完整的示例（PIO 程序，以及一个用于加载它并运行的 C 程序），演示了如何在同一个状态机上同时启用自动加载（autopull）和自动推出（autopush）。状态机 0 的功能是将数据从 TX FIFO 传输至 RX FIFO，吞吐量为每两个时钟周期一个字。该程序还演示了当 OSR 和 TX FIFO 均为空时，状态机会在执行 `OUT` 时进入暂停状态。

```
1 .program auto_push_pull
2 
3 .wrap_target
4     out x, 32
5     in x, 32
6 .wrap
```
```c
#include "tb.h" // TODO this is built against existing sw tree, so that we get printf etc

#include "platform.h"
#include "pio_regs.h"
#include "system.h"
#include "hardware.h"

#include "auto_push_pull.pio.h"

int main()
{
    tb_init();

    // Load program and configure state machine 0 for autopush/pull with
    // threshold of 32, and wrapping on program boundary. A threshold of 32 is
    // encoded by a register value of 00000.
    for (int i = 0; i < count_of(auto_push_pull_program); ++i)
        mm_pio->instr_mem[i] = auto_push_pull_program[i];
    mm_pio->sm[0].shiftctrl =
            (1u << PIO_SM0_SHIFTCTRL_AUTOPUSH_LSB) |
            (1u << PIO_SM0_SHIFTCTRL_AUTOPULL_LSB) |
            (0u << PIO_SM0_SHIFTCTRL_PUSH_THRESH_LSB) |
            (0u << PIO_SM0_SHIFTCTRL_PULL_THRESH_LSB);
    mm_pio->sm[0].execctrl =
            (auto_push_pull_wrap_target << PIO_SM0_EXECCTRL_WRAP_BOTTOM_LSB) |
            (auto_push_pull_wrap << PIO_SM0_EXECCTRL_WRAP_TOP_LSB);

    // Start state machine 0
    hw_set_bits(&mm_pio->ctrl, 1u << (PIO_CTRL_SM_ENABLE_LSB + 0));

    // Push data into TX FIFO, and pop from RX FIFO
    for (int i = 0; i < 5; ++i)
        mm_pio->txf[0] = i;
    for (int i = 0; i < 5; ++i)
        printf("%d\n", mm_pio->rxf[0]);

    return 0;
}

```

图44展示了状态机执行该程序的过程。初始时，OSR 为空，所以状态机会在第一条 `OUT` 指令处等待。当 TX FIFO 中有数据时，状态机会将数据传输到 OSR 中。在下一个周期，`OUT` 可以利用 OSR 中的数据执行（本例中将数据传输到可擦写寄存器 X 中），同时状态机会使用 FIFO 中的新数据填充 OSR。由于每一条 `IN` 指令都会立即填充 ISR，因此 ISR 一直为空，`IN` 就会直接把数据从 X 传输到 RX FIFO。

| ![图44](figures/figure-44.png) |
|:--:|
| 图44. `auto_push_pull` 程序的执行过程。状态机会在 `OUT` 上暂停，直到数据从 TX FIFO 进入 OSR。然后，<br>每次执行 OUT 操作，由于其比特数为 32，因此 OSR 都会同时自动加载，而 `IN` 的数据会越过 ISR，直接进入 RX FIFO。<br>当 FIFO 空且 OSR 也为空时状态机会暂停。 |

为了在正确的时间触发自动推出或自动加载，状态机会使用一对 6 比特计数器跟踪 ISR 和 OSR 的总移位数。

- 复位后，或在 `CTRL_SM_RESTART` 时，ISR 移位计数器设置为 0 （尚未移入任何数据），OSR 移位计数器设置为 32 （没有任何待移出数据）
- `OUT` 指令会给 OSR 移位计数器增加 `Bit count`
- `IN` 指令会给 ISR 移位计数器增加 `Bit count`
- `PULL` 指令或自动加载会将 OSR 计数器清为 0
- `PUSH` 指令或自动推出会将 ISR 计数器清为 0
- `MOV OSR, x` 或 `MOV ISR, x` 会将 OSR 或 ISR 移位计数器清为 0
- `OUT ISR, n` 指令会将 ISR 移位计数器设置为 `n`

在执行任何 `OUT` 或 `IN` 指令时，状态机都会将移位计数器与 `SHIFTCTRL_PULL_THRESH` 和 `SHIFTCTRL_PUSH_THRESH` 比较，来决定是否要采取动作。自动加载和自动推出分别由 `SHIFTCTRL_AUTOPULL` 和 `SHIFTCTRL_AUTOPUSH` 字段启用。

#### 3.5.4.1. 自动推出详解

下面是启用了自动推出（autopush）的 IN 指令的伪代码：

```
isr = shift_in(isr, input())
isr count = saturate(isr count + in count)

if rx count >= threshold:
    if rx fifo is full:
        stall
    else:
        push(isr)
        isr = 0
        isr count = 0
```

注意硬件平台执行上述步骤只需要一个机器时钟周期（除非出现暂停）。

阈值的范围为 1 ~ 32。

#### 3.5.4.2. 自动加载详解

在非 OUT 周期，硬件执行以下伪代码：

```
if MOV or PULL:
    osr count = 0

if osr count >= threshold:
    if tx fifo not empty:
        osr = pull()
        osr count = 0
```

因此，自动加载可以在两个 OUT 之间的任何点发生，取决于数据何时到达 FIFO。

在 OUT 周期，步骤略有不同：

```
if osr count >= threshold:
    if tx fifo not empty:
        osr = pull()
        osr count = 0
    stall
else:
    output(osr)
    osr = shift(osr, out count)
    osr count = saturate(osr count + out count)

    if osr count >= threshold:
        if tx fifo not empty:
            osr = pull()
            osr count = 0
```

硬件能够在移出全部数据的同时填充 OSR，因为这两个操作可以并行执行。但是，硬件无法在同一个周期执行填充 OSR 并 'OUT' 同一份数据，因为这样做会造成很长的逻辑链。

可以认为，对于程序而言，填充操作是异步的，但 'OUT' 就像一个数据屏障，状态机永远不能 OUT 尚未写入 FIFO 的数据。

注意，在自动加载启用时，从 OSR 'MOV' 的操作是未定义的；根据与系统 DMA 的竞合状态，你可能会读到尚未移出的残留数据，或读到 FIFO 的新数据。与此类似，'MOV' 到 OSR 的操作可能会覆盖刚刚自动加载进来的数据。但是，'MOV' 进 OSR 的数据永远不会被覆盖，因为 'MOV' 会更新移位计数器。

如果你**确实**需要读取 OSR 的内容，应当显式地执行某种 'PULL'。上面描述的不确定性是由硬件自动进行加载的代价。启用自动加载会改变 'PULL' 的行为：如果 OSR 为满，则 PULL 为误操作。这样做是为了避免与系统 DMA 之间的竞合冲突。它就像一个屏障：或者自动加载已经开始执行，此时 'PULL' 操作无效；或者程序会在 'PULL' 指令处等待，直到 FIFO 有数据。

'PUSH' 不存在相似的行为，因为自动推出没有这种不确定性。

### 3.5.5. 时钟分频器

PIO 依靠系统时钟运行，但对于绝大多数接口来说，系统时钟太快了，而可插入的 `Delay` 周期数是有限的。像 UART 等设备需要精确地控制并改变信号的速率，最好是运行同一个程序的多个状态机也能够独立地改变速率。因此，每个状态机都有一个独立的时钟分频器。

时钟分频器不会降低系统时钟速率，而是定义了多少个系统时钟周期等于执行 PIO 程序的“一个周期”。它会产生一个时钟允许信号，该信号能够在每个系统时钟周期暂停或恢复执行。时钟分频器会以一定的间隔产生允许脉冲，从而状态机能够以某个固定的、比系统时钟慢很多的速率运行。

时钟分频器可以简化状态机与系统之间的接口，降低延迟，而且占用的芯片面积很小。在时钟允许信号为低电平时，状态机完全处于空闲状态，不过系统仍然可以访问状态机的 FIFO 并改变其配置。

时钟分频器支持 16 比特整数、8 比特分数，小数分频器采用一阶积分-微分（first-order delta-sigma）。时钟分割比例可以在 1 ~ 65536 之间以 1/256 的增量改变。

如果时钟分割比例设置为 1，则状态机在每个时钟周期都会执行，即全速执行：

| ![图45](figures/figure-45.png) |
|:--:|
| 图45. 时钟分割比例为 1 时的状态机执行情况。通过 CTRL 寄存器允许状态机之后，每个时钟周期其时钟允许信号都为高。 |

通常，整数分割比例会让状态机以每 *n* 个时钟周期等于 1 个执行周期的速率执行，因此有效时钟速率为 *f<sub>sys</sub> / n*。

| ![图46](figures/figure-46.png) |
|:--:|
| 图46. 整数时钟分割比例产生一个周期性的时钟允许信号。时钟分频器会不断地从 *n* 开始递减，在到达 1 时产生一个允许脉冲。 |

分数分割比例会维持稳定的 *n + f / 256* 的分割速率，其中 *n* 和 *f* 是该状态机的 `CLKDIV` 寄存器的整数和分数部分。它会有选择地在 `n` 个周期到 `n+1` 个周期之间延长某个分割期间。

| ![图47](figures/figure-47.png) |
|:--:|
| 图47. 分数时钟分割比例，平均分割比例为 2.5。时钟分频器会在每个分割周期上改变分数部分的值，当到达 1 时，<br>则给整数部分加 1，以延长下一个分割周期。 |

对于较小的 *n*，分数分割比例导致的抖动可能是无法接受的。但是，当分割比例很大时，抖动几乎可以忽略不计。

**注意** 对于高速异步串口来说，最好使用偶数分割比例，或 1M 波特的整数倍，而不要使用传统的 300 的倍数，以避免不必要的抖动。

### 3.5.6. GPIO 映射

在内部，PIO 有一个 32 位寄存器，表示它能驱动的每个 GPIO 管脚的输出电平，以及另一个寄存器表示每个输出允许的情况（高低阻抗）。在每个系统时钟周期，每个状态机都可以写入这些寄存器中的部分或全部 GPIO。

| ![图48](figures/figure-48.png) |
|:--:|
| 图48. 状态机有两个独立的输出通道，一个由 OUT/SET 共享，另一个由 side-set 使用（可在任何时刻发生）。<br>三个独立的映射（每个映射包括第一个 GPIO 管脚号和 GPIO 个数）控制 OUT、SET 和 side-set 被定向到哪个 <br>GPIO。输入数据会根据哪个 GPIO 被映射到 IN 数据的 LSB 进行旋转。|

输出电平和输出允许寄存器的写数据和写掩码来自以下来源：

- 一条 `OUT` 指令，最多能写 32 比特。根据指令的 `Destination` 字段，该指令可以应用于管脚或管脚方向。`OUT` 数据的最低位映射到 `PINCTRL_OUT_BASE`，向后依次映射 `PINCTRL_OUT_COUNT` 个比特，在到达 GPIO31 后折返。
- 一条 `SET` 指令，最多能写 5 比特。根据指令的 `Destination` 字段，该指令可以应用于管脚或管教方向。`SET` 数据的最低位映射到 `PINCTRL_SET_BASE`，向后依次映射 `PINCTRL_SET_COUNT` 个比特，在到达 GPIO31 后折返。
- 一个 side-set 操作，最多能写 5 比特。根据 `EXECCTRL_SIDE_PINDIR` 寄存器字段的值，该操作可以应用于管脚或管教方向。side-set 数据的最低位映射到 `PINCTRL_SIDESET_BASE`，向后依次映射 `PINCTRL_SIDESET_COUNT` 个比特，在到达 GPIO31 后折返。

每个 `OUT`/`SET`/side-set 操作都会写连续的管脚区间，但每个区间都在 32 比特的 GPIO 空间内有独立的大小和位置。对于大多数应用来说，这已经足够灵活了。例如，如果一个状态机在一组管脚上实现了像 SPI 等接口，那么另一个状态机可以运行同一个程序，映射到另一组管脚上，提供第二个 SPI 接口。

在任意时钟周期，状态机可以执行一次 `OUT` 或 `SET`，并且可以同时执行一次 side-set。管脚映射逻辑会生成一个 32 比特的写掩码，并根据情趣内容和管脚映射配置，将输出电平和输出允许寄存器写入数据总线。

如果同一个状态机在同一个时钟周期内的 side-set 操作与 `OUT`/`SET` 操作重叠，那么在重叠的区域中 side-set 优先。

#### 3.5.6.1. 输出优先级

| ![图49](figures/figure-49.png) |
|:--:|
| 图49. 每个状态机对于每个 GPIO 通过写掩码选择优先级。<br>每个 GPIO 会考虑来自四个状态机的写入电平和方向，然后应用编号最大的状态机的值。 |

每个状态机在每个周期会通过其管脚映射硬件执行一次 `OUT`/`SET` 和 side-set。这样，每个状态机都会为 GPIO 输出电平和输出允许寄存器生成 32 比特的写数据和写掩码。

对于每个 GPIO，PIO 会考虑来自所有四个状态机的写操作，然后应用编号最大的状态机的写操作。这一步骤是针对输出电平和输出值分别进行的，所以可能出现在同一个周期内一个状态机同时改变了同一个管脚的电平和方向（例如通过同时发出的 `SET` 和 side-set），或一个状态机改变了 GPIO 方向，另一个状态机改变了同一个 GPIO 的电平。如果没有状态机写入 GPIO 的电平或方向，则值保持不变。

#### 3.5.6.2. 输入映射

`IN` 指令收到的数据的方式是，LSB 映射到 `PINCTRL_IN_BASE` 设置的 GPIO，后续的更高位来自后续更高编号的 GPIO，到达 31 时折返。

换句话说，`IN` 总线是 GPIO 输入值根据 `PINCTRL_IN_BASE` 的右旋结果。如果不够 32 个 GPIO，则 PIO 输入会在相应位置上填充 0， 以补齐 32 比特。

像 `WAIT GPIO` 等指令会使用绝对 GPIO 编号，而不是 `IN` 数据总线中的索引。此时不会进行右旋。

#### 3.5.6.3. 输入同步器

为了保证 PIO 的稳定性，每个 GPIO 输入都有两个标准 2-fliplop 同步器。这会导致输入采样延迟两个周期，但好处是，状态机可以在任何时刻执行 `IN PINS`，而且只会看到干净的高低电平，而不会看到可能会影响状态机电路的中间值。对于 UART RX 等异步接口来说这一点非常重要。

有时某些 GPIO 可能需要跳过同步器。这样可以减少延迟，但用户必须自行保证状态机不会在错误的时刻进行采样。通常，只有 SPI 等同步接口才能做到这一点。设置 `INPUT_SYNC_BYPASS` 中的相应比特即可跳过同步器。

**警告** 对不稳定的输入进行采样会导致不可预测的状态机行为，应当避免这种情况。

### 3.5.7. 强制指令和被 EXEC 的指令

除了指令内存外，状态机还可以从其他三个来源执行指令：

- `MOV EXEC` 可以从 `Source` 指定的某个寄存器执行指令
- `OUT EXEC` 可以执行 OSR 移出的数据
- 从 `SMx_INSTR` 控制寄存器执行指令，系统可以将指令直接写入该寄存器，以立即执行

```
 1 .program exec_example
 2 
 3 hang:
 4     jmp hang
 5 execute:
 6     out exec, 32
 7     jmp execute
 8 
 9 .program instructions_to_push
10 
11     out x, 32
12     in x, 32
13     push
```

```c
#include "tb.h" // TODO this is built against existing sw tree, so that we get printf etc

#include "platform.h"
#include "pio_regs.h"
#include "system.h"
#include "hardware.h"

#include "exec_example.pio.h"

int main()
{
    tb_init();

    for (int i = 0; i < count_of(exec_example_program); ++i)
        mm_pio->instr_mem[i] = exec_example_program[i];

    // Enable autopull, threshold of 32
    mm_pio->sm[0].shiftctrl = (1u << PIO_SM0_SHIFTCTRL_AUTOPULL_LSB);

    // Start state machine 0 -- will sit in "hang" loop
    hw_set_bits(&mm_pio->ctrl, 1u << (PIO_CTRL_SM_ENABLE_LSB + 0));

    // Force a jump to program location 1
    mm_pio->sm[0].instr = 0x0000 | 0x1; // jmp execute

    // Feed a mixture of instructions and data into FIFO
    mm_pio->txf[0] = instructions_to_push_program[0]; // out x, 32
    mm_pio->txf[0] = 12345678;                        // data to be OUTed
    mm_pio->txf[0] = instructions_to_push_program[1]; // in x, 32
    mm_pio->txf[0] = instructions_to_push_program[2]; // push

    // The program pushed into TX FIFO will return some data in RX FIFO
    while (mm_pio->fstat & (1u << PIO_FSTAT_RXEMPTY_LSB))
        ;

    printf("%d\n", mm_pio->rxf[0]);

    return 0;
}
```

这里我们向状态机中加载了一个示例程序，它做了两件事：

- 进入无限循环
- 进入一个循环，该循环反复地从 TX FIFO 加载 32 比特数据，然后将低 16 比特作为指令执行

C 程序将状态机设置为运行状态，然后进入 `hang` 循环。在状态机执行时，C 程序强制执行一条 `jmp` 指令，使状态机跳出循环。

当一条指令写入 `INSTR` 寄存器后，状态机会立即解码并执行该指令，而不会执行从 PIO 指令内存读取到的指令。程序计数器不会增加，因此在下一个周期（假设 `INSTR` 寄存器强制执行的指令没有导致等待状态）状态机会从原来的位置继续执行当前程序，除非写入的指令修改了 `PC`。

写入 `INSTR` 寄存器的指令中的延时周期会被忽略，并立即执行，不管状态机的时钟分频器如何设置。这个接口用于执行初始化或改变流程控制，所以指令会尽快执行，而不管状态机如何配置。

写入 `INSTR` 的指令可以进入等待状态，此时状态机会锁住该指令，直到其完成。出现这种情况时，会设置 `EXECCTRL_EXEC_STALLED` 标志。重启状态机或向 `INSTR` 写入 `NOP` 可以清除该标志。

上述示例状态机程序的第二阶段使用了 `OUT EXEC` 指令。`OUT` 本身需要一个执行周期，`OUT` 执行的指令在下一个执行周期进行。注意被执行的指令之一也是 `OUT`——状态机每个周期只能执行一条 `OUT` 指令。

`OUT EXEC` 会将 `OUT` 移出的数据写入内部的指令锁存器。下一个周期中，状态机知道它不应该执行指令内存，而是执行该锁存器的内容，而且知道此时不应当增加 `PC`。

该程序在运行时会输出 "12345678"。

**注意** 如果写入 `INSTR` 的指令造成等待状态，该指令会被写入到与 `OUT EXEC` 和 `MOV EXEC` 所用的同一个锁存器，并覆盖其中保存的指令。因此，如果使用了 `EXEC` 指令，那么写入 `INSTR` 的指令不能造成等待。
