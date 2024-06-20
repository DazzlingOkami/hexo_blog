---
title: 第一次向FreeRTOS内核提交代码
author: WangGaojie
---

## 背景
前段时间我在测试[wolfSSL](https://www.wolfssl.com/)中RSA相关代码的时候，发现在进行加密、解密、key生成过程中存在一些性能问题。在分析问题的过程中，发现算法对堆内存的申请非常频繁。生成一对2048长度的公钥和私钥，会进行10万次以上的内存申请。我为了跟踪函数在执行过程中的内存具体使用情况，决定设计一套堆内存跟踪机制来评估函数的内存使用情况。

在设计内存跟踪机制的过程中，我发现了一个FreeRTOS内核中的Bug，最终该Bug由我亲自修复，相关修复代码已经合并入内核代码主线。

## 设计一个内存跟踪机制
原本想设计一个不依赖堆实现的的内存跟踪机制，但我发现各种堆内存的实现可能都存在一些差别，尤其是不太好分析内存释放时相关内存块的大小(C库的实现就是这样)。妥协后我决定基于目前的情况，配合FreeRTOS的heap实现来设计相关的内存跟踪机制。

FreeRTOS预留了traceMALLOC(pv, size)和traceFREE(pv, size)两个跟踪宏，用于指示内存申请和释放。它会指示内存指针和内存的大小。

基于这两个宏，我就能记录内存的分配次数、释放次数、内存的实时用量、内存的最大用量、内存泄露等。再通过一个简单的异或操作，可用非常方便的检测内存的申请和是否是否成对。

整个跟踪机制的代码如下：

```c
// heapt.h
#ifndef _HEAPT_H
#define _HEAPT_H
#include <stdint.h>

typedef struct {
    uint32_t trace_enable;
    uint32_t malloc_count;
    uint32_t free_count;
    uint32_t malloc_used_count;
    uint32_t malloc_max_count;
    uint32_t memory_used;
    uint32_t memory_max_used;
    uint32_t ptr_xor;
    uint32_t malloc_faild;
} heapt_info_t;

void heapt_init(void);
void heapt_deinit(void);
const heapt_info_t *heapt_get_info(void);

// FreeRTOS hook
void heapt_trace_malloc(void *ptr, uint32_t size);
void heapt_trace_free(void *ptr, uint32_t size);

#endif
```
```c
#include "heapt.h"
#include <string.h>

static heapt_info_t heapt_info;

void heapt_init(void){
    memset(&heapt_info, 0, sizeof(heapt_info));
    heapt_info.trace_enable = 1;
}

void heapt_deinit(void){
    memset(&heapt_info, 0, sizeof(heapt_info));
}

const heapt_info_t *heapt_get_info(void){
    return &heapt_info;
}

void heapt_trace_malloc(void *ptr, uint32_t size){
    if(heapt_info.trace_enable == 0){
        return ;
    }
    if(ptr == NULL){
        heapt_info.malloc_faild++;
        return ;
    }
    heapt_info.malloc_count++;
    heapt_info.malloc_used_count++;
    heapt_info.memory_used += size;
    if(heapt_info.malloc_used_count > heapt_info.malloc_max_count){
        heapt_info.malloc_max_count = heapt_info.malloc_used_count;
    }
    if(heapt_info.memory_used > heapt_info.memory_max_used){
        heapt_info.memory_max_used = heapt_info.memory_used;
    }
    heapt_info.ptr_xor ^= (uint32_t)ptr;
}

void heapt_trace_free(void *ptr, uint32_t size){
    if(heapt_info.trace_enable == 0){
        return ;
    }
    heapt_info.free_count++;
    heapt_info.malloc_used_count--;
    heapt_info.memory_used -= size;
    heapt_info.ptr_xor ^= (uint32_t)ptr;
}

int heapt(int argc, char * const argv[]){
    int ret;

    if(argc < 2){
        printf("Usage: %s PROG [ARGS]\r\n", argv[0]);
        return 1;
    }

    heapt_init();

    ret = run_cmd(argc - 1, &argv[1]);

    const heapt_info_t *info = heapt_get_info();
    printf("max heap use: %lu\r\n", info->memory_max_used);
    printf("max alloc nm: %lu\r\n", info->malloc_max_count);
    printf("malloc count: %lu\r\n", info->malloc_count);
    printf(" free  count: %-8lu"  , info->free_count);
    if(info->malloc_count != info->free_count){
        printf(" [Warning]\r\n");
    }else{
        printf("\r\n");
    }
    if(info->memory_used != 0){
        printf("used at exit: %-8lu [Warning]\r\n", info->memory_used);
    }
    if(info->ptr_xor){
        printf("mem ptr xor : %08lx [Warning]\r\n", info->ptr_xor);
    }
    if(info->malloc_faild){
        printf("malloc faild: %-8lu [Warning]\r\n", info->malloc_faild);
    }

    heapt_deinit();

    return ret;
}
```

基于该内存跟踪，我测试了一些常用的代码，发现它都工作良好，直到我测试RSA相关代码时。RSA一个解密操作就动辄100k+次的内存申请量，导致系统出现了一些意外的情况。

系统日志显示：
```text
max heap use: 17264
max alloc nm: 16
malloc count: 279544
 free  count: 279544
used at exit: 4294967288 [Warning]
```
日志显示内存分配和释放的内存字节数不相等，导致程序退出时内存记录的量不为0。起初我以为是内存有用法问题，但wolfSSL作为一款商业级的软件，应该不存在这样低级的错误。且mem ptr xor也没有报告内存泄露，内存申请和释放次数也是相等的。至此我不得不怀疑FreeRTOS提供的traceMALLOC(pv, size)和traceFREE(pv, size)存在问题。

## 发现FreeRTOS内核代码存在问题
FreeRTOS的heap4实现是基于空闲块链表实现的，将所有的可用内存块以链表的形式连接起来，当分配的内存小于内存块时，会将内存块进行分割，将其中一部分返回到用户，另一部分重新插入到空闲链表中。
```c
    // pvPortMalloc()
    // ....
    if( ( pxBlock->xBlockSize - xWantedSize ) > heapMINIMUM_BLOCK_SIZE )
    {
        /* This block is to be split into two.  Create a new
            * block following the number of bytes requested. The void
            * cast is used to prevent byte alignment warnings from the
            * compiler. */
        pxNewBlockLink = ( void * ) ( ( ( uint8_t * ) pxBlock ) + xWantedSize );
        configASSERT( ( ( ( size_t ) pxNewBlockLink ) & portBYTE_ALIGNMENT_MASK ) == 0 );

        /* Calculate the sizes of two blocks split from the
            * single block. */
        pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;
        pxBlock->xBlockSize = xWantedSize;

        /* Insert the new block into the list of free blocks. */
        pxNewBlockLink->pxNextFreeBlock = pxPreviousBlock->pxNextFreeBlock;
        pxPreviousBlock->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER( pxNewBlockLink );
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }

    xFreeBytesRemaining -= pxBlock->xBlockSize;

    if( xFreeBytesRemaining < xMinimumEverFreeBytesRemaining )
    {
        xMinimumEverFreeBytesRemaining = xFreeBytesRemaining;
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }

    // ....

    traceMALLOC( pvReturn, xWantedSize );

    //...
```
传入到traceMALLOC(pv, size)的size参数是用户传入的内存大小，这里返回到用户的内存卡大小可能比用户预期想要的要大。因为内存块可拆分的条件是内存块拆分后剩余的大小必须大于16字节，如果不满足条件则该内存不拆分且不会进一步去寻找其它的内存块，这样返回到用户的内存块大小实际上要比用户预期的要大一些。
```c
    // vPortFree()
    // ...
    vTaskSuspendAll();
    {
        /* Add this block to the list of free blocks. */
        xFreeBytesRemaining += pxLink->xBlockSize;
        traceFREE( pv, pxLink->xBlockSize );
        prvInsertBlockIntoFreeList( ( ( BlockLink_t * ) pxLink ) );
        xNumberOfSuccessfulFrees++;
    }
    ( void ) xTaskResumeAll();

    // ...
```
而传入到traceFREE(pv, size)的size正好是内存块的大小。这就导致了malloc和free时记录到的内存用量不一致。

这真是一个有意思的发现。traceMALLOC机制由早在2013年的V7.5.3版本引入，在过去的这段时间内大家似乎并没有经常使用它，导致这个问题潜伏了10年之久。好在这仅仅是一个作为调试功能存在的bug，而且它不会影响FreeRTOS的主要功能。

6月5日，我在[FreeRTOS-Kernel](https://github.com/FreeRTOS/FreeRTOS-Kernel)上提交了相关[Issue](https://github.com/FreeRTOS/FreeRTOS-Kernel/issues/1078)，内核开发小组很快确认该问题并建议我提交一个Pull Request来修复它。
*注：存在该问题的最后版本应该为V11.1.0*

## 修复内核代码
这个问题看起来非常简单，只需要将实际分配的内存块大小传入到 trace_MALLOC()函数中即可。但实际操作起来却让我非常为难。
- 当内存申请失败时，该size是0还是用户想要的内存大小xWantedSize？
- 直接修改xWantedSize还是新建变量进行处理？
- heap1和heap3是否需要同步修改？

我向内核提交的第一版代码是在内存块不拆分时修改xWantedSize，这样就能够使得trace_MALLOC获得正确的大小。但是官方在第一次审核我的代码时建议我建立一个新的变量用于表示内存块的大小而非使用xWantedSize，于是我修改了代码，这样看起来代码的可读性更高。由于不同处理器架构的heap实现有多个，我没法逐个去测试验证。PR代码只测试了heap4，但是其它的修改是一致的，且代码会经过GIT服务器上的自动化流程测试。确保代码在代码风格、可集成性、功能上都满足规范要求。

从我提交[PR - Fix traceMALLOC() allocated bytes](https://github.com/FreeRTOS/FreeRTOS-Kernel/pull/1089)到将代码合并入主干仅花了两天时间，过程很顺利。由于我的github账号是中文昵称，这导致合并到主线的代码作者名称显示为中文昵称，而不是我的英文名。这在Git-Graph中看起来怪怪的，似乎我没有看到其它开发者使用中文昵称提交的代码。😅

在我发现问题和解决问题的过程中，内核开发者都比较友好，也对我做的工作给予了肯定。这也是第一次尝试在大型开源项目中贡献代码，一段非常有意思的经历。
