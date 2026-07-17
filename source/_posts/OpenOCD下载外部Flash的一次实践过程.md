---
title: OpenOCD下载外部Flash的一次实践过程
author: WangGaojie
---

## 实现使用openocd配合cmsis-dap实现将固件下载到片外Flash

在使用STM32单片机时，使用调试器下载固件到外部Flash并不是一件很简单的事，因为单片机本身没有提供这样的功能。为了实现将固件下载到外部Flash，出现了一个东西叫下载算法，将一部分用于控制Flash的代码固件写入到RAM中，再配合调试来实现将固件传输并编程到外部Flash中。

常见的STlink和Jlink对于下载算法的格式和通讯协议都有固定的模板，虽然有模板格式可供使用，但在实践时仍然有一定的难度。而且换不同的下载器就要对应不同的下载算法，如果外部Flash的IO绑定关系变化了也要重新编译新的下载算法，所以在使用的灵活性上也较差。

## OpenOCD的方案
OpenOCD同样支持对STM32的外部Flash进行编程，但它不需要去编译下载算法，仅通过一个配置脚本就能打通与外部Flash的连接。

在配置脚本中需要做以下设置：
  - 设置调试适配器，例如stlink或CMSIS-DAP
  - 指定芯片型号
  - 通过配置芯片寄存器使能主要的时钟
  - 通过配置芯片寄存器完成GPIO的配置
  - 通过配置芯片寄存器完成QSPI\OSPI的初始化工作

其中前面两项很简单，参考openocd中的一些配置模板就能完成。

后面三项内容要稍微复杂一些，需要直接通过写寄存器的方式来让外设初始化完成并正常工作，所以部分需要有一定的耐心。

需要三条基本的openocd指令：
  - mww addr value：在指定的地址写入一个32位数，注意地址对齐
  - mmw addr setbits clearbits：通过两个掩码来操作指定地址下内存的值，方便单独设置或清除指定bit
  - mdw addr [count]: 读取指定内存的值，可选数据长度

依靠这三条指令就能完成对各个时钟外设的初始化工作，这部分工作一定要有耐心，对照芯片手册的寄存器表和C代码模板来完成寄存器的设定。

openocd中有部分示例cfg可以参考，但需要检查是否与自己的板子适配。

GPIO部分的配置有工具可以直接生成配置代码，工具在openocd中可以找到。https://github.com/openocd-org/openocd
```bash
perl gpio_conf_stm32.pl -c "PB06:AF10:V, PB02:AF09:V, PD13:AF09:V, PD12:AF09:V, PD11:AF09:V, PE02:AF09:V" -b 0x58020000
```
其中-b是设置GPIO外设的基地址，具体的值需要查芯片手册。
-c选项是设置与QSPI外设相关的GPIO配置。

格式是这样的：端口号:配置:速度
其中端口号就是STM32的端口号名称，配置有AFx(复用功能，例如AF10)、PP(推挽输出)、PO(开漏输出)、INUP(上拉输入)、INDO(下拉输入)，速度有L(低速)、M(中速)、H(高速)、V(超高速)

上面这段代码，生成了如下的配置：
```
        # PB06:AF10:V, PB02:AF09:V, PD13:AF09:V, PD12:AF09:V, PD11:AF09:V, PE02:AF09:V
        # Port B: PB06:AF10:V, PB02:AF09:V
        mmw 0x58020400 0x00002020 0x00001010    ;# MODER
        mmw 0x58020408 0x00003030 0x00000000    ;# OSPEEDR
        mmw 0x5802040C 0x00000000 0x00003030    ;# PUPDR
        mmw 0x58020420 0x0A000900 0x05000600    ;# AFRL
        # Port D: PD13:AF09:V, PD12:AF09:V, PD11:AF09:V
        mmw 0x58020C00 0x0A800000 0x05400000    ;# MODER
        mmw 0x58020C08 0x0FC00000 0x00000000    ;# OSPEEDR
        mmw 0x58020C0C 0x00000000 0x0FC00000    ;# PUPDR
        mmw 0x58020C24 0x00999000 0x00666000    ;# AFRH
        # Port E: PE02:AF09:V
        mmw 0x58021000 0x00000020 0x00000010    ;# MODER
        mmw 0x58021008 0x00000030 0x00000000    ;# OSPEEDR
        mmw 0x5802100C 0x00000000 0x00000030    ;# PUPDR
        mmw 0x58021020 0x00000900 0x00000600    ;# AFRL
```

