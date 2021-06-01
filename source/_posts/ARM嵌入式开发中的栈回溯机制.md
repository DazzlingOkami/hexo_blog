---
title: ARM嵌入式开发中的栈回溯机制
---

## 两种ABI规范
根据编译器的实现不同，存在两种ABI机制，分别为APCS(ARM Procedure Call Standard)和AAPCS(ARM Archtecture Procedure Call Standard)。两种机制存在明显的区别，且单独APCS和AAPCS又有很多变种，这里就不详细展开叙述其本质。本文主要讨论分析栈帧相关的内容，所以这里仅仅介绍这两种ABI机制下栈帧的区别。

## APCS规范下函数调用和栈帧
#### 相关寄存器
APCS中规定了几个别名寄存器，分别是fp、ip、sp、lr、pc寄存器，它们实际对应的寄存器及含有有下表：

| APCS | Reg | 意义      |
|------|-----|---------|
| fp   | r11 | 栈帧指针寄存器 |
| ip   | r12 | 临时变量寄存器 |
| sp   | r13 | 栈指针寄存器  |
| lr   | r14 | 链接寄存器   |
| pc   | r15 | 程序位置寄存器 |

#### 函数调用时的栈帧
函数调用入口处将pc、lr、sp、fp寄存器依次入栈。入栈前先将sp保存到ip，入栈后再将ip的值给fp，这样fp的值就是当前栈帧的起始位置。
后续如果需要使用局部变量，就将sp减一个值实现在栈空间上开辟局部变量的空间。函数退出前将sp寄存器处理到堆栈平衡状态，再通过fp寄存器找到当前函数的栈帧位置，通过栈帧找到返回函数的fp寄存器和sp寄存器、lr寄存器。
简要的代码如下：
``` arm
<func1>:
mov ip, sp
push {fp, ip, lr, pc}
sub fp, ip, #4
sub sp, sp, #8      ;8 字节局部变量空间
;...
sub sp, fp, #12
ldm sp, {fp, sp, pc}
```

通过分析栈帧格式我们能够知道在函数任意位置，都能够通过fp寄存器找到当前栈帧位置，而不必考虑当前使用了多少局部变量空间，栈帧中能够得到当前函数的返回地址(lr)，当前运行地址(pc)，栈地址(sp)，同时根据栈帧中上一个函数的fp寄存器再找到上一层函数的栈帧数据，这样层层回溯就实现了栈的回溯。

#### 如何使用APCS
gcc手册上对使用APCS有相关的描述和控制选项，-mapcs-frame选项编译出来的代码就是使用APCS规范，默认情况是-mno-apcs-frame，关于这个选项的描述GCC手册原文是：
``` 
Generate a stack frame that is compliant with the ARM Procedure Call Standard for all functions, even if this is not strictly necessary for correct execution of the code. Specifying -fomit-frame-pointer with this option causes the stack frames not to be generated for leaf functions. The default is -mno-apcs-frame. This option is deprecated.
```

这里说明了使用相关选项可以保证APCS的栈帧格式，如果不适用这个选项则不保证。
APCS为1993年推出的标准，在后续推出AAPCS相关版本后它已经显得太旧了。在gcc5.0之后，该选项已经被遗弃，所以新版本的编译器已经不能使用APCS调用方式进行栈回溯。

## AAPCS下函数的调用
#### 相关寄存器
AAPCS相比于APCS少了几个别名寄存器。它有sp、lr、pc寄存器，对应表如下：

| APCS | Reg                | 意义      |
|------|--------------------|---------|
| sp   | r13(ARM)/r7(THUMB) | 栈指针寄存器  |
| lr   | r14                | 链接寄存器   |
| pc   | r15                | 程序位置寄存器 |

#### 函数调用时的栈帧
在这种调用规范下，栈的回溯要比APCS下要复杂的多，函数的调用过程中不保存栈帧的指针，而且栈帧中也没有固定的格式，所以不能通过当前的栈帧层层返回数据。所以仅仅通过堆栈数据是不能进行栈回溯的。
那么这种模式下如何保证堆栈平衡，能不能通过汇编代码的堆栈平衡实现栈回溯呢？
通过前面我们知道APCS模式下函数进入到退出时的堆栈平衡时通过保存sp寄存器和恢复sp寄存器实现的。而AAPCS模式下堆栈平衡是直接依靠pop和push平衡实现的，即不会直接修改sp来实现堆栈平衡。
通过前面的分析，如果我们能够准确知道每个函数关于栈的操作，这样也能够实现栈的跟踪。通过适当的编译选项，我们能够将这部信息收集到一起，这就是arm exception handler index，简称ARM.exidx，这段数据就能指导我们实现栈回溯，把这种栈回溯方式称为unwind。

