---
title: C语言宏的图灵完备
author: WangGaojie
---

> 这里讨论的C语言宏系统仅关于#define，不包含#include, #if，#ifdef，#elif，#else，#endif等。

## C语言宏的可选参数
前段时间在某个嵌入式软件项目中涉及一个这样的功能，需要将一个wav音频文件嵌入式到二进制固件中，这样可以方便获取音频文件数据且不需要文件系统等复杂的组件。以前的做法是将音频文件写入到Flash中固定的地址下，然后再软件代码中从该地址访问相关的数据。但是这样的做法存在一定的灵活性问题，比如文件是否可能和固件重叠、无法统一烧录等问题。

后面我想到可以使用gcc汇编中的一条命令来实现在代码中包含一个文件，那就是.incbin指令。该指令在手册的描述如下：
```text
.incbin "file"[,skip[,count]]

The incbin directive includes file verbatim at the current location. You can control the 
search paths used with the ‘-I’ command-line option (see Command-Line Options). Quotation 
marks are required around file.

The skip argument skips a number of bytes from the start of the file. The count argument 
indicates the maximum number of bytes to read. Note that the data is not aligned in any 
way, so it is the user’s responsibility to make sure that proper alignment is provided 
both before and after the incbin directive.
```

将这个命令简单的使用宏封装一下，就可以在C代码文件中非常轻松的嵌入一个音频文件了。
```c
#define INCFILE(name, file)                             \
    __asm(                                              \
        ".global __" INCF_STR(name) "_start\n"          \
        ".type __" INCF_STR(name) "_start, %object\n"   \
        ".global __" INCF_STR(name) "_end\n"            \
        ".type __" INCF_STR(name) "_end, %object\n"     \
        "__" INCF_STR(name) "_start:\n"                 \
        ".incbin \"" file "\"\n"                        \
        "__" INCF_STR(name) "_end:\n"                   \
        ".word 0\n");                                   \
    extern int __##name##_start;                        \
    extern int __##name##_end;

#define FILEPTR(name) ((void*)&(__##name##_start))

#define FILESIZE(name) ((char*)&(__##name##_end) - (char*)&(__##name##_start))
```

封装之后，就可以在C代码中直接使用如下方式来嵌入音频文件：
```c
INCFILE(audio, "audio.wav");
int main()
{
    printf("audio size: %d\n", FILESIZE(audio));
    printf("audio ptr: %p\n", FILEPTR(audio));
    return 0;
}
```

再后来的使用过程中发现wav音频文件前的44字节是音频文件头，可以忽略，只需要使用后面的PCM数据即可。这样就需要将incbin中skip选项了，但是上面涉及的INCFILE()宏没有提供这个功能，所有又需要重新写一个宏。
```c
#define INCFILE_PART(name, file, offset, count)         \
    __asm(                                              \
        ".global __" INCF_STR(name) "_start\n"          \
        ".type __" INCF_STR(name) "_start, %object\n"   \
        ".global __" INCF_STR(name) "_end\n"            \
        ".type __" INCF_STR(name) "_end, %object\n"     \
        "__" INCF_STR(name) "_start:\n"                 \
        ".incbin \"" file "\"," INCF_STR(offset)        \
        "," INCF_STR(count) "\n"                        \
        "__" INCF_STR(name) "_end:\n"                   \
        ".word 0\n");                                   \
    extern int __##name##_start;                        \
    extern int __##name##_end;
```

这样的写法似乎也不太复合要求，因为这里不需要count参数，需要包含文件的全部内容，所以需要再封装一个宏？

类似的操作为什么需要写多个宏？这样的操作似乎就太不优雅了。

能不能实现一个这样的宏，能够提供多个可选参数呢？根据参数个数生成不同的代码。
```c
INCFILE(audio, "audio.wav")
INCFILE(audio, "audio.wav", 44)
INCFILE(audio, "audio.wav", 44 1024)
```

这样就引入了一个非常关键的问题，**C语言的宏定义如何根据参数个数来动态生成代码呢？**
```c
#define INCFILE(...) \
    if (NUM(__VA_ARGS__) == 2) { INCFILE(__VA_ARGS__)} else { INCFILE_PART(__VA_ARGS__) }
```

可以很明确的说，上面的代码是错误的，C语言的宏定义无法执行条件判断，所以无法实现这个功能。