QSPI的配置是最麻烦的，需要配置QSPI外设，初始化外部Flash芯片，正常开启映射模式后就能使用了。

openocd启动后，执行reset init命令可以触发执行自定义的寄存器配置。

这里需要注意的时，需要根据实际的Flash型号来设置一些参数、比如Flash大小(FSIZE)、地址大小(ADSIZE)等，一项不正确就会导致无法读写外部Flash。

我在最后开启Flash Memory Mapped卡了一段时间，最终排查就是ADSIZE和DCYC配置不当导致的，我的w25q64jv仅支持3字节地址模式，但是我在4字节地址模式下调试了很久，数据能读取，但是无法执行擦除操作。后面意识到ADSIZE配置错误，修正后数据读取又出错了，但是观察读取到的数据很明显的DCYC配置不正确导致的，修复后就能正常工作了。

按照w25q64jv手册的意思，芯片上电或复位后的DCYC=2，但是我需要配置为6才能正常工作，我不知道是不是复位指令没生效，但是我在我的Bootloader中确实通过0xC0指令修改过Read Parameters。这部分内容还需要仔细核对一遍。

Reset指令如果按照4线模式再执行一次，DCYC需要设置为0才能正常工作。这又是为什么呢？这说明复位指令已经生效了。那为什么在w25qxx中设置的dummy clocks和QSPI中的DCYC始终差2呢？

