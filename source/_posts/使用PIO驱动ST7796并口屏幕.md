---
title: 使用 PIO 驱动 ST7796 并口屏幕
author: WangGaojie
---

## 前言

这是我的另一个个人项目的前置探索任务，目的是验证树莓派 RP2350 芯片能否驱动一块小屏幕，并达到预期的显示效果。综合考虑屏幕尺寸、分辨率、刷新率和驱动难度等因素，最终选择了一块基于 ST7796U 驱动芯片的 3.5 寸屏幕，分辨率为 480x320。

ST7796U 采用 16 位 8080 并口传输数据。由于只需要向屏幕写入数据，需要控制的信号线不多，时序也比较简单。

在开发过程中，我发现目前没有类似的开源实现，因此将代码上传到了 GitHub：[st7796u_demo](https://github.com/DazzlingOkami/st7796u_demo) ，有类似需求的朋友可以作为参考。

## 项目概述

项目基于 LVGL 图形库构建了一个演示程序，可以充分评估这块屏幕的显示效果和触摸效果。

实际使用下来发现，受限于 150MHz 的主频：
- 复杂动画场景下帧率约为 30FPS
- 简单界面下帧率能够稳定在 50FPS

整个实现采用 DMA + PIO 方案，实际数据传输帧率可以达到 100FPS。因此瓶颈在于处理器的渲染能力跟不上，而非屏幕接口。

考虑到 RP2350 是多核处理器，我尝试开启了 LVGL 的多任务渲染模式，但运行 benchmark 发现渲染能力反而有所下降，具体原因没有深入研究。

## PIO 核心代码

8080 并口协议的核心 PIO 代码仅用 8 行就实现了：

```asm
.program st7796u_8080_bus
.side_set 1 opt

.wrap_target
    pull                            ; Wait command and data
    set pins, 0                     ; CS low and DCX low
    out pins, 16            side 0  ; Output command to data lines on negedge
    out x, 32               side 1  ; Get parameter length
    set pins, 2                     ; Set DCX high

bus_write_loop:
    out pins, 16            side 0  ; Host output data on negedge
    jmp x-- bus_write_loop  side 1  ; ST7796U captured on posedge

    set pins, 3                     ; Set CS high
.wrap
```

### 代码说明

- `side_set 1 opt`：使用 1 位 side-set 引脚控制 WRX 信号
- `pull`：从 FIFO 接收命令和数据
- `out pins, 16`：输出 16 位数据到并口
- `out x, 32`：获取参数长度
- `bus_write_loop`：循环写入数据

## 踩坑记录

### 屏幕复位时序问题

在开发驱动代码的过程中，屏幕一直无法点亮。经过排查，最终发现是复位时序延时不够导致的。测量 IO 时序没有发现问题，直到对照数据手册后才确认问题所在。

### 触摸驱动问题

触摸屏的 I2C 从设备地址会根据复位时序不同而变化。由于 RST 初始电平不正确，导致触摸屏一直不响应触摸事件。这些问题最终都得到了解决。

## 画面撕裂问题与垂直同步

测试阶段发现屏幕画面存在撕裂现象，这需要通过垂直同步（TE 信号）来处理。由于早期设计 demo 电路板时忽略了 TE 信号线，它没有连接到主控芯片。

解决思路：将 TE 信号连接到单片机后，向 0x35 寄存器写入 0x00 开启 mode1 模式的同步信号输出。由于总线速度（100FPS）比内部显示刷新速度（60FPS）快，单片机写显存前应等待 TE 输出高电平，即内部刷新完成一次的结束时刻。

同时 PIO 代码也需要做适当调整，将 command 的 16 位数拆分为高 8 位和低 8 位，高 8 位用于标识该条指令是否需要垂直同步：

```asm
.program st7796u_8080_bus
.side_set 1 opt

.wrap_target
    pull                            ; Wait command and data
    set pins, 0                     ; CS low and DCX low
    out pins, 8             side 0  ; Output command to data lines on negedge
    out y, 8                        ; Y register stores vsync flag
    out x, 32               side 1  ; Get parameter length
    set pins, 2                     ; Set DCX high

    jmp !y no_vertical_sync         ; Skip vertical sync wait if not needed
    wait 1 pin 0                    ; Wait input pin0(TE) to high

no_vertical_sync:
bus_write_loop:
    out pins, 16            side 0  ; Host output data on negedge
    jmp x-- bus_write_loop  side 1  ; ST7796U captured on posedge

    set pins, 3                     ; Set CS high
.wrap
```

## 总结

该触摸屏的实际使用效果比较符合我的预期。垂直同步功能将在后续具体项目中进行完善。
