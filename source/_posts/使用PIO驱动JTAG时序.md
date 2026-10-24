---
title: 使用PIO驱动JTAG时序
author: WangGaojie
---

## 制作新的CMSIS-DAP下载器
最近的业余时间都在研究做一些有意思的CMSIS-DAP兼容的调试器，之前做了一个带触摸屏的脱机下载器，做完之后感觉很有意思，主要是非常享受开发探索的过程，包括焊接PCBA、外壳建模这些都比较好玩儿。

最近又完成了一个比较有价值的事情，让RP2350单片机的PIO能够同时兼容驱动SWD和JTAG。之前做的那些调试器都只完成了SWD调试功能，主要是平时JTAG也基本上不怎么用。但有句话说的好，JTAG可以不用，但不能没有。

这次的设计目标就是做一个拥有20P的JTAG连接器端口的下载，使用RP2350芯片驱动，所有的驱动端口带可调参考电压的收发缓冲芯片。在动手做PCBA之前，我已经和AI充分讨论过可行性，在兼容SWD模式的情况下增加对JTAG的支持是可行的。但是AI给我的两个方案都存在一些性能问题被我抛弃了，转头我结合AI的思路找到了一条可行且高效的方案。

SWD模式的支持可以参考树莓派官方的设计方案，我的JTAG模式的支持也是在其之上魔改的。树莓派官方虽然在半年前就在讨论支持JTAG，但是一直没有下文。自己动手才是最有意思的事情。

## 硬件外围驱动与信号
首先介绍一下我的外围驱动电路，TCK、TDI、TDO都是单向的信号，之间经过一个数据缓冲芯片即可，TMS/SWD的信号要稍微复杂些，JTAG模式下TMS信号仅输出，但在SWD模式中它会频繁的切换收发方向。pico官方固件的做法是用一个IO保持持续输入，另一个IO保持输出并在需要读取数据时将输出IO切换到高阻态。这种方法的缺点是除了要控制外部缓冲器的开关外，还需要切换内部IO的方向。原固件时用PIO指令来切换方向，这种做法其实并不高明，它除了挤占TCK的时序外，还占用PIO指令空间。

我驱动SWD信号的方案是用两个IO信号，分别作为输入和输出，即SWD_DO和SWD_DI，两个IO经过缓冲芯片后并联到外部的SWD，SWD_DI的输入缓冲是常开的。SWD_DO的输出是通过另一个IO驱动的。这样RP2350内部不需要切换IO方向，通过一个IO来控制当前的SWD是由哪个端口驱动。

完整的信号通路是这样的：

| 信号 | 通路 | 说明 |
| --- | --- | --- |
| SWD_DO / TMS | GPIO6 → 缓冲器 A → 外部 | 缓冲器 A 的通断由 `PROBE_SWD_DOE_PIN`（GPIO4）控制，高 = 正常驱动，低 = 高阻 |
| SWD_DI | 外部 → GPIO5 | 独立输入线，经过独立的缓冲器 |
| TCK/SWCLK | GPIO3，PIO sideset输出 | 两种模式共用 |
| TDI | GPIO0，JTAG 下归 PIO 输出 TDI；SWD 下归 CDC 桥 UART0 TX | UART TX 在 JTAG 模式改由 GPIO12 输出 |
| TDO | GPIO2，JTAG 下由 PIO 输入 | 独立输入通路，与 DOE 无关；已置 `input_sync_bypass` |
| nTRST | GPIO9，推挽输出 |  |
| nRESET | GPIO7，开漏模拟 |  |

TDO 与 SWD_DI 都是独立输入线，不经缓冲器 A，因此读回不受 DOE 影响。而 TMS 与 SWD_DO 共用 GPIO6、必须经缓冲器输出，所以 **JTAG 扫描期间 DOE 必须恒为 1**。

PIO程序中通过IN、OUT指令与外部GPIO交换数据，SWD模式和JTAG模式下的IO绑定需要动态切换。SWD模式下，数据输出OUT绑定到SWD_DO，数据输入IN绑定到SWD_DI。JTAG模式下数据输出OUT绑定到TDI，数据输入IN绑定到TDO。

