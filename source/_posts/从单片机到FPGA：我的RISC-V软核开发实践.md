---
title: 从单片机到FPGA：我的RISC-V软核开发实践
author: WangGaojie
---

# 从单片机到 FPGA：我的 RISC-V 软核开发实践

## 一、单片机数据处理瓶颈引发的技术思考

在近期的项目开发中，我遭遇了一个棘手的技术挑战：需要在单片机端直接处理 2.6Msps 的数据量，而 500MHz 主频限制下，每个数据的处理时间仅有不到 200 个时钟周期。传统方案中，double 类型的高精度运算严重消耗计算资源，数据总线带宽压力巨大。经过反复优化，最终通过将数据类型替换为 long 型，并借助汇编实现部分并行处理，才勉强满足实时性要求。

这次经历让我深刻意识到：单片机在面对大规模数据并行运算时存在天然局限。其冯・诺依曼架构下的顺序执行模式，难以高效处理高密度计算任务。而 FPGA 凭借硬件并行处理的特性，显然更适合这类场景。于是，我决定通过一个实战项目开启 FPGA 的学习之旅，目标是构建 STM32 与 FPGA 协同工作的异构计算系统。

## 二、项目架构设计：构建异构计算系统

### （一）核心功能定义

项目的核心目标是在 FPGA 内运行 RISC-V 软核，由 STM32 通过 FMC 总线完成程序加载与运行控制，并实现两者的数据交互。这一架构借鉴了工业控制中 "ARM+FPGA" 的经典组合模式：STM32 负责上层逻辑控制，FPGA 承担底层并行计算，通过内存映射的 FMC 总线实现高效通讯。

### （二）关键技术选型