#### ARM.exidx位置和结构
不同的编译器如何编译出exidx表具有不同的方式，但是如果可执行文件中包括exidx表，那么使用readelf -S file.elf 能够看到可执行文件中存在一个名为.ARM.exidx，类型为ARM_EXIDX的段。这就表示编译器已经生产的exidx段。
关于exidx的结构及其含义这里做一个简单介绍，详细信息可以参考ARM官方网站。exidx是一个由多个相同单元组成的表，每个单元由两个uint32_t的数据构成，结构类似于：
```c
struct {
    uint32_t offset;
    uint32_t world;
}
```
其中offset表示目标函数到当前位置的偏移，它的最高位固定为0。而world中存放了编码数据，它表示了对栈的不同操作，通过操作它就能够知道目标函数上关于栈的操作。

#### Linux中的栈回溯
Linux中关于ARM架构的栈回溯就是使用的exidx表实现的，这里以linux5.3版本内核中关于unwind frame的代码进行演示。
涉及的主要代码文件有arch/arm/kernel/unwind.c arch/arm/include/asm/unwind.h arch/arm/include/asm/ptrace.h stacktrace.h等。
主要将unwind.c移植到我们的嵌入式平台就能够使用unwind栈回溯了。

#### 代码移植
由于linux中存在内核空间和用户空间，两种空间下存在各种的exidx表，而通常我们的代码没有所谓的两个地址空间。所以需要将涉及到地址空间判断的全部判断为是否为代码空间段，linux对于用户空间段的exidx表是通过链表由用户手动添加的，内核段的exidx表是直接由编译符号完成的。其他的代码修改主要是涉及到Linux平台特性的。
下面介绍的相关修改都是基于gcc编译器(ver>5.0)实现的。
 - 1.实现两个关键的函数：
    ```c
    #define core_kernel_text(addr)    ((addr) > 0x08000000 && (addr) <= 0x08100000)
    #define kernel_text_address(addr) ((addr) > 0x08000000 && (addr) <= 0x08100000)
    ```
 - 2.添加日志输出的相关接口，主要就是两个，其中pr_debug日志可以选择不输出。
    ```c
    #define pr_debug(...) printf(__VA_ARGS__)
    #define pr_warn(...)  printf(__VA_ARGS__)
    ```
 - 3.关于分支预测的代码，可能在某些arm平台下不支持分支预测，所以这里的移植就把这个特性关闭掉。
    ```c
    #define likely(c)   (c)
    #define unlikely(c) (c)
    ```
 - 4.字节对齐函数，用于把某个数据按多少字节对齐。
    ```c
    #define ALIGN(x,a) (((x)+(a)-1)&~(a-1))
    ```
 - 5.内存操作的相关接口，这里其实并不会使用到内存相关申请，它只是在添加用户空间exidx表时才使用，而我们的嵌入式平台上是显然是用不到的。
    ```c
    #define kmalloc(size, flag)  malloc(size)
    #define kfree(ptr)           free(ptr)
    ```
 - 6.exidx表的位置，不同的编译器在链接脚本中关于exidx表的起始终止位置的描述可能不一样，这里需要重新定义一下。
    ```c
    #define __start_unwind_idx __exidx_start
    #define __stop_unwind_idx  __exidx_end
    ```
 - 7.实现获取关键寄存器的操作，主要是获取sp寄存器，如果不是在gcc编译器中，还需要实现获取lr寄存器和sp寄存器
    ```c
    register char * stack_ptr asm("sp");
    #define current_stack_pointer ((unsigned long) stack_ptr)
    ```
 - 8.linux中针对部分代码使用了spinlock进行保护，在我们的嵌入式平台上通常是不需要的，当然如果有必要可以进行选择性的实现类似spinlock的操作。
 - 9.关于代码是arm模式还是thumb模式，如果代码是thumb模式，还需要定义一个宏，他将决定fp寄存器是r7还是r11
    ```c
    #define CONFIG_THUMB2_KERNEL
    ```
 - 10.对于.h文件，主要是是保留unwind.h文件，删除其中不相干的代码，其他头文件中可能只是会引用部分数据和数据结构可以直接拷贝到unwind.h文件中。
 - 11.可以选择是否保留关于添加unwind_table的相关函数，这些函数通常不会使用。

