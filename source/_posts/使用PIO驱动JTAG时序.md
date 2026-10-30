---
title: 使用PIO驱动JTAG时序
author: WangGaojie
---

## 制作新的CMSIS-DAP调试器
最近的业余时间都在研究做一些有意思的CMSIS-DAP兼容的调试器，之前做了一个带触摸屏的脱机调试器，做完之后感觉很有意思，主要是非常享受开发探索的过程，包括焊接PCBA、外壳建模这些都比较好玩儿。

最近又完成了一个比较有价值的事情，让RP2350单片机的PIO能够同时兼容驱动SWD和JTAG。之前做的那些调试器都只完成了SWD调试功能，主要是平时JTAG也基本上不怎么用。但有句话说的好，JTAG可以不用，但不能没有。

这次的设计目标就是做一个拥有20P的JTAG连接器端口的下载，使用RP2350芯片驱动，所有的驱动端口带可调参考电压的收发缓冲芯片。在动手做PCBA之前，我已经和AI充分讨论过可行性，在兼容SWD模式的情况下增加对JTAG的支持是可行的。但是AI给我的两个方案都存在一些性能问题被我抛弃了，转头我结合AI的思路找到了一条可行且高效的方案。

SWD模式的支持可以参考树莓派官方的设计方案，我的JTAG模式的支持也是在其之上魔改的。树莓派官方虽然在半年前就在讨论支持JTAG，但是一直没有下文。自己动手才是最有意思的事情。

该版本还重点优化调试器的传输速度，用该版本的调试器向STM32H7芯片写SRAM，速度可以超过500k字节每秒，已经非常接近USB2.0FS的速度极限了。

## 项目开源
项目的完整代码提交到了 https://github.com/DazzlingOkami/debugprobe_jtag

但非常抱歉的是因为项目中之前包含了一些敏感信息，我就将历史提交信息删除了，但相关的设计细节都有非常详细的文档可以参考。

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

## 实测效果与分析
我将PCBA焊接完成并把固件开发调试完成后，实测在SWD模式下时钟频率能够跑的30MHz以上，JTAG模式能将时钟跑到25MHz以上。我的测试目标板时STM32F407芯片，这是否是这个调试器的极限还是未知的。

理论上CMSIS-DAP使用bulk模式传输数据包会比HID快得多，但这有一些前提。那就是上位机能够使用队列来发送请求，将我的CMSIS-DAP的队列长度设置到8之后，我发现在openocd(v0.12)中速度仅仅比无队列版本提升了一倍，无队列版本写入SRAM的速度在46kB/s，打开队列后速度来到了95kb/b。

经过深入的排查，发现openocd在该版本中限制了队列的长度：
```c
cmsis_dap_handle->packet_count = MIN(MAX_PENDING_REQUESTS, pkt_cnt);
```
在该版本的openocd中，无论CMSIS-DAP中设置的包队列长度是多少，都会有一个下限，值为3。但是仍然不能解释我实测的95kb/s，95kb/s相对于队列中仅有一个在排队处理。通过抓取CMSIS-DAP的日志也能确认这一点。

为了分析速度问题，更进一步锁定到openocd使用libusb库时调用阻塞存在用法问题，在函数cmsis_dap_swd_run_queue实现中，检测到当前的队列中存在待处理数据时，本来应该无阻塞的读取数据，结果实际上是阻塞的模式进行调用。导致openocd的队列在未满的情况下出现了阻塞。在目前的openocd主线版本中这个问题已经解决，默认最小队列也增加到了4。但是目前主线没有发表正式的版本，要体验更快的性能只能自己编译或使用其它分支的发布版。

在SWD协议中，每写入32bit的数据实际上至少需要46个TCK时钟，即20MHz的时钟时写SRAM的速度最高为1.7MB/s，30MHz时钟的写入速度为2.6MB/s。

USB2.0FS的USB数据包大小为64字节，使用DAP_TransferBlock请求是，有效数据承载为56字节。每毫秒的bulk事务可以传输最多19个满载数据包。考虑到有DAP的响应数据包等因素，如果按照满载10个数据包计算，则写SRAM的最高速度为560kb/s。该速度对应的SWD时钟为6.44MHz。

在RP2350的USB上实测bulk的吞吐能力，每次PC写入64的字节包，设备返回4字节的数据包。在常规的一收一发模式下，能够达到每秒4839个请求。如果是批量发送再批量接收，每秒可以达到6105个请求。异步收发模式下，每秒能够达到8541个请求。将主机端的处理深度由4增加到8后，提升仅有1%。以最新版本的openocd的队列长度4作为测试基准比较有意义。

| 模式 | depth | 响应 | 请求/秒 | OUT | IN | 合计 |
| --- | --- | --- | --- | --- | --- | --- |
| serial | 1 | 4 B | 5162 | 330.3 KB/s | 20.6 KB/s | 351.0 KB/s |
| batch | 4 | 4 B | 6693 | 428.3 KB/s | 26.8 KB/s | 455.1 KB/s |
| batch | 8 | 4 B | 6929 | 443.4 KB/s | 27.7 KB/s | 471.1 KB/s |
| async | 1 | 4 B | 10103 | 647.0 KB/s | 40.4 KB/s | 687.4 KB/s |
| async | 2 | 4 B | 10430 | 667.6 KB/s | 41.7 KB/s | 709.3 KB/s |
| async | 4 | 4 B | 10409 | 666.2 KB/s | 41.6 KB/s | 707.8 KB/s |
| async | 8 | 4 B | 9996 | 639.8 KB/s | 40.0 KB/s | 679.7 KB/s |
| async | 16 | 4 B | 10181 | 651.6 KB/s | 40.7 KB/s | 692.3 KB/s |
| async | 8 | 16 B | 10243 | 655.6 KB/s | 163.9 KB/s | 819.5 KB/s |
| async | 8 | 32 B | 9740 | 623.8 KB/s | 311.7 KB/s | 935.5 KB/s |
| async | 8 | 64 B | 7989 | 511.7 KB/s | 511.3 KB/s | 1023.0 KB/s |

在RP2350芯片上如果要让双核MCU跑满，除了RTOS任务调度要配置合理外，将程序放到SRAM中运行是非常有必要的。如果程序不放到SRAM中运行，两个核心的任务会同时抢占XIP的缓存，导致整体性能及其不稳定，尤其是在追求极限下载速度时。

## 后续计划
目前我看到市面上有量产版本的DAPLink设备能够将SRAM写入测试速度跑到1MB/s。我认为这个PIO的玩法已经到头了，脱机下载也做了好几个设备了。后面我打算用FPGA来开发一些相关的设备，主要是为了消除底层JTAG时序间的“气泡”。将读写速度做到2MB/s，成本控制到100元内。
慢慢玩儿吧。