**RISC-V 软核**：选择 [PicoRV32](https://github.com/YosysHQ/picorv32) 作为核心，因其轻量级设计和开源生态，便于快速集成。

**硬件平台**：

FPGA 选用安路科技 EF2L45LG144B 开发板，其内置的 256KB 双端口 ERAM 成为关键：一个端口连接 RISC-V 核（32 位数据宽度），另一个端口通过 FMC 接口与 STM32（16 位数据宽度）通信，数据位宽转换通过地址低位自动选择高低字节实现。

自主设计 STM32 扩展板，通过FPGA开发板预留接口与 STM32 实现硬件级联，构建完整的双核心系统。

## 三、Verilog 设计实践：从模块集成到时序优化

### （一）顶层架构设计

```verilog
`timescale  1 ps / 1 ps

module top(
    input clk_ref,
    input i_rst,

    inout [15:0] fmc_data,
    input fmc_wrn,
    input fmc_rdn,
    input fmc_csn,
    input fmc_nadv
);
    wire clk;
    wire rstn;

    sys_clk u_sys_clk(
        .clk_ref(clk_ref),
        .i_rst(i_rst),
        .clk(clk),
        .rstn(rstn)
    );

    wire [15:0] mem_addr;
    wire [15:0] mem_wdata;
    wire [15:0] mem_rdata;
    wire mem_wen;
    wire mem_do;
    fmc_bus u_fmc_bus (
        .sys_clk(clk),
        .sys_rst_n(rstn),

        .db(fmc_data),
        .wrn(fmc_wrn),
        .rdn(fmc_rdn),
        .csn(fmc_csn),
        .nadv(fmc_nadv),

        .ram_addr(mem_addr),
        .ram_wdata(mem_wdata),
        .ram_rdata(mem_rdata),
        .ram_wen(mem_wen),
        .ram_do(mem_do)
    );

    wire sram_valid;
    assign sram_valid = (mem_addr < 16'h8000);
    wire [15:0] sram_rdata;
    picorv32_ctrl u_picorv32_ctrl(
        .clk(clk),
        .rstn(rstn),
        .ram_addr(mem_addr),
        .ram_wdata(mem_wdata),
        .ram_wen(mem_wen),
        .ram_rdata(sram_rdata),
        .ram_do(mem_do & sram_valid)
    );

    assign mem_rdata =     sram_valid ? sram_rdata :
                        16'h0000;
endmodule
```

顶层模块采用分层设计：

**时钟子系统**：通过 FPGA 内置 PLL 生成 100MHz 稳定时钟，满足 RISC-V 核与 FMC 总线的时序要求。

**总线接口层**：FMC 模块实现异步总线协议转换，将 STM32 的 16 位 FMC 信号映射为 16 位内存接口，地址空间划分为前 64KB（RISC-V 程序区）和扩展区（预留未来功能）。

**核心控制层**：PicoRV32_CTRL 模块负责软核与内存的交互，通过双端口 RAM 实现 STM32（端口 B）与 RISC-V（端口 A）的并行访问。

### （二）RV32核心控制

```verilog
module picorv32_ctrl(
    input clk,
    input rstn,

    input [15:0] ram_addr,
    input [15:0] ram_wdata,
    input ram_wen,
    output [15:0] ram_rdata,
    input ram_do
);
    wire trap;
    wire mem_valid;
    wire mem_instr;

    reg mem_ready;
    wire [31:0]    mem_rdata;
    wire [31:0] mem_addr;
    wire [31:0] mem_wdata;
    wire [3:0] mem_wstrb;

    wire mem_la_read;
    wire mem_la_write;
    wire [31:0] mem_la_addr;
    wire [31:0] mem_la_wdata;
    wire [3:0] mem_la_wstrb;

    reg rv32_rstn;
    reg rv32_state;
    reg [15:0] rv32_ctrl_out;

    always@(posedge clk) begin
        if(~rstn) begin
            rv32_rstn <= 1'b0;
        end
        else begin
            rv32_rstn <= rv32_state;
        end
    end

    always@(posedge clk) begin 
        if(~rstn) begin
            rv32_state <= 1'b0;
        end else if(ram_do) begin
            case(ram_addr)
                16'h4001: begin
                    if(ram_wen)
                        rv32_state = ram_wdata[0];
                    else
                        rv32_ctrl_out <= {15'b0, rv32_state};
                end
                default: begin
                    rv32_ctrl_out <= 16'h0000;
                end
            endcase
        end
    end

    picorv32 #(.COMPRESSED_ISA(1))
    picorv32_core (
        .clk         (clk         ),
        .resetn      (rv32_rstn   ),
        .trap        (trap        ),
        .mem_valid   (mem_valid   ),
        .mem_instr   (mem_instr   ),
        .mem_ready   (mem_ready   ),
        .mem_addr    (mem_addr    ),
        .mem_wdata   (mem_wdata   ),
        .mem_wstrb   (mem_wstrb   ),
        .mem_rdata   (mem_rdata   ),
        .mem_la_read (mem_la_read ),
        .mem_la_write(mem_la_write),
        .mem_la_addr (mem_la_addr ),
        .mem_la_wdata(mem_la_wdata),
        .mem_la_wstrb(mem_la_wstrb)
    );

    wire [31:0] ram_32_rdata;
    wire [31:0] ram_32_wdata;
    wire [3:0] ram_byte_en;
    assign ram_byte_en = ram_addr[0] == 1'b0 ? 4'b0011 : 4'b1100;
    assign ram_32_wdata = {ram_wdata, ram_wdata};
    assign ram_rdata = (ram_addr[0] == 1'b0 && ram_addr[15:14] == 0) ? ram_32_rdata[15:0] :
                       (ram_addr[0] == 1'b1 && ram_addr[15:14] == 0) ? ram_32_rdata[31:16] :
                       rv32_ctrl_out;
    EF2_BRAM256 u_EF2_BRAM256(
        .doa(ram_32_rdata),
        .dia(ram_32_wdata),
        .cea(ram_do & ram_addr[15:14] == 0),
        .clka(clk),
        .wea(ram_wen),
        .rsta(~rstn),
        .addra(ram_addr[13:1]),
        .weabyte(ram_byte_en),

        .dob(mem_rdata),
        .dib(mem_la_wdata),
        .ceb(1'b1),
        .clkb(clk),
        .web(mem_la_write),
        .rstb(~rstn),
        .addrb(mem_la_addr[14:2]),
        .webbyte(mem_la_wstrb)
    );

    always @(posedge clk) begin
        if (~rstn) begin
            mem_ready <= 1'b0;
        end else begin
            mem_ready <= 1'b1;
        end
    end
endmodule
```

模块内例化了两个模块，分别是rv32内核模块和BRAM模块。picorv32模块通过一个简单的内存预取接口将其连接到32k字节的内存块。picorv32_ctrl提供的整个地址空间，前面32kb直连BRAM内存块，后面32kb连接内部实现的寄存器，目前寄存器只有一个，用于控制rv32内核的复位状态来控制内核的运行。

BRAM的内存宽度是32bit，前面FMC的内存宽度是16bit，所有这里做了一个巧妙的处理，通过地址低位来选择输入输出数据的高位还是低位。

picorv32默认没有开启指令压缩和乘法扩展，这里通过COMPRESSED_ISA手动打开指令压缩。为了节省一点LUT资源就不开乘法扩展了。在通过riscv-gcc编译程序时的架构选型应为-march=rv32ic。

### （三）FMC异步接口

这个接口代码应该是不复杂的，但我发现我写的代码看着很别扭，代码质量不高，所以就不展示了。😂😂😂

## 四、软件生态构建：从编译链到控制程序

### （一）RISC-V 测试程序开发

采用 MounRiver Studio 开发工具中集成的 RISC-V 工具链，编写包含解密功能的测试程序：

**启动汇编（start.S）**：初始化栈指针到 1KB 地址处，建立程序运行的基本环境：

```asm
.section .init
.global main
lui sp, %hi(0x00001000)       // 加载栈基址高12位
addi sp, sp, %lo(0x00001000)  // 生成完整32位地址
jal ra, main                  // 调用C主函数
ebreak                        // 程序结束
```

**C 语言逻辑**：实现 ROT13 解密算法，将混淆字符串转换为可读信息：

```c
volatile char *pmem = (void*)0x1000;     // 定义输出缓冲区
void putc(char c)
{
    *pmem++ = c;
}

void puts(const char *s)
{
    volatile int j;
    while (*s)
    {
        putc(*s++);
    }
}

void *memcpy(void *dest, const void *src, int n)
{
	while (n) 
	{
		n--;
		((char*)dest)[n] = ((char*)src)[n];
	}
	return dest;
}

void main()
{
    // 待解密的字符串
    char message[] = "$Uryyb+Jbeyq!+Vs+lbh+pna+ernq+guvf+zrffntr+gura$gur+CvpbEI32+PCH"
            "+frrzf+gb+or+jbexvat+whfg+svar.$$++++++++++++++++GRFG+CNFFRQ!$$";
    // ROT13解密逻辑
    for (int i = 0; message[i]; i++)
        switch (message[i])
        {
        case 'a' ... 'm':
        case 'A' ... 'M':
            message[i] += 13;
            break;
        case 'n' ... 'z':
        case 'N' ... 'Z':
            message[i] -= 13;
            break;
        case '$':
            message[i] = '\n';
            break;
        case '+':
            message[i] = ' ';
            break;
        }

    // 输出解密后的数据
    puts(message);

    while(1);
}
```

**内存链接脚本（firmware.lds）**：定义 32KB 统一编址空间，整合代码与数据：

```
MEMORY {
    mem : ORIGIN = 0x00000000, LENGTH = 0x8000
}

SECTIONS {
    .memory : {
        . = 0x000000;
        start*(.text);
        *(.init);
        *(.text);
        *(*);
        end = .;
        . = ALIGN(4);
    } > mem
}
```

### （二）STM32 控制程序实现

通过 FMC 总线实现固件加载与状态控制：

```c
#include <stdio.h>
#include "incfile.h"
INCFILE(fw, "main.bin");    // 嵌入二进制固件
int main(void)
{
  /* MCU Configuration--------------------------------------------------------*/
  MPU_Region_InitTypeDef MPU_InitStruct = {0};

  /* Disables the MPU */
  HAL_MPU_Disable();

  /* Configure the MPU attributes for the FMC */
  MPU_InitStruct.Enable           = MPU_REGION_ENABLE;
  MPU_InitStruct.Number           = MPU_REGION_NUMBER0;
  MPU_InitStruct.BaseAddress      = 0x60000000;
  MPU_InitStruct.Size             = MPU_REGION_SIZE_256MB;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.IsBufferable     = MPU_ACCESS_NOT_BUFFERABLE;
  MPU_InitStruct.IsCacheable      = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsShareable      = MPU_ACCESS_NOT_SHAREABLE;
  MPU_InitStruct.DisableExec      = MPU_INSTRUCTION_ACCESS_DISABLE;
  MPU_InitStruct.TypeExtField     = MPU_TEX_LEVEL0;
  MPU_InitStruct.SubRegionDisable = 0x00;
  HAL_MPU_ConfigRegion(&MPU_InitStruct);
  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* Configure the system clock */
  SystemClock_Config();

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_FMC_Init();        // 初始化FMC总线
  MX_USART1_UART_Init();// 初始化输出串口

#define SRAM_ADDR (0x60000000)
  HAL_Delay(1000);      // 等待FPGA程序上电复位完成
  uint16_t *addr = (void*)SRAM_ADDR;

  // 测试FMC内存读写
  for(int i = 0; i < 1024; i++){
      addr[i] = i;
  }
  for(int i = 0; i < 1024; i++){
      if(addr[i] != i){
          printf("sram error %d %d\r\n", i, addr[i]);
          break;
      }
  }
  printf("sram check complate\r\n");

  // 加载固件到FPGA
  const uint16_t *fw = FILEPTR(fw);
  int fw_size = FILESIZE(fw);
  for(int i = 0; i < fw_size/2+1; i++){
      addr[i] = fw[i];
  }
  printf("firmware load complate\r\n");

  // 控制RISC-V运行
  uint16_t *run_enable = (uint16_t*)(0x60000000 + (0x4001 << 1));
  *run_enable = 1; // 写入1启动软核
  HAL_Delay(1000);
  *run_enable = 0; // 写入0暂停软核

  // 读取并显示结果
  const uint16_t *msg = (uint16_t*)(0x60000000 + (0x1000));
  uint16_t buf[64];
  for(int i = 0; i < 64; i++){
      buf[i] = msg[i];
  }
  const char *msg_str = (void*)buf;
  printf("\r\n%s\r\n", msg_str);

  while (1)
  {
      HAL_Delay(500);
      printf("hello world\r\n");
  }
}
```

程序通过内存映射方式将 FPGA 的 ERAM 视为 STM32 的外部 SRAM，直接通过指针操作实现数据交互。控制寄存器（0x4001）的单个 bit 即可启动 / 停止 RISC-V 核，体现了内存映射接口的简洁性。

## 五、调试过程与关键验证

### （一）时序约束要点

**时钟配置**：对 PLL 生成的 100MHz 时钟进行时序约束，确保时钟抖动小于 500ps

**异步接口**：在 FMC 模块中使用寄存器打拍处理异步信号，避免亚稳态问题

### （二）测试结果验证

当 STM32 完成固件加载并启动 RISC-V 核后，串口终端显示：

```text
sram check complate
firmware load complate


Hello World! If you can read this message then
the PicoRV32 CPU seems to be working just fine.

                TEST PASSED!
```

这表明：

- FMC 总线通讯正常，数据读写无误
- RISC-V 软核正确执行了 ROT13 解密算法
- 双核心系统的协同控制机制有效 

### （三）学习RISC-V汇编

所有都正确执行后，再回过头看看RISC-V测试程序的反汇编代码，看看有没有因为编译优化而存在程序运行作弊的情况。

```asm
00000000 <init>:
   0:   00000137                lui     sp,0x0
   4:   40010113                addi    sp,sp,1024 # 400 <end+0x233>
   8:   0cc000ef                jal     ra,d4 <main>
   c:   9002                    ebreak

0000000e <putc>:
   e:   15802783                lw      a5,344(zero) # 158 <pmem>
  12:   00178693                addi    a3,a5,1
  16:   14d02c23                sw      a3,344(zero) # 158 <pmem>
  1a:   00a78023                sb      a0,0(a5)
  1e:   8082                    ret

...
000000d4 <main>:
  d4:   7175                    addi    sp,sp,-144
  d6:   08000613                li      a2,128
  da:   05400593                li      a1,84
  de:   850a                    mv      a0,sp
  e0:   c706                    sw      ra,140(sp)
  e2:   3fa9                    jal     3c <memcpy>
  e4:   870a                    mv      a4,sp
  e6:   04d00593                li      a1,77
...
00000158 <pmem>:
 158:   1000
```

看看putc函数的行为，从0x158地址取出数保存到a5寄存器，a5寄存器加1后保存到a3寄存器，再将a3寄存器的值写回到0x158地址处。最后将a0寄存器的值写入a5地址下实现数据输出。这里可以看到0x158地址下的初值就是0x1000，也就是pmem变量的初值。功能同C语言描述的一致，同时也表明代码和数据为统一编址，没有区分RAM和ROM。如果要区分ROM和RAM，需要添加一些启动代码来处理data段数据的拷贝。

前面RISC-V的C代码中有一个memcpy函数不知道你注意到没有，为什么需要这个函数呢？如果将这个函数删除掉，确实会导致无法编译通过。通过检查汇编才发现main函数确实会调用memcpy，根据汇编代码和map文件，main函数的逻辑是原静态字符串存储在rodata区，然后在栈上分配了一段内存来保存message数据，所以这里需要memcpy来处理一大段内存的拷贝。这也是同ARM汇编不一样的地方，学的新知识了😄

### （四）观察RISC-V的运行

通过FPGA开发工具提供的芯片调试工具，可以去捕获观测FPGA运行过程中寄存器的值。通过观测reg_pc和mem_rdata，更加直观的看到RISC-V的运行过程和内存取值的过程。发现每个指令执行的周期为1~7个时钟周期不等。jal跳转指令执行最快，仅需1个时钟周期。内存读写指令(sw)、条件跳转(bnez)、加法(addi)指令等需要5个时钟周期。

## 六、总结

回顾开发过程，学会了Verilog模块化设计的基本方法，尤其是内存接口的应用。此外还增强了对RISC-V的了解，实现了从源码、工具链再到硬件执行的完整流程。

回忆起我刚毕业参加工作，那时公司研发的项目基本上全采用ARM+FPGA的开发形式，ARM侧做纯粹的软件应用逻辑，FPGA控制底层的硬件。ARM和FPGA采用FMC接口进行通讯，通过内存映射的方式传输数据，这样ARM可以很方便的和FPGA进行通讯。当年由于能力有限，我并没有太深入的去了解这其中的通讯原理。主要是当时开发ARM的和开发FPGA的是两拨人，我主要负责ARM软件的开发，并没有机会接触到FPGA开发，后来觉得非常的遗憾。

通过这个小项目的开发经历也算是弥补了当年未能深入参与开发的一些遗憾。期待将项目中得到的经验应用到实际工作中，进一步探索FPGA在嵌入式领域的应用。