## 附上我的openocd脚本配置
```
# This is for using CMSIS-DAP
source [find interface/cmsis-dap.cfg]
transport select swd
set CHIPNAME stm32h750xx

set QUADSPI  1
set OCTOSPI1 0
set OCTOSPI2 0

source [find target/stm32h7x.cfg]
reset_config none separate

proc qspi_init { qpi } {
    mmw 0x580244E0 0x000007FF 0             ;# RCC_AHB4ENR |= GPIOAEN-GPIOKEN (enable clocks)
    mmw 0x580244D4 0x00004000 0             ;# RCC_AHB3ENR |= QSPIEN (enable clock)
    sleep 1                                 ;# Wait for clock startup

    # PB06:AF10:V, PB02:AF09:V, PD13:AF09:V, PD12:AF09:V, PD11:AF09:V, PE02:AF09:V

    # Port B: PB06:AF10:V, PB02:AF09:V
    mmw 0x58020400 0x00002020 0x00001010    ;# MODER
    mmw 0x58020408 0x00003030 0x00000000    ;# OSPEEDR
    mmw 0x5802040C 0x00000000 0x00003030    ;# PUPDR
    mmw 0x58020420 0x0A000900 0x05000600    ;# AFRL
    # Port D: PD13:AF09:V, PD12:AF09:V, PD11:AF09:V
    mmw 0x58020C00 0x0A800000 0x05400000    ;# MODER
    mmw 0x58020C08 0x0FC00000 0x00000000    ;# OSPEEDR
    mmw 0x58020C0C 0x00000000 0x0FC00000    ;# PUPDR
    mmw 0x58020C24 0x00999000 0x00666000    ;# AFRH
    # Port E: PE02:AF09:V
    mmw 0x58021000 0x00000020 0x00000010    ;# MODER
    mmw 0x58021008 0x00000030 0x00000000    ;# OSPEEDR
    mmw 0x5802100C 0x00000000 0x00000030    ;# PUPDR
    mmw 0x58021020 0x00000900 0x00000600    ;# AFRL

    # for single flash mode w25q64jv
    mww 0x52005000 0x05500018               ;# QUADSPI_CR: PRESCALER=5, APMS=1, FTHRES=1, FSEL=0, DFM=0, SSHIFT=1, TCEN=1
    mww 0x52005004 0x00160701               ;# QUADSPI_DCR: FSIZE=0x16, CSHT=0x05, CKMODE=0
                                            ;# FSIZE flash的大小。

    mww 0x52005030 0x00001000               ;# QUADSPI_LPTR: deactivate CS after 4096 clocks when FIFO is full
                                            ;# 11010000000000 10 010100000011
    mww 0x52005014 0x0D002503               ;# QUADSPI_CCR: FMODE=0x3, DMODE=0x1, DCYC=0x0, ADSIZE=0x2, ADMODE=0x1, IMODE=0x1
    mmw 0x52005000 0x00000001 0             ;# QUADSPI_CR: EN=1

    # Exit QPI mode
    #mmw 0x52005000 0x00000002 0            ;# QUADSPI_CR: ABORT=1
    #mww 0x52005014 0x000003FF              ;# QUADSPI_CCR: FMODE=0x0, DMODE=0x0, DCYC=0x0, ADSIZE=0x0, ADMODE=0x0, IMODE=0x3, INSTR=Exit QPI
    sleep 1

    # Reset flash
    mmw 0x52005000 0x00000002 0             ;# QUADSPI_CR: ABORT=1
    mww 0x52005014 0x00000166               ;# QUADSPI_CCR: FMODE=0x0, DMODE=0x0, DCYC=0x0, ADSIZE=0x0, ADMODE=0x0, IMODE=0x1, INSTR=0x66
    mww 0x52005014 0x00000199               ;# QUADSPI_CCR: FMODE=0x0, DMODE=0x0, DCYC=0x0, ADSIZE=0x0, ADMODE=0x0, IMODE=0x1, INSTR=0x99
    sleep 1

    # Enter QPI mode
    mmw 0x52005000 0x00000002 0             ;# QUADSPI_CR: ABORT=1
    mww 0x52005014 0x00000138               ;# QUADSPI_CCR: FMODE=0x0, DMODE=0x0, DCYC=0x0, ADSIZE=0x0, ADMODE=0x0, IMODE=0x1, INSTR=Enter QPI
    sleep 1

    # Do need set read parameters? INSTR=0xC0
    # TODO

    # memory-mapped read mode with 3-byte addresses
    mmw 0x52005000 0x00000002 0             ;# QUADSPI_CR: ABORT=1
    mww 0x5200501C 0x000000EF               ;# QUADSPI_ABR: AlternateBytes=0xEF
    mww 0x52005014 0x0F18EFEB               ;# QUADSPI_CCR: FMODE=0x3, DMODE=0x3, DCYC=0x6, ADSIZE=0x3, ADMODE=0x3, IMODE=0x3, INSTR=Fast Read Quad I/O 
}

$_CHIPNAME.cpu0 configure -event reset-init {
    global QUADSPI
    mmw 0x52002000 0x00000004 0x0000000B    ;# FLASH_ACR: 4 WS for 192 MHZ HCLK
    mmw 0x58024400 0x00000001 0x00000018    ;# RCC_CR: HSIDIV=1, HSI on
    mmw 0x58024410 0x10000000 0xEE000007    ;# RCC_CFGR: MCO2=system, MCO2PRE=8, HSI as system clock
    mww 0x58024418 0x00000040               ;# RCC_D1CFGR: D1CPRE=1, D1PPRE=2, HPRE=1
    mww 0x5802441C 0x00000440               ;# RCC_D2CFGR: D2PPRE2=2, D2PPRE1=2
    mww 0x58024420 0x00000040               ;# RCC_D3CFGR: D3PPRE=2
    mww 0x58024428 0x00000040               ;# RCC_PPLCKSELR: DIVM3=0, DIVM2=0, DIVM1=4, PLLSRC=HSI
    mmw 0x5802442C 0x0001000C 0x00000002    ;# RCC_PLLCFGR: PLL1RGE=8MHz to 16MHz, PLL1VCOSEL=wide
    mww 0x58024430 0x01070217               ;# RCC_PLL1DIVR: 192 MHz: DIVR1=2, DIVQ=8, DIVP1=2, DIVN1=24
    mmw 0x58024400 0x01000000 0             ;# RCC_CR: PLL1ON=1
    sleep 1
    mmw 0x58024410 0x00000003 0             ;# RCC_CFGR: PLL1 as system clock
    sleep 1

    adapter speed 4000

    echo "start"

    if { $QUADSPI } {
        qspi_init 1
        flash probe 1
    }
}
```

我用的测试开发板是WeActStudio的STM32H750Mini。
