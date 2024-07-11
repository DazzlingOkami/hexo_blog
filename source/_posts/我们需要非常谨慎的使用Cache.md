---
title: 我们需要非常谨慎的使用Cache
author: WangGaojie
---

最近在学习的过程中遇到一个软件上的问题，该问题非常折磨人。

在一个嵌入式软件系统中使用WiFi网络并通过FTP服务读取设备内存储在SD上的文件，就是这样一个看起来非常简单的逻辑，最近在测试过程中出现了一点问题。

发现小概率情况下文件传输会导致系统崩溃，崩溃的问题点也是非常奇怪的，会爆发出很多稀奇古怪的问题(包括但不限于malloc返回null、内存申请卡死、网络协议栈数据异常等等)，且该故障似乎很难在调试模式下复现。

该故障仅在上述条件下会触发，其它情况下没有复现：
- 通过FTP读取内存中的文件(ramfs)不会导致系统崩溃
- 通过USB连接网络，仍然通FTP传输SD卡上的文件，系统不会崩溃
- 在系统内通过文件拷贝方法在SD和nand直接传输文件，系统不会崩溃
- 关闭系统D-Cache，使用wifi通过FTP读取SD卡的文件，系统不会崩溃

分析以上情况，好像很难确定问题点。但该故障大概率是与Cache相关的，因为这可以解释为什么在调试模式下问题不那么容易复现。

整个系统相当复杂，底层磁盘驱动、文件系统、文件系统抽象层、网络协议栈、FTP服务、WIFI驱动、操作系统等等。我花费了相当多的额时间去梳理WIFI驱动代码，这是我最不熟悉的一部分，但是WIIF驱动看来没有明显的问题。

其中多个与硬件相关的组件均会使用到DMA和Dcache的操作，好像没有办法把问题聚焦到一个点上。

系统出现故障时的一种常见现象是内存分配器出现故障，包括无法分配内存(但显示存在剩余内存)、无法释放内存等。

我设计了一种检查机制，即在内存分配和释放的时候检查一下内存分配器的链表释放正常。该方法确实有用，它能够迅速的捕获系统最近出现故障的时刻，这使得复现和分析问题非常有帮助。
```c
vTaskSuspendAll();
{
    int tot_len = 0;
    BlockLink_t * pxBlock;
    pxBlock = heapPROTECT_BLOCK_POINTER( xStart.pxNextFreeBlock );
    while( pxBlock != pxEnd ){
        tot_len += pxBlock->xBlockSize;
        pxBlock = heapPROTECT_BLOCK_POINTER( pxBlock->pxNextFreeBlock );
    }

    configASSERT( tot_len == xFreeBytesRemaining );
}
( void ) xTaskResumeAll();
```
当发现系统内剩余内存和链表中描述的剩余内存不一致后立即中断系统运行。

配合内存分配日志和Cache操作日志，确定是SD卡驱动的一次INVALID DCACHE操作后内存分配器就出现故障了。

```c
int sdmmc_read(void *buf, int len){
    // 1:
    CLEAN_DCACHE(buf, len);

    // 2:
    SD_ReadBlocks_DMA(buf, len);

    // 3:
    INVALID_DCACHE(buf, len);

    return 0;
}
```

解释一下这段代码的含义。第3行代码，当使用DMA读取数据后，需要将缓存中的数据清除掉，这是为了后续CPU读取数据时可以读取到物理内存中的数据，而非Cache中的数据。第1行代码，这段代码有点不太好解释，当buf和Cache边界未对齐时，后续的第三步可能会使得buf边界外到Cache边界中的数据被清除掉，这就使得系统内的数据完整性无法得到保证，所以设计了第1步的代码来将附近的数据全部同步到内存中。

以上的设计在我以前的文章中提到过，可以去翻看一下。

这样的设计在单线程环境下是没有问题的，但是在多线程下就会有点问题，因为第2行代码是一个耗时操作，这就使得已经被同步到内存中的数据可能又被另一个线程取出放入Cache，而后续的第3行代码还不知道这样的情况，直接将缓存清除掉了，导致系统内部分数据丢失，进而导致出现系统崩溃的情况。

一个安全的做法是为DMA读取操作单独分配一块与Cache边界对齐的一块缓存，用它作为数据传输的桥梁，由DMA读取完成后再由CPU将数据搬运到其它内存空间下。但是这样有一些缺陷，即无法连续读取多块数据，这会导致SDMMC出现严重的性能瓶颈，一般SDMMC连续读取多块扇区有更明显的速度优势。

所以需要设计另外一种方法来安全的清理Cache，且必须考虑线程安全问题。
```c
#define CACHE_SIZE (32)
void InvalidDCache(void *addr, int len){
    __disable_irq();
    static char cache_head_buf[CACHE_SIZE - 1];
    static char cache_end_buf[CACHE_SIZE - 1];
    uint32_t cache_head_addr;
    uint32_t cache_head_len;
    uint32_t cache_end_addr;
    uint32_t cache_end_len;

    cache_head_addr = ((uint32_t)(addr) & ~(CACHE_SIZE - 1));
    cache_head_len = ((uint32_t)(addr) & (CACHE_SIZE - 1));
    cache_end_addr = ((uint32_t)(addr) + len);
    cache_end_len = CACHE_SIZE - (cache_end_addr & (CACHE_SIZE - 1));
    if(cache_end_len == CACHE_SIZE){
        cache_end_len = 0;
    }

    if(cache_head_len > 0){
        memcpy(cache_head_buf, cache_head_addr, cache_head_len);
    }
    if(cache_end_len > 0){
        memcpy(cache_end_buf, cache_end_addr, cache_end_len);
    }

    SCB_InvalidateDCache_by_Addr(addr, len);

    if(cache_head_len > 0){
        memcpy(cache_head_addr, cache_head_buf, cache_head_len);
    }
    if(cache_end_len > 0){
        memcpy(cache_end_addr, cache_end_buf, cache_end_len);
    }
    __enable_irq();
}
```
这段代码可以解决缓存Invalidate带来的风险，它将边界上的内存主动保存了一遍。

但还有一个问题似乎不可忽略，那就是缓存自动写入的情况，当切换线程上下文后执行了其它的内存密集操作，当缓存不足时，可能导致原先的缓存被自动写入到内存中，尤其是边界上的，这仍然可能出现数据失效的问题。

最终，我选择牺牲性能来保证可靠性，通过一个中间缓存来读取数据，确保DMA和Cache的操作都是安全可靠的。当然在实现过程中可以通过检查输入的内存指针是否与Cache边界对齐，如果是对齐的情况，那么就可以绕过中间缓存直接读取数据。

Cache的问题总是那么有意思。最好的情况还是应该在软件设计时就能够保证数据边界对齐，或者牺牲一定的性能来保证数据的可靠性和系统安全性。

PS：为什么wifi传输数据异常而USB网卡传输数据正常呢？因为USB网卡驱动内部没有频繁的内存申请操作，而wifi设备驱动中有频繁的内存申请操作。出现Cache问题的那块内存正好是从堆内获取的，且故障边界正好命中内存管理器的空闲链表数据项。所以频繁的内存申请操作会导致该问题更容易复现。