宏定义如何实现条件分支流程呢？我们从一个简单的需求开始，请设计一个宏，判断宏参数列表中的参数个数是0个还是1个。
```c
#define PARA_FOLLOW(...) 0, ##__VA_ARGS__, 1, 0
```

我们来看看这段宏填入一些参数后展开的情况：
```c
/* 
PARA_FOLLOW(a) ->
    0, a, 1, 0

PARA_FOLLOW() ->
    0, 1, 0
 */
```

仔细观察可以发现展开后的结果中第三个数据就是该宏输入的参数个数。这似乎就解决了参数条件判断的问题，那如何进行分支流程呢？
将PARA_FOLLOW宏中跟随的0和1替换为分支语句，这样就完成了完整的条件分支。

```c
#define GET_P4(p1, p2, p3, p4, ...) p4
#define SELECT_012_PARA(p0, p1, p2, ...) GET_P4(0, ##__VA_ARGS__, p2, p1, p0)

/* 
SELECT_012_PARA( \
    printf("hello world\n");, \
    printf("hello macro\n");, \
    printf("hello hexo\n");) ->
    printf("hello world\n");

SELECT_012_PARA( \
    printf("hello world\n");, \
    printf("hello macro\n");, \
    printf("hello hexo\n");, 1) ->
    printf("hello macro\n");

SELECT_012_PARA( \
    printf("hello world\n");, \
    printf("hello macro\n");, \
    printf("hello hexo\n");, a, b) ->
    printf("hello hexo\n");
 */
```

SELECT_012_PARA()宏可以根据可以参数列表中参数的数量来生成不同的代码，而不关心参数具体的值。

据此，我重新设计了INCFILE()宏：
```c
#define INCFILE_OFFSET(file, offset)                    \
        ".incbin \"" file "\"," INCF_STR(offset) "\n"
        
#define INCFILE_COUNT(file, offset, count, ...)         \
        ".incbin \"" file "\"," INCF_STR(offset)        \
        "," INCF_STR(count) "\n"

#define INCFILE_PARA(name, file, offset, ...)           \
    __asm(                                              \
        ".global __" INCF_STR(name) "_start\n"          \
        ".type __" INCF_STR(name) "_start, %object\n"   \
        ".global __" INCF_STR(name) "_end\n"            \
        ".type __" INCF_STR(name) "_end, %object\n"     \
        "__" INCF_STR(name) "_start:\n"                 \
        SELECT_012_PARA(                                \
            INCFILE_OFFSET(file, 0),                    \
            INCFILE_OFFSET(file, offset),               \
            INCFILE_COUNT (file, offset, ##__VA_ARGS__, 0), \
            ##__VA_ARGS__)                              \
        "__" INCF_STR(name) "_end:\n"                   \
        ".word 0\n"                                     \
    )

#define INCFILE(name, file, ...)                        \
    INCFILE_PARA(name, file, ##__VA_ARGS__, 0);         \
    extern int __##name##_start;                        \
    extern int __##name##_end;
```

INCFILE可以使用不同的参数数量，来选择是否需要跳过头部数据。
```c
// hello.txt content is 12345678hellohello12345

INCFILE(file1, "hello.txt")
INCFILE(file2, "hello.txt", 8)
INCFILE(file3, "hello.txt", 8, 10)

int main(void){
     assert(memcmp(FILEPTR(file1), "12345678hellohello12345", 
         FILESIZE(file1)) == 0);
     assert(memcmp(FILEPTR(file2), "hellohello12345", FILESIZE(file2)) == 0);
     assert(memcmp(FILEPTR(file3), "hellohello", FILESIZE(file3)) == 0);
     return 0;
}
```
一个宏函数实现了多种功能，看起来有点多态的性质。

至此，一个在嵌入式软件开发过程中一个不起眼的问题就被成功解决了。

## 宏定义是否可以实现递归？
完成条件分支功能后，我进行了进一步的思考，在宏定义中能否实现递归呢？

请设计一个宏，该宏可以传入任意个整数，返回一个表达式，表示其最大值。

