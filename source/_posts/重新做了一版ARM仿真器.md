---
title: 重新做了一版ARM仿真器
author: WangGaojie
---

## 简介

我最近又重新设计制作的了一款ARM仿真器，相比于之前做的版本，增加了一块屏幕，并使用编码器拨轮进行交互，还内置了一块电池，可以实现脱机程序烧录。

下面是主要配置：

- 160x128 像素分辨率的1.8寸屏幕

- RP2350 MCU

- 内置16MB存储空间+8MB扩展运行内存+32GB TF卡支持

- 900mAh/3.7V 锂电池

- 一路UART接口

- 一路SWD接口，支持20MHz速率

- 可对外提供3.3v/1A供电

支持的主要功能有：

- CMSIS-DAP调试器功能

- GDB调试器功能

- 脱机下载功能，支持上百款MCU

- 脱机串口功能，可在屏幕上实时显示串口接收到的数据

- 设备内文件管理功能

- 模拟U盘功能

## 开发过程

### 1.硬件部分

其实上一版开发完成后，我就准备着手这个新版本的开发了，主要是上一个版本的功能亮点不足，最终我决定再进行一次大的升级。主要是围绕设备能够脱机使用，添加了脱机下载和脱机串口功能。

首先是硬件开发方面，将MCU由RP2040更换为RP2350，其主频、SRAM、IO数量都做了升级。选择该芯片的原因是RP2350芯片的价格已经比较实惠了且性能有了比较大的提升，最关键的是由于增加了不少外设(屏幕、存储等)使得IO分配比较紧张。

该MCU还支持外扩PSRAM，所以加了一颗AP6404L的芯片，扩展SRAM主要是为了提高脱机下载时的速度，有了充足的SRAM就可以将固件全部加载到内存中进行再下载。

在存储方面，没有直接使用存放固件的flash来存储数据，而是单独添加了一块norflash芯片，还添加了TF卡的卡座，norflash和tf卡共用一条SPI进行数据传输。屏幕为一块ST7735主控的LCD屏幕，还是使用SPI进行数据传输，为了保证屏幕刷新和存储数据的读写，屏幕和存储芯片各自使用一个独立的SPI。

拨轮是SIQ-02FVS3，为AB编码器接口，原本计划使用PIO驱动，但是开发软件时才发现IO分配不合理导致只能使用中断驱动。

电源方面的电路变化不大，主要是增加了充电芯片和开关机管理电路，充电芯片为TP4057，对于这个简单的设备来说，暂时不需要快充等功能。开关机管理使用一些MOS管进行搭建，如下图：

<center>

![开关电路](power_switch.png)
*开关电路*
</center>

开关机按键复用编码器拨轮的按键，器件选择上主要是要考虑电流参数。

在这一版中，外部的SWD接口和UART接口上都添加了ESD保护芯片。这样外部的包括USB在内的三个插拔接口都有了防静电保护。

来看看PCB：
![PCB](PCB.png)

拨轮的安装比较特殊，在PCB上挖了一个洞，使得波轮能够沉到底部，在背部焊接。如果正常安装焊接，它将是整个PCB上最高的元件。现在这种安装方式可以减小整个PCB的厚度。
![PCB](encoder.png)

PCB上电池座子的+-符号标识反了，但是不影响正常使用。所有的元件都是用风枪焊上去的，焊接效果还比较满意，这次温度把控的比较好，塑料件没有任何变形。

*Tips:MCU旁边有颗电容尺寸不对，原本应该用0402的封装，但是该容值的电容手边没有，所以用0603的封装*

### 2.软件部分

硬件弄好后就是软件开发工作，在软件开发方面，首先是需要将主要的外设驱动开发完成，包括屏幕、NorFlash、TF卡，编码器等。

ST7735驱动都有现成比较好用的，主要是要考虑异步渲染和刷新的性能问题，其它就是调试一下屏幕边界和颜色翻转就完成了，淘宝上5块钱买的屏幕，效果只能说一般般。

TF卡的驱动要麻烦一些，RP2350没有SDIO接口，用PIO模拟目前也不会。只能使用SPI进行通讯了，时钟速度20MHz，NorFlash和它共用SPI，NorFlash速度本可以再高一些，但是只能兼顾TF的速度了，速度再高一些TF卡似乎存在一些问题。TF卡使用FatFS驱动，文件读写可以在400kB/s左右，NorFlash使用LittleFs读取速度要快一些。

ST7735的SPI采用DMA传输数据，考虑到使用DMA会到了额外的中断上下文切换和线程切换，对于小于15字节的传输任然采用阻塞传输会比较好。通过后续测试发现部分页面上的全屏动效运行起来的CPU占用也就不到50%。

