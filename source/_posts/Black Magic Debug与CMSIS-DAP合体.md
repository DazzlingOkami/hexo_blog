---
title: Black Magic Debug与CMSIS-DAP合体
author: WangGaojie
---

## 移植Black Magic Probe固件
虽然BMP官方已经支持了许多的芯片，但是我决定做一些不一样的尝试，我最开始将Black Magic Probe的代码移植到了STM32H750芯片平台上，希望它较高的主频可以实现更快的调试速度。但实际测试下来发现或许是因为相同的USB2.0FS，性能并不比STM32F411平台有特别明显的优势。后来又将其移植到CH32V307平台上，该芯片拥有USB2.0HS接口，传输速度将不是它的性能瓶颈，但是该芯片的主频与F411类似，综合权衡后，我决定采用CH32V307芯片重新设计一版Black Magic Probe。但是正当我把硬件原理图设计完成后，我突然想到一个更合适的选择，那就是RP2040芯片。

## 为什么选择RP2040芯片
选择它的主要原因是它的PIO模块可以实现较高效率的SWD时序，可以不必依赖与处理器去模拟SWD时序。对于SWD信号的准确性和稳定性都是非常关键的，所以选择RP2040就比较合适。我本来已经完成了SWD时序的PIO代码以及和Black Magic Probe的适配工作，但是我发现树莓派官方推出了基于rp2040芯片的CMSIS-DAP调试器，其中使用PIO完成的SWD时序也比较符合我的预期。所以我决定将Black Magic Probe固件代码移植到rp2040芯片上，并保留CMSIS-DAP调试器的支持,使得一个硬件拥有两种调试能力。

## 设计硬件
最开始打算使用树莓派官方的Debug Probe硬件，但是60多元的售价将我劝退。而且为了满足Black Magic Probe的一些特殊功能，Debug Probe硬件并不是最理想的硬件。所以我决定自行设计一个硬件。主要是增加3.3V电源输出管理和电源监测功能，需要添加ADC电路和功率开关电路。

![hardware](hardware.png)