C语言的初学者可能都写过这样的宏：
```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))

/* 
MAX(1, 2) ->
    ((1) > (2) ? (1) : (2))
 */
```
这只能接受2个参数，如果传入3个参数，就会报错。任意个参数的递归版本也许是这样的：
```c
#define MAX(a, ...) (a > MAX(__VA_ARGS__) ? a : MAX(__VA_ARGS__))
```

实际展开情况呢？
```c
/* 
MAX(1, 2, 3) ->
    (1 > MAX(2, 3) ? 1 : MAX(2, 3))
 */
```

宏定义看起来无法正常的递归展开。这时候去检索相关的资料发现，C语言的宏定义确实无法直接实现递归。

但是在查询资料的时候，我发现通过一种通过延迟求值的技巧，可以做到有限的递归。

说到延迟求值，那就有必要先回顾一下C语言宏定义的求值规则。

## 宏定义的求值规则
先展开内层实参，再展开外层宏函数。如果形参前有#、#@或者形参前后有##，则对应实参不展开。

宏函数求值规则特别复杂，简单来说，先计算实参的值，替换到宏函数中，然后重新进行计算。

每当宏展开为非宏的标识符后，将触发重新扫描，从开始处重新计算。当对宏函数A()求值时遇到A，会将其标记为非宏标识符以避免递归，即使从外层重新对其进行扫描也无法对其继续展开。

一个例子进行求值说明：
```c
#define A(p1, p2, p3, p4, p5) p5
#define B() 1, 2, 3
#define C(a) A(a, 0, B())

#define A_I(...) A(__VA_ARGS__)
#define C_I(a) A_I(a, 0, B())

C(1) ->      // 先计算内层参数值，1无需计算
C(1) ->      // 展开外层宏函数
A(1, 0, B()) // Error, 展开后只有三个参数，无法匹配宏定义，直接报错

C_I(1) ->    // 先计算内层参数值，1无需计算
C_I(1) ->    // 展开外层宏函数
A_I(1, 0, B()) ->   // A_I为不定参数个数的宏，可以展开，先计算内层参数值，需要计算B()
A_I(1, 0, 1, 2, 3) ->  // 展开外层宏函数
A(1, 0, 1, 2, 3)   ->  // 计算内层参数值，所有参数都无需计算，直接展开外层宏函数
3

// 看一个特别的例子
#define A_I2(...) A(##__VA_ARGS__)
#define C_I2(a) A_I2(a, 0, B())
C_I2(1) ->   // 先计算内层参数值，1无需计算
C_I2(1) ->   // 展开外层宏函数
A_I2(1, 0, B()) ->  // 此时本应该先展开内层参数，也就是B()，但A_I2参数中的##阻止
                    // 了1, 0, B()这个整体被求值。所以继续展开外层宏函数
A(1, 0, B()) // Error, 参数数量不匹配，报错

// 可以仔细对比C_I()和C_I2()的差异，唯一的区别就是__VA_ARGS__多了一个##
// 正常情况下__VA_ARGS__前的##表示忽略可选参数带来的额外空格。
// 在大多数情况下影响不大，但在遇到宏嵌套复合求值时会导致宏展开顺序不一致的问题。
```

再看一个许多网友都没有正确理解的例子：
```c
#define  cat(a,b)     a ## b
#define  xcat(x, y)   cat(x, y)
xcat ( xcat ( 1 , 2 ) , 3 ) -> 
xcat ( cat ( 1 , 2 ) , 3 ) ->
xcat ( 12 , 3 ) ->
cat ( 12 , 3 ) ->
123

// 有部分文章在使用这个例子介绍宏嵌套求值顺序时写错了。大家不要被错误的信息误导。
// 如果你对此表示怀疑，那么你可以使用ppstep工具去查看宏展开过程中的细节。
```

在编写宏函数是注意一个技巧，在描述宏嵌套时，对于外层的宏函数实现来说，其参数不能直接含有#、#@、##，需要通过另一个宏函数间接实现。在编写宏函数时注意这一点可以避免很多意料之外的求值顺序问题。

以上这样的宏定义求值简直太简单了。真正的大招在后面~

## 对宏定义递归的探索
在StackOverflow上，有人提出是否可以实现这样的递归宏定义：
```c
#define pr(n) ((n==1)? 1 : pr(n-1))
```