对于W25Q128和TF卡共用的SPI来说，使用DMA要稍微复杂一些。这里我也是摸索了好久才找到正确的写法，RP2350的SPI必须要有TX的数据才能完成正常的一次数据传输，所以最开始我只写了DMA接收，发现DMA并没有启动传输，那是因为SPI没有发出数据，自然也就无法接收数据，所以数据收发必须成对，即使用DMA也必须要这样。对于数据接收来说，我是这样写的：

```c
int spi_shard_read(uint8_t *buf, int len){
    int ret;
    if(len <= 15){
        ret = spi_read_blocking(SPI_INS, 0xff, buf, len);
    }else{
        uint8_t tx_dummy = 0xff;

        /* DMA read dummy data Not increment */
        dam_channel_read_increment(dma_tx_chan, false);

        /* Read the data from the SPI FIFO by DMA */
        dma_channel_transfer_to_buffer_now(dma_rx_chan, buf, len);

        /* Write the dummy data to the SPI FIFO by DMA, Used to generate clock signals */
        dma_channel_transfer_from_buffer_now(dma_tx_chan, &tx_dummy, len);

        /* Wait Rx DMA to complete */
        if(xSemaphoreTake(spi_dma_rx_sync, pdMS_TO_TICKS(200)) == pdFALSE){
            ret = 0;
        }else{
            ret = len;
        }

        /* Recover Tx channel auto increment */
        dam_channel_read_increment(dma_tx_chan, true);

        /* Clear the Tx sync semaphore */
        xSemaphoreTake(spi_dma_tx_sync, 0);
    }
    return ret;
}
```

创建了两个DMA请求，但是Tx发送的数据临时关闭了地址自增，因为发送的数据本就是dummy，所以不需要地址自增。DMA请求发出后，DMA就会自动从SPI的FIFO中将数据读取出来，等待DMA中断发出的同步信号就表示数据传输完成。

对于SPI的DMA发送来说，也有一些情况需要处理：

```c
int spi_shard_write(const uint8_t *buf, int len){
    int ret;
    if(len <= 15){
        ret = spi_write_blocking(SPI_INS, buf, len);
    }else{
        /* Write the date to the SPI FIFO by DMA */
        dma_channel_transfer_from_buffer_now(dma_tx_chan, buf, len);

        /* Wait for the DMA to complete */
        if(xSemaphoreTake(spi_dma_tx_sync, pdMS_TO_TICKS(200)) == pdFALSE){
            ret = 0;
        }else{
            ret = len;
        }

        /* Wait for the SPI to complete */
        while(spi_is_busy(SPI_INS)){
            if(spi_is_readable(SPI_INS)){
                (void)spi_get_hw(SPI_INS)->dr;
            }
            taskYIELD();
        }

        /* Clear the RX FIFO and clear the overrun flag */
        while(spi_is_readable(SPI_INS))
            (void)spi_get_hw(SPI_INS)->dr;
        spi_get_hw(SPI_INS)->icr = SPI_SSPICR_RORIC_BITS;
    }
    return ret;
}
```

这里只创建DMA数据发送请求即可，SPI接收回来到RxFIFO中的数据可能会溢出，但是可以忽略它们。需要注意的是，DMA传输完成不代表SPI已经传输完成，因为SPI内部是带有FIFO的，这里简单的用spi_is_busy来判断SPI是否传输完成，传输完成后再将RxFIFO中的数据清空以避免对后续的数据读取产生干扰。这里更严格的写法是使用的SPI的RxFIFO为空中断去处理SPI等待的问题，不过考虑到FIFO本身并不大且SPI的时钟速率较高，这里阻塞的时间可以忽略。

看门狗如何检测两个MCU核心都在正常工作呢？刚开始就出现系统的一个核心已经死机了，但是喂狗的操作在另一个核心上，导致系统无法及时复位。后面我想到了一个比较好的办法：

```c
static void watchdog_task(void *ptr){
    if (watchdog_caused_reboot()){
        reset_usb_boot(0, 0);
    }

    watchdog_enable(1000, 1);
    while(1){
#if (configNUMBER_OF_CORES > 1)
        /* Update watchdog on all cores */
        for(int i = 0; i < configNUMBER_OF_CORES; i++){
            vTaskCoreAffinitySet(NULL, 1 << i);
            watchdog_update();
            vTaskDelay(pdMS_TO_TICKS(500));
        }
#else
        watchdog_update();
        vTaskDelay(pdMS_TO_TICKS(500));
#endif
    }
}
```