除了上述代码的移植外，编译时需要开启-funwind-tables选项，使编译器能够正常生产exidx段。在连接时要保证链接脚本中有存放exidx段的位置，以及预留出索引exidx段的起止符号，类似这样的链接脚本：
```
  .ARM.extab   : { *(.ARM.extab* .gnu.linkonce.armextab.*) } >FLASH
  .ARM : {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } >FLASH
```

#### 将代码移植到ARMCC编译器中
keil作为一个应用在嵌入式开发领域的一个重要软件，如果能够将这个功能应用到ARMCC编译器环境下，那这个功能将显得更加具有现实意义。
前面的移植主要是在gcc编译平台下完成的，下面介绍如何将这段代码和功能移植到ARMCC编译平台下。
 - 1.编译环境相关的设置
    要使编译器能够正常编译出代码exidx数据段的代码需要开启特别的选项 --exceptions --exceptions-unwind，它的意义是开启异常处理机制，因为最初exidx段设计来就是用于处理类型c++中的异常机制的。在开启了这些选项后编译出的代码就会包含exidx段，可以使用指令进行检验查看，如readelf -S main.o，其中就能够看到相关的数据段。当然不同的文件中的exidx段还没有链接到一起，要把它们链接到一起还比较麻烦。
    由于ARMCC的优化策略，未使用的数据段都会在链接阶段被优化掉，而我们由不能直接使用到exidx中的数据(没有相关的符号指向这里)，所以正常情况下，即使开启的前面的编译选项，最终编译出来的文件也没有任何区别。在链接阶段添加选项--keep *(.ARM.exidx)，这个选项的意思就是在链接时保留exidx段(注意.ARM.exidx是exidx段的标准名称，前面没有提到，不同的编译器编译出来都是这个段名，且段类型都是未ARM_EXIDX)，这样链接出来的文件可以发现会比未加这个选项的文件大了些，这就新增的exidx段数据导致的。
    但是这时候还是存在一个问题，编译出的文件包含了exidx段的数据，但是却没在我们想要的地方，或者说没没办法确定这段数据在哪儿。readelf -S能够看到段的结构没有增加，而ER_IROM1段的数据变多了，这就说明exidx段的数据与代码数据放入了同一个段，这显然不是我们想要的。
    修改分散加载文件，在链接选项中指定使用自定义分散加载文件。我们起始可以看到原sct文件的结构为：
    ```
    LR_IROM1 0x08000000 0x00120000  {    ; load region size_region
        ER_IROM1 0x08000000 0x00100000  {  ; load address = execution address
            *.o (RESET, +First)
            *(InRoot$$Sections)
            .ANY (+RO)
            .ANY (+XO)
        }
        RW_IRAM1 0x20000100 0x00030000  {  ; RW data
            .ANY (+RW +ZI)
        }
    }
    ```

    我们在这里添加exidx段的声明后的结构为：
    ```
    LR_IROM1 0x08000000 0x00120000  {    ; load region size_region
        ER_IROM1 0x08000000 0x00100000  {  ; load address = execution address
            *.o (RESET, +First)
            *(InRoot$$Sections)
            .ANY (+RO)
            .ANY (+XO)
        }

        ER_EXIDX +0 0x00020000 {
            .ANY (.ARM.exidx)
        }

        RW_IRAM1 0x20000100 0x00030000  {  ; RW data
            .ANY (+RW +ZI)
        }
    }
    ```

    经过对编译器链接器的设置，我们终于将exidx段的数据放到了期望的地方，在编译器中使用符号```Image$$ER_EXIDX$$Base```和```Image$$ER_EXIDX$$Limit```就能访问到exidx段的数据(编译器特性)。
 - 2.处理完成了编译器的相关内容，还需要修改源代码以适应ARMCC编译器，需要修改的地方主要体现在使用到了gcc内建函数的地方，以及gcc支持的一些拓展语法上，具体如何修改这里不做详细的介绍了。以上包括gcc和keil平台下的移植都经过测试，能够非常准确的回溯函数的调用链，包括在使用函数指针的地方都能进行准确的回溯，对于分析软件bug具有一定的帮助。

## 技术参考
本文中所设计到的内容主要来自于ARM官方的技术文档库，GCC编译手册，Linux内核源码等。可自行检索参考。