我们可以根据之前学习的规则来尝试展开pr(5):
```c
/* 
pr(5) -> ((5==1)? 1 : pr(5-1)) -> ?
此时这个结果将无法进一步展开，为什么呢?在一次宏函数求值过程中，无法访问宏定义本身。
此时pr(5-1)将被宏处理器记住，它在后续永远也不会展开。
 */
```

为了避免这个问题，大佬们提出了一种延时求值的技巧。
```c
#define EMPTY(...)
#define DEFER(...) __VA_ARGS__ EMPTY()
#define OBSTRUCT(...) __VA_ARGS__ DEFER(EMPTY)()
#define EXPAND(...) __VA_ARGS__

#define pr_id() pr
#define pr(n) ((n==1)? 1 : DEFER(pr_id)()(n-1))
```

可以尝试再一次展开pr(5):
```c
/* 
pr(5) -> // 展开外层宏函数
((5==1)? 1 : DEFER(pr_id)()(5 -1)) 
    -> // 展开外层函数时存在DEFER()需展开，注意pr_id后面没有括号，所以pr_id不展开
((5==1)? 1 : pr_id()(5 -1))     
       // 这就是最终的结果，因为此时外层没有宏函数包裹，宏处理器不会进一步进行求值

// 如果在外面包裹一个EXPAND，那就能进一步展开
EXPAND(pr(5)) ->
...     // 省略一些过程
EXPAND(((5==1)? 1 : pr_id()(5 -1))) -> // 外层存在一个宏函数，内层需要求值。需要计算pr_id()
EXPAND(((5==1)? 1 : pr(5 -1)))      -> // 注意，此时内层已经展开完成了。开始展开外层宏函数
((5==1)? 1 : pr(5 -1))  -> 展开外层宏函数后，重新求值发现存在一个pr()需要进一步展开
((5==1)? 1 : ((5 -1==1)? 1 : pr_id ()(5 -1 -1)))    // 这就是最终的结果
 */
```

EXPAND()的作用就是进行一次展开。显然上面的递归定义需要多次展开。所以可以将多个EXPAND叠加在一起组合为一个新的宏函数。
```c
#define EVAL(...)  EVAL1(EVAL1(EVAL1(__VA_ARGS__)))
#define EVAL1(...) EVAL2(EVAL2(EVAL2(__VA_ARGS__)))
#define EVAL2(...) EVAL3(EVAL3(EVAL3(__VA_ARGS__)))
#define EVAL3(...) EVAL4(EVAL4(EVAL4(__VA_ARGS__)))
#define EVAL4(...) EVAL5(EVAL5(EVAL5(__VA_ARGS__)))
#define EVAL5(...) __VA_ARGS__
```
调用EVAL()可以进行多少次求值呢？243次。

这里通过一次DEFER操作实现了对pr的延时求值，使得在外部通过调用EXPAND来在外部展开，避免了无法直接递归的问题。但是很显然，在外部手动调用EXAND的次数也是有限的，不能无限递归。

至此，到这里仅仅完成了递归嵌套的定义，对于递归来说，最重要的是递归中止条件，这才是用宏函数实现递归的一大难点。在进一步探索递归前，我们学习一下大佬们实现的用宏函数完成的循环语句。