通过FreeRTOS创建一个最低优先级的任务(比IDLE线程优先级高)来运行看门狗。看门狗的超时时长设置为1秒，该任务准备每500ms进行一次喂狗操作，即使高优先级的任务有时间片抢占的情况，还有500ms的冗余空间。这里比较取巧的是我每次喂狗完成后就将任务切换到下一个核心上运行，这样就确保能够对所有的核心都起到监控的作用。

SWD那部分的布线或者电路可能还需要优化，芯片和软件支持的最高速率是37.5MHz，而实际上在30MHz时钟下工作会出现无法识别外部芯片的问题，目前最高只能工作在20MHz，我怀疑是走线不规范或者阻抗相关的问题，如果要做下一版再去着重解决。

SWD的底层采用PIO来驱动，时序上肯定没有问题，但是目前遇到一个与PIO交互的性能问题，与PIO交互是通过一个硬件FIFO实现的，每次遇到FIFO为空或满时都需要进行阻塞式的等待，这会使得低优先级的任务被长时间阻塞。如果采用中断信号通知FIFO状态，我担心频繁的中断信号也会带来性能问题，这个地方或许还要好好想想怎么优化，或许需要大改底层的SWD实现方法。

在软件功能设计上，CMSIS-DAP和GDB调试功能在上一个版本就已经开发完成了，本次主要的开发工作都是围绕GUI相关功能进行的。GUI部分采用LVGL9图形框架，搭配GUI-Guider进行开发，目前GUI-Guider还不支持LVGL9，所以UI的开发工作还是比较多。

在GUI上除了设计UI耗费时间外，文件管理功能和脱机下载功能耗费了比较多的开发时间，NorFlash和TF工作在两套不同的文件系统下，要实现文件的统一管理，必须设计更上层的抽象接口层。TF卡还需要允许通过USB挂载到电脑上，其中的功能细节也比较多。为了获得最佳的下载速度，我将固件从文件中读取并解析完成后全部存储在RAM中，然后再进行下载，这对于HEX类固件来说可以显著提高下载速度。

目前主要支持hex格式的固件下载，bin格式的固件只能下载到芯片内Flash首地址上，还不支持自定义，主要是设备没有键盘和触摸屏，不方便输入。对于elf格式的固件还在准备开发中。

分享一个开发工程中遇到的一个竞态问题，一个很有意思的问题。该问题出现在USB串口功能中，我创建了一个独立的线程用于将硬件UART上收发到的数据传输到USB协议栈中。问题现象是USB协议栈收到电脑发来的数据后从硬件串口发出后异常，小概率出现数据中多了一段乱码的数据。导致该问题的原因是在UART的DMA中断处理函数中出现了重入，导致对FIFO的操作出现了抢占，导致FIFO内的数据长度异常。为什么中断函数会重入？因为RP2350是一个双核MCU，它允许中断同时在两个核心上运行，根据PICO-SDK的手册来看irq_set_exclusive_handler()和irq_set_enabled()函数都是只对MCU当前的核心生效，为什么中断会同时运行在两个核心上？经过仔细的分析，首先解释第一点，为什么该中断函数通过irq_set_exclusive_handler会同时将中断绑定两个核心上，根据芯片手册，两个核心拥有独立的中断向量表。但是这里我忽略了一点，第二个核心是由FreeRTOS启动，它默认直接复用了核心1的中断向量表，导致两个核心将共享中断向量表。

FreeRTOS对于RP2350的port文件：

```c
BaseType_t xPortStartScheduler( void )
{
    ...
    multicore_reset_core1();
    multicore_launch_core1( prvDisableInterruptsAndPortStartSchedulerOnCore );
    xPortStartSchedulerOnCore();

    /* Should not get here! */
    return 0;
}
```

启动第二给核心的函数是multicore_launch_core1(),在它内部的实现上会复用当前中断向量表地址。如果要给第二个核心赋予独立的中断详表，应该使用multicore_launch_core1_raw接口。到了这里就明白了为什么第二个核心会存在中断函数。再解释第二点，为什么该中断被使能了。

在创建UART任务时是这样写的：

```c
xTaskCreate(cdc_thread, "UART", configMINIMAL_STACK_SIZE, NULL, UART_TASK_PRIO, &uart_taskhandle);
#if (configNUMBER_OF_CORES > 1)
vTaskCoreAffinitySet(uart_taskhandle, 1 << 0);
#endif
```