主要的外围接口和我上一个制作的Black Magic Probe保持相同的接口布局。[原理图](https://datasheets.raspberrypi.com/debug/raspberry-pi-debug-probe-schematics.pdf)可参考树莓派的Debug Probe。

硬件上添加了一个拨动开关，可以用于控制3.3V的电源的输出，3.3V的电源输出也可以由BMP软件控制打开。

让我感到非常意外的是RP2040的ADC性能不太理想，采样时间固定到了2us，没有调整的空间。手册上说ADC的输入阻抗可达100kΩ，但实际我使用100kΩ加47kΩ的分压电路都导致ADC测量结果严重偏低(偏低0.15v)，最终我将分压电阻的值缩小了10倍后ADC的测量结果才有所好转。如果采样时间可以修改或许可以提高采样精度。

除PCB成本外，元器件的成本约20元。

## 代码
完整的代码已经放到[github](https://github.com/DazzlingOkami/debugprobe)。

代码采用rp2040芯片最新的2.0版SDK，构建工具依赖的picotools已经生成了一个Windows的本地版本。如果你在Windows下进行rp2040芯片软件开发，这个二进制版本的picotools将对你有一定帮助。构建这个Win版本的picotools有许多曲折的经历，不过最后还是弄好了，不然的话只能转向Linux开发了。

代码在树莓派debugprobe固件的基础上移植了比较完整的Black Magic Probe固件(除了BMP-bootloader)。JTAG调试接口由于原版树莓派没有支持，所以BMP上的JTAG adapter还未完成。

该版本的BMP相比于其它版本有一些优点，准确且高效的SWD时序模拟，时钟频率理论可达30MHz，时序由PIO驱动，时钟精度比较高。相比于采用处理器模拟SWD时序，它的时钟稳定性更佳。使用逻辑分析仪测量波形：

![bmp-pico](bmp-pico.png)

从图中可以看到CLK时序还算比较规整。由于在底层设计中PIO需要和处理器做同步交互，使得在SWD时序中CMD-ACK-DATA之间存在不可避免的时序空闲时间。

下面看看STM32F411使用指令模拟SWD时序的波形：

![bmp_native](bmp-native.png)

可以看出CLK的占空比不稳定，这种现象称为时钟抖动(Jitter)，它主要影响信号的采样过程，过大的抖动可能导致传输误码。

该版本代码还有一些其它优化，重构了cdc-uart的代码，串口改为DMA驱动，配合应用层的FIFO，实现了串口驱动到USB协议栈之间数据的零拷贝。

此外，为了实现debugprobe代码和BMP的代码同时运行，debugprobe代码在保证功能性的前提下做了一些细节部分的调整，具体内容可以参考代码提交记录。

我还启用了FreeRTOS的多核特性，以充分利用rp2040的2个核心。这部分代码还未提交，我还需要进一步测试。

## PIO驱动SWD原理
先给出Raspberrypi的PIO代码，如果对PIO还不够深入了解可以看看我之前的文章。
```asm
.program probe
.side_set 1 opt

public write_cmd:
public turnaround_cmd:                      ; Alias of write, used for probe_oen.pio
    pull
write_bitloop:
    out pins, 1             [1]  side 0x0   ; Data is output by host on negedge
    jmp x-- write_bitloop   [1]  side 0x1   ; ...and captured by target on posedge
                                            ; Fall through to next command
.wrap_target
public get_next_cmd:
    pull                         side 0x0   ; SWCLK is initially low
    out x, 8                                ; Get bit count
    out pindirs, 1                          ; Set SWDIO direction
    out pc, 5                               ; Go to command routine

read_bitloop:
    nop                                     ; Additional delay on taken loop branch
public read_cmd:
    in pins, 1              [1]  side 0x1   ; Data is captured by host on posedge
    jmp x-- read_bitloop         side 0x0
    push
.wrap                                       ; Wrap to next command

```

代码非常简短，共11条指令。代码的入口在get_next_cmd，从输入fifo中得到需要执行的指令，执行分为4种(读、写、next、turnaround)，读和写就是构建基本的数据传输波形，next相当于一条nop指令，但是可以用于直接切换DIO的方向，turnaround将产生一个额外的时钟周期并切换DIO方向。

处理器将指令偏移压入fifo中，pio将fifo中的数据放入到pc寄存器实现指令的跳转。

## PIO练习
在使用debugprobe的代码之前，我自己练习写了一份SWD的时序代码，该代码的优点是在数据传输过程中自动进行DIO的方向切换不需要单独占用一个fifo空间。该代码需要较多的指令空间，由于篇幅有限且代码质量一般，这里仅列出in时序相关内容。
```asm
.program swdptap_seq_in
.side_set 1 opt

.wrap_target
start:
    pull            ; 获取待处理的数据
    out x, 1        ; 检查是否需要Trn切换
    jmp !x no_turnaround
    set pindirs 0 side 1 [1] ; 切换DIO到输入模式
no_turnaround:
    out x, 5        ; 获取输入数据长度
    out y, 1        ; 是否获取校验位
    jmp loop2
loop:
    nop [3]
loop2:
    in pins, 1    side 0 [4]
    jmp x-- loop  side 1
    push                     ; 将采集到的数据写入FIFO
    jmp !y start             ; 如果y==0则不校验返回到起始位置
    nop [1]
    in pins, 1    side 0 [4] ; 获取校验信息
    nop           side 1 [4] ; 一个Trn周期
    set pindirs 1 side 0     ; 切换DIO到输出模式
    push                     ; 将校验bit写入FIFO
.wrap

% c-sdk {
#include "hardware/clocks.h"
#include "hardware/gpio.h"

static inline void swdptap_seq_in_program_init(PIO pio, uint sm, uint offset, uint pin_swclk, uint pin_swdio, uint freq){
    pio_gpio_init(pio, pin_swclk);
    pio_gpio_init(pio, pin_swdio);
    pio_sm_set_pindirs_with_mask(pio, sm, (1u << pin_swclk), (1u << pin_swclk));
    pio_sm_set_pindirs_with_mask(pio, sm, 0, (1u << pin_swdio));

    pio_sm_config c = swdptap_seq_in_program_get_default_config(offset);

    sm_config_set_in_pins(&c, pin_swdio);
    sm_config_set_sideset_pins(&c, pin_swclk);
    sm_config_set_set_pins(&c, pin_swdio, 1);

    sm_config_set_fifo_join(&c, PIO_FIFO_JOIN_NONE);

    float div = (float)clock_get_hz(clk_sys) / (freq * 2) / 5;
    sm_config_set_clkdiv(&c, div);

    sm_config_set_out_shift(&c, true, false, 8);
    sm_config_set_in_shift(&c, true, false, 32);

    pio_sm_init(pio, sm, offset, &c);
    pio_sm_set_enabled(pio, sm, true);
}
%}
```

```c
uint32_t swdptap_seq_in(size_t clock_cycles){
    if(clock_cycles == 0){
        return 0;
    }

    int ret;
    ret = cros_sem_pend(swdptap_lock, 1000u);
    if(ret){
        return 0;
    }

    uint32_t has_trn = 0;
    if(dio_dir != SWDIO_STATUS_FLOAT){
        dio_dir = SWDIO_STATUS_FLOAT;
        has_trn = 1;
    }

    if(seq_out_count > 0){
        pio_sm_get_blocking(SWD_PIO, SWD_OUT_SM);
        seq_out_count--;
    }
    
    pio_sm_put_blocking(SWD_PIO, SWD_IN_SM, has_trn | ((clock_cycles - 1) << 1));
    ret = pio_sm_get_blocking(SWD_PIO, SWD_IN_SM) >> (32U - clock_cycles);

    cros_sem_post(swdptap_lock);
    return ret;
}

bool swdptap_seq_in_parity(uint32_t *ret, size_t clock_cycles){
    if(clock_cycles == 0){
        return false;
    }

    if(cros_sem_pend(swdptap_lock, 1000u)){
        return false;
    }

    uint32_t has_trn = 0;
    if(dio_dir != SWDIO_STATUS_FLOAT){
        dio_dir = SWDIO_STATUS_FLOAT;
        has_trn = 1;
    }

    if(seq_out_count > 0){
        pio_sm_get_blocking(SWD_PIO, SWD_OUT_SM);
        seq_out_count--;
    }

    pio_sm_put_blocking(SWD_PIO, SWD_IN_SM, has_trn | ((clock_cycles - 1) << 1) | (1 << 6));
    uint32_t value = pio_sm_get_blocking(SWD_PIO, SWD_IN_SM) >> (32U - clock_cycles);
    uint32_t parity = pio_sm_get_blocking(SWD_PIO, SWD_IN_SM) >> 31;
    dio_dir = SWDIO_STATUS_DRIVE;

    cros_sem_post(swdptap_lock);
    *ret = value;
    return __builtin_parity(value) == parity;
}

```

该代码完成的时序效果如下，可以得到更紧凑的CLK波形，CLK空闲时间更短。

![pio-swd](pio-swd.png)

## 一个额外的软件
很多人喜欢Jlink除了它性能强大外，还因为它用于丰富的上位机软件支持，比如J-Flash，实现固件烧录比较方便。对于CMSIS-DAP来说，上位机软件选择似乎不多，除了IDE内部集成外，只有openocd这个选择。但是openocd没有官方支持的GUI程序，使用命令行在一些效率上还是不及J-Flash这种图形界面来的方便，毕竟openocd的命令太多了。

所以我开发了一个用于CMSIS-DAP的上位机工具叫DFlash，支持大部分Cortex单片机(底层采用openocd，所以就是openocd支持的芯片)。支持固件的上传和下载，支持elf、hex、bin文件，可以显示文件的二进制数据视图。文件支持拖拽打开，大部分逻辑和jflash类似。打开bin文件时会弹出对话框提示输入下载地址，hex和elf文件可以自动识别下载地址。

支持v1和v2版本的CMSIS-DAP，可以搜索本机连接的多个CMSIS-DAP，并选择下载器。

![DFlash](dflash.png)

目前软件的主要功能已经完成，还需要添加unlock等chip操作功能。此外还需要完善下载进度和UI上的一些调整。

功能完善后会发布并开源该软件。