## 使用宏定义实现循环语句
参考内容：[Is the C99 preprocessor Turing complete?](https://stackoverflow.com/questions/3136686/is-the-c99-preprocessor-turing-complete)

有大佬实现了一个循环语句，通过宏定义实现。这里把其中的关键代码整理如下：
```c
#define EMPTY(...)
#define DEFER(...) __VA_ARGS__ EMPTY()
#define OBSTRUCT(...) __VA_ARGS__ DEFER(EMPTY)()
#define EXPAND(...) __VA_ARGS__

#define EVAL(...)  EVAL1(EVAL1(EVAL1(__VA_ARGS__)))
#define EVAL1(...) EVAL2(EVAL2(EVAL2(__VA_ARGS__)))
#define EVAL2(...) EVAL3(EVAL3(EVAL3(__VA_ARGS__)))
#define EVAL3(...) EVAL4(EVAL4(EVAL4(__VA_ARGS__)))
#define EVAL4(...) EVAL5(EVAL5(EVAL5(__VA_ARGS__)))
#define EVAL5(...) __VA_ARGS__

#define CAT(a, ...) PRIMITIVE_CAT(a, __VA_ARGS__)
#define PRIMITIVE_CAT(a, ...) a ## __VA_ARGS__

#define INC(x) PRIMITIVE_CAT(INC_, x)
#define INC_0 1
#define INC_1 2
#define INC_2 3
#define INC_3 4
#define INC_4 5
#define INC_5 6
#define INC_6 7
#define INC_7 8
#define INC_8 9
#define INC_9 9

#define DEC(x) PRIMITIVE_CAT(DEC_, x)
#define DEC_0 0
#define DEC_1 0
#define DEC_2 1
#define DEC_3 2
#define DEC_4 3
#define DEC_5 4
#define DEC_6 5
#define DEC_7 6
#define DEC_8 7
#define DEC_9 8

#define CHECK_N(x, n, ...) n
#define CHECK(...) CHECK_N(__VA_ARGS__, 0,)

#define NOT(x) CHECK(PRIMITIVE_CAT(NOT_, x))
#define NOT_0 ~, 1,

#define COMPL(b) PRIMITIVE_CAT(COMPL_, b)
#define COMPL_0 1
#define COMPL_1 0

#define BOOL(x) COMPL(NOT(x))

#define IIF(c) PRIMITIVE_CAT(IIF_, c)
#define IIF_0(t, ...) __VA_ARGS__
#define IIF_1(t, ...) t

#define IF(c) IIF(BOOL(c))

#define EAT(...)
#define EXPAND(...) __VA_ARGS__
#define WHEN(c) IF(c)(EXPAND, EAT)

#define REPEAT(count, macro, ...) \
    WHEN(count) \
    ( \
        OBSTRUCT(REPEAT_INDIRECT) () \
        ( \
            DEC(count), macro, __VA_ARGS__ \
        ) \
        OBSTRUCT(macro) \
        ( \
            DEC(count), __VA_ARGS__ \
        ) \
    )
#define REPEAT_INDIRECT() REPEAT
```

来看看其强大之处：
```c
#define M(i, _) i,
int nums[] = {EVAL(REPEAT(8, M, ~))};
// -> int nums[] = {0, 1, 2, 3, 4, 5, 6, 7,};

#define OPR(i, _) i _
int sum = EVAL(REPEAT(9, OPR, +)) 0;
// -> int sum = 0 + 1 + 2 + 3 + 4 + 5 + 6 + 7 + 8 + 0;

int product = EVAL(REPEAT(9, OPR, *)) 1;
// -> int product = 0 * 1 * 2 * 3 * 4 * 5 * 6 * 7 * 8 * 1;
```

这个循环的内在逻辑这里就不多解释了，多看看就会了，有很多的技巧。

当我掌握了这里面的一些技巧后，我开始设计一个求多个数求最大值的宏函数。这并不容易，需要使用递归实现并完成递归中止条件的处理。

## 宏函数递归实现求最大值
思路如下：
- 1.采用延时求值表达式完成正常的递归表达；
- 2.完成递归中止条件，通过判断参数个数来判断是否递归结束；
- 3.递归内部分支流程，中止条件完成后需要进行分支流程，一种是继续递归，一种是当只有一个参数时直接返回。

首先是递归表达式：
```c
#define MAX_NUM(a, ...) \
    ( \
        a > DEFER(MAX_INDIRECT)(__VA_ARGS__)(__VA_ARGS__) ? \
        a : DEFER(MAX_INDIRECT)(__VA_ARGS__)(__VA_ARGS__) \
    )
#define MAX_INDIRECT(...) MAX_NUM
```
这里应用了延时求值的技巧。

接下来是递归中止条件，需要判断参数个数是否为1，先实现一个判断宏参数是否存在的宏函数：
```c
#define GET_P64_I(\
    p1, p2, p3, p4, p5, p6, p7, p8, p9, p10,\
    p11, p12, p13, p14, p15, p16, p17, p18, p19, p20,\
    p21, p22, p23, p24, p25, p26, p27, p28, p29, p30,\
    p31, p32, p33, p34, p35, p36, p37, p38, p39, p40,\
    p41, p42, p43, p44, p45, p46, p47, p48, p49, p50,\
    p51, p52, p53, p54, p55, p56, p57, p58, p59, p60,\
    p61, p62, p63, p64, ... \
    ) p64
// 避免内层参数未展开导致参数数量不匹配的问题
#define GET_P64(...) GET_P64_I(__VA_ARGS__)

#define ONE_X62 \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, \
    1, 1

// 避免直接使用带有##__VA_ARGS__的宏，影响求值顺序
#define CHECK_ARGS_I(...) \
    GET_P64(~, ##__VA_ARGS__, \
        ONE_X62, 0)

#define CHECK_ARGS(...) CHECK_ARGS_I(__VA_ARGS__)

/* 
CHECK_ARGS(1) -> 1
CHECK_ARGS() -> 0
CHECK_ARGS(1,2) -> 1
CHECK_ARGS(1,2,3) -> 1
 */
```
CHECK_ARGS()宏函数可以判断宏参数个数，如果有一个或多个参数则返回1，参数列表为空则返回0。

接下来使用CHECK_ARGS()来实现一个判断参数是否只有一个的宏(处理递归结束)：
```c
#define EAT_FIRST(x, ...) __VA_ARGS__

#define CHECK_ARGS1(...) CHECK_ARGS(EAT_FIRST(__VA_ARGS__))

/* 
CHECK_ARGS1(1) -> 0
CHECK_ARGS1(1, 2) -> 1
CHECK_ARGS1() -> Error, 无法求值
 */
```

接下来通过CHECK_ARGS1()配合CAT()宏，拼接出MAX_1和MAX_0两个分支：
```c
#define MAX_NUM(a, ...) \
    ( \
        a > DEFER(MAX_INDIRECT)(__VA_ARGS__)(__VA_ARGS__) ? \
        a : DEFER(MAX_INDIRECT)(__VA_ARGS__)(__VA_ARGS__) \
    )
#define MAX_INDIRECT(...) MAX_BRANCH(CHECK_ARGS1(__VA_ARGS__))
#define MAX_BRANCH(n) PRIMITIVE_CAT(MAX_, n)

#define FIRST(a, ...) a

#define MAX_1 MAX_NUM
#define MAX_0 FIRST
```

至此MAX_NUM()就能够完成主要的功能，由于延迟求值的原因，需要在外层再套一个EVAL()。每多一个参数就多一层递归，也就多一次延迟求值。前面写的EVAL()可以处理243次延迟求值。
```c
#define MAX(...) EVAL(MAX_NUM(__VA_ARGS__))

/* 
MAX(1, 2) ->
    ( 1 > 2 ? 1 : 2 )

MAX(1, 2, 3) ->
    ( 1 > ( 2 > 3 ? 2 : 3 ) ? 1 : ( 2 > 3 ? 2 : 3 ) )

MAX(1, 2, 3, 4) ->
    ( 1 > ( 2 > ( 3 > 4 ? 3 : 4 ) ? 2 : ( 3 > 4 ? 3 : 4 ) ) ? 1 : 
        ( 2 > ( 3 > 4 ? 3 : 4 ) ? 2 : ( 3 > 4 ? 3 : 4 ) ) )

MAX(a, b, c, d, e, f) ->
    ( a > ( b > ( c > ( d > ( e > f ? e : f ) ? d : ( e > f ? e : 
    f ) ) ? c : ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) ) 
    ) ? b : ( c > ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) 
    ) ? c : ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) ) ) ) 
    ? a : ( b > ( c > ( d > ( e > f ? e : f ) ? d : ( e > f ? e : 
    f ) ) ? c : ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) ) 
    ) ? b : ( c > ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) 
    ) ? c : ( d > ( e > f ? e : f ) ? d : ( e > f ? e : f ) ) ) ) )

int max3(v1, v3, v3){
    return MAX(v1, v2, v3);
} ->
    int max3(v1, v3, v3){
        return ( v1 > ( v2 > v3 ? v2 : v3 ) ? v1 : ( v2 > v3 ? v2 : v3 ) );
    }

----
Error:
MAX() -> ( > ? : )
MAX(1) -> ( 1 > ? 1 : )
 */

```

可以看出，递归展开后呈二次增长。无论生成的表达式多复杂，结果都是正确的。受限于CHECK_ARGS()，这只能处理不超过63个数的最大值计算。

这里还存在一点点瑕疵，前面是先递归再分支判断，所以传入一个或零个参数时的结果不正确。可以先直接调用MAX_INDIRECT()做一次分支判断，当只有一个参数时就直接返回。
```c
#define MAX(...) EVAL(MAX_INDIRECT(__VA_ARGS__)(__VA_ARGS__))

/* 
MAX(1) -> 1
MAX(1, 2) -> ( 1 > 2 ? 1 : 2 )
----
MAX() -> 返回为空

MAX_INDIRECT为什么有两个参数列表？
    第一个参数列表用于去判断参数数量做分支处理，第二个参数列表是实际待处理数据。
 */
```

## 数值系统
MAX()真正意义上来说没有得到一个数列的最大值，原因是C语言的宏定义没有数值计算系统。所以这里返回了一个表示最大值表达式。

进一步思考一下，宏定义能不能实现数值计算？
```c
#define ADD(a, b) (a+b)
```

这并不是我想要的，这仅仅返回了一个表达式。我需要的是能够直接返回计算结果。
```c
/* 
ADD(1, 2) -> (1+2) // Error
ADD(1, 2) -> 3     // It's OK. How to implement it?
 */
```

注意前文的两个宏函数INC()、DEC()，它们能够实现一个数的加1或减1：
```c
/* 
INC(5) -> 6
INC(0) -> 1
DEC(1) -> 0
DEC(9) -> 8
INC(DEC(1)) -> 1
 */
```

那么ADD(a, b)函数可以表达为需要调用a次的INC(b)。或者说a+b=(a-1)+(b+1)，ADD(a, b)=ADD(DEC(a), INC(b))。这看起来又是一种递归的形式了，当a等于0时结束递归。这是一种尾递归的形式，可以使用表达为循环。来看看C语言的形式：
```c
int add(int a, int b){
    if(a){
        return add(a - 1, b + 1);
    }else{
        return b;
    }
}
```
这样的形式就很容易使用宏定义表达：
```c
#define ADD_INDIRECT() ADD
#define ADD(a, b) \
    WHEN(a) (OBSTRUCT(ADD_INDIRECT)() (DEC(a), INC(b))) \
    WHEN(NOT(a)) (b)

/* 
这里使用OBSTRUCT()来进行延时求值，所以最终的表达式需要使用EVAL()进行展开求值。

EVAL(ADD(1,2)) -> 3
EVAL(ADD(2,2)) -> 4
EVAL(ADD(5,0)) -> 5
EVAL(ADD(0,5)) -> 5
EVAL(ADD(0,0)) -> 0
EVAL(ADD(2,EVAL(ADD(2,2)))) -> 6
 */
```

思考一下，能不能实现一个数值版本的MAX()宏函数？它能真正计算出数列的最大值。
```c
/* 
MAX(1, 2)    -> 2
MAX(3, 2, 1) -> 3
 */
```

这似乎是一个新的挑战，需要完成宏定义中没有的数值比较功能。后面有时间再研究。😅

## 总结
仅用#define功能就完成了条件语句、循环、递归的实现。这是不是意味着C语言的宏是图灵完备的呢？

我查阅了很多网友的讨论，大部分人的看法是有限个数的宏标识符无法完成尽可能大的递归深度。通过C语言宏实现的counter机制要求每个值都需要一个宏标识符，这就限制了宏的复杂性。因为C语言的规范中限定了单次宏处理标识符的最大数量。这和其它图灵机的无限内存是有本质区别的。

此外，前面所实现的循环和递归都是有限深度的。无法实现无限循环和无穷递归就不具备完整的图灵完备性。但它确实完成了迭代、条件等类似的结构，这是不是意味着它是存在有限的图灵完备性。

本文仅讨论C语言宏的一些内在性质，并不推荐在实际的开发中使用如此复杂的代码。

## 参考
1.[theory - 什么是图灵完备？](https://stackoverflow.org.cn/questions/7284)
2.[C/C++ 宏的图灵完备性](https://chengzi.tech/posts/language/c-macro.md/)
3.[Is the C99 preprocessor Turing complete?](https://stackoverflow.com/questions/3136686/is-the-c99-preprocessor-turing-complete)
4.[Dave Prosser's C Preprocessing Algorithm](https://www.spinellis.gr/blog/20060626/)
5.[Best abuse of the C preprocessor](https://www.ioccc.org/2001/herrmann1.hint)