两个复位引脚作为普通IO控制即可，这里要注意nRESET必须用开漏输出，不能推挽输出高电平，否则与其它的复位电路一起工作时会发生短路。

这里还有注意一点，为了兼容Jlink的20P端口定义，TDI端口在SWD模式中是复用作为串口的TX使用的。在模式切换时需要注意串口端口的切换。

## PIO程序
下面这段程序就是该项目最具精华的部分了，尤其是实现TMS序列的驱动，我自认为很有创意，整个项目的代码虽然完全由AI完成，但其中的细节和巧思都是我反复斟酌打磨的结果。

```
.program probe
.side_set 2 opt

public turnaround_cmd:
turnaround_bitloop:
    nop                         [1]  side 0x0
    jmp x-- turnaround_bitloop  [1]  side 0x1
    jmp get_next_cmd

public write_cmd:
    pull
public short_write_cmd:
write_bitloop:
    out pins, 1                 [1]  side 0x2   ; Data is output by host on negedge
    jmp x-- write_bitloop       [1]  side 0x3   ; ...and captured by target on posedge
                                                ; Fall through to next command
.wrap_target
public get_next_cmd:
    pull                             side 0x0   ; SWCLK initially low, OEn disabled
    out x, 7                                    ; Get bit count
    out pc, 5                                   ; Go to command routine

read_bitloop:
    nop                                         ; Additional delay on taken loop branch
public read_cmd:
    in pins, 1                  [1]  side 0x1   ; Data is captured by host on posedge
    jmp x-- read_bitloop             side 0x0
    push
    jmp get_next_cmd                            ; Fall through to next command

; JTAG commands. TMS shares the SWD_DO pin (set pin group, base configured per mode
; on the C side), TDI is driven by the out pin group (same base), TDO is sampled by
; the in pin group. Sideset bit 1 = DOE must stay 1, otherwise the external buffer
; goes high-Z and TMS cannot be driven.
public jtag_tms_lo_cmd:
    set pins, 0                      side 0x2   ; DOE=1, TCK=0
    jmp get_next_cmd

public jtag_tms_hi_cmd:
    set pins, 1                      side 0x2
    jmp get_next_cmd

; TMS sequence: after "set", "out pc, 5" loads the entry address of the next beat
; (a 5-bit field in the command word), so one FIFO command chains 1 to 4 beats of
; "TMS + clock" without dispatching a command on every beat.
public jtag_tms_seq_lo_cmd:
    set pins, 0                 [1]  side 0x2   ; 2 cycles: TMS=0, TCK low
    out pc, 5                   [1]  side 0x3   ; 2 cycles: load next entry, TCK high

public jtag_tms_seq_hi_cmd:
    set pins, 1                 [1]  side 0x2
    out pc, 5                   [1]  side 0x3

public jtag_rw_cmd:
    pull                                        ; second FIFO word carries TDI data
public jtag_rw_short_cmd:                       ; TDI inlined in command word [31:12]
jtag_rw_bitloop:
    out pins, 1                 [1]  side 0x2   ; 2 cycles: TCK low, shift out one TDI bit
    in pins, 1                       side 0x3   ; 1 cycle: TCK high, sample TDO
    jmp x-- jtag_rw_bitloop          side 0x2   ; 1 cycle: TCK low, gives TDO 3 cycles to settle
    push
.wrap

```

## 实测效果
我将PCBA焊接完成并把固件开发调试完成后，实测在SWD模式下时钟频率能够跑的30MHz以上，JTAG模式能将时钟跑到25MHz以上。我的测试目标板时STM32F407芯片，这是否是这个调试器的极限还是未知的。

但有一点是可以确定的，那就是RP2350的USB不是480M的高速USB，性能的瓶颈永远不是底层的IO驱动时序和单片机的频率。

## 项目开源
项目的完整代码提交到了 https://github.com/DazzlingOkami/debugprobe_jtag

但非常抱歉的是因为项目中之前包含了一些敏感信息，我就将历史提交信息删除了，但相关的设计细节都有非常详细的文档可以参考。

## 后续计划
我认为这个PIO的玩法已经到头了，脱机下载也做了好几个设备了。后面我打算用FPGA来开发一些相关的设备，那样应该更有意思。