可以看到已经通过vTaskCoreAffinitySet函数将该任务绑定到了核心0上，但是我忽略了一点，由于当前任务的优先级比创建的UART任务优先级低，导致UART任务创建完成后就立即执行，而任务函数内进行中DMA中的开关操作，导致核心1上的DMA中断也打开了。这是在极短时间内出现的。这是一种非常典型的竟态问题，修复的手段也非常简单，就是在串口任务的入口处，将任务本身绑定到和核心0上。再说一下为什么在任务内进行了中断的开关操作，因为FIFO数据接口的操作需要锁定，理论上来说刚开机初始化时没有数据收发才对，但由于串口刚开始莫名奇妙收到一个字符，就是这一连串的问题导致了这个bug。分析这个问题至少花了大半天时间，可以看看这个[commit](https://github.com/DazzlingOkami/debugprobe/commit/b175dbc9ad1e485d05a44415c197c272882e007b)。

RP2350+FreeRTOS的低功耗模式也有进行测试，开启configUSE_TICKLESS_IDLE是必要的，这样运行systick的核心0可以正常进入休眠状态。如果第二个核心也运行了，那还需要开启configUSE_PASSIVE_IDLE_HOOK，让第二个核心通过WFI指令进入睡眠状态。只有两个核心都进入休眠，RP2350才能进入低功耗模式。开启低功耗模式后，系统功耗有一定降低，但是我我的设备功耗本身不高，可以轻松待机10小时以上，所以低功耗模式是没有开启的。

### 3.外壳部分

我使用UG为整个设备设计了一个外壳，好像很长一段时间没用UG建模，感觉不是很熟练了😅。

| ![](Forface3dModelTop.png) | ![](Forface3dModelBottom.png) |
| -------------------------- | ----------------------------- |

本次设计的PCB和外壳都是在嘉立创制作的，3D打印的外壳还比较便宜，原计划使用CNC做一个金属的外壳，不过看了价格，还是放弃了。3D打印只要5块钱，CNC应该不低于300元，价格劝退，外壳上的结构特征基本上都是为了方便CNC加工而设计的。

## 功能展示

原本打算拍个视频来展示一下这个设备的功能，后面感觉拍不好就算了。我在设备内做了一个截图功能，就截取一些操作界面进行展示吧。

<center>
<table>
    <tr>
        <td><center><img src="1.bmp"/><br>1.主界面</center></td>
        <td><center><img src="2.bmp"/><br>2.一个列表式的文件管理器</center></td>
        <td><center><img src="3.bmp"/><br>3.串口接收界面</center></td>
        <td><center><img src="4.bmp"/><br>4.设置界面</center></td>
    </tr>
    <tr>
        <td><center><img src="5.bmp"/><br>5.固件下载过程</center></td>
        <td><center><img src="6.bmp"/><br>6.下载完成界面 1MHz时钟</center></td>
        <td><center><img src="7.bmp"/><br>7.下载完成界面 20MHz时钟</center></td>
        <td><center><img src="8.bmp"/><br>8.添加了一个小游戏😄</center></td>
    </tr>
</table>
</center>

下载一个92k字节的固件，除去擦除时间，下载时间只有388ms，速度非常块。

再来看看设备运行起来后MCU的状态：

```text
CPU freq:150MHz
Stat Period: 3000000us
ID   Name         CPU State Priority  Stack       MSP   Static SFREE  SC       DelayTime
 1 DAP           0.00%  S      3    0x2002b7d0 0x2002bb58  D   224    0
 2 TUD           1.11%  B      2    0x2002c6a0 0x2002ca30  D   182    3001      1
 3 UART          0.30%  B      3    0x2002cb30 0x2002cea8  D   210    600       2
 4 BMP           0.09%  X      3    0x2002cfc0 0x2002dc98  D   767    30
 5 IDLE0        98.58%  R      0    0x2002e050 0x2002e3d8  D   212    2993
 6 IDLE1        98.55%  X      0    0x2002e4e0 0x2002e888  D   228    2992
 7 Tmr Svc       0.00%  B     31    0x2002ea70 0x2002f9e0  D   986    0         -1
 8 GUI           0.77%  B      1    0x200301a8 0x200310b8  D   326    156       5
Task scheduler CPU percent -- 0.57%
MEM: 23304/153600 (15%)
```

正常空闲状态下，CPU的占用都比较低，USB在传输数据时会占用一部分CPU。第一次使用FreeRTOS的双核特性，两个核心的调度还是比较合理的，即使GUI线程的CPU达到90%以上后，两个空闲线程的CPU占比都能够做到均匀分配。设备功耗不到0.2W，独立工作也能有较长的续航时间。

最后来看看组装好的效果：

![组装图](device.png)

最后还需要在上面盖一块亚克力的面板，这样就能将四个螺丝隐藏掉，整体外观效果就会好很多了。
![组装图](device2.png)
