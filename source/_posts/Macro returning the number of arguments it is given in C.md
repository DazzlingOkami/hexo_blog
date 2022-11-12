---
title: Macro returning the number of arguments it is given in C?
---

## 一个简单的实现
能否使用一个简单的宏在C语言中计算不定参数的数量，例如这样:
```
foo(1) -> 1
foo(cat, dog) -> 2
foo(red, green, blue) -> 3
```
一个比较简单的实现是这样的：
```c
#define PP_NARG(...) (sizeof((const void*[]){__VA_ARGS__})/sizeof(void*))
```
当参数为空时可能在某些编译器下不能正常的工作，但是稍微修改一下就行了。
```c
#define PP_NARG(...) (sizeof((int[]){0, ##__VA_ARGS__})/sizeof(int) - 1)
#pragma GCC diagnostic ignored "-Wint-conversion" /* Ignore warning */
```

这里使用了sizeof()关键字，就意味着它不是在预处理阶段计算不定参数的长度。同时要求参数必须为有意义的符号。这就限制了它的使用场景，比如说将不定参数的个数作为数组定义时的长度。

## 另外一种有趣的写法
这是我从网上找到的一种写法，它能够处理1~64个不定产生的情况，没有依赖于编译器的关键字，仅仅靠预处理就完成了参数的计数。
```c
#ifndef JLSS_ID_NARG_H
#define JLSS_ID_NARG_H

/*
** http://groups.google.com/group/comp.std.c/browse_thread/thread/77ee8c8f92e4a3fb/346fc464319b1ee5?pli=1
**
**    Newsgroups: comp.std.c
**    From: Laurent Deniau <laurent.deniau@cern.ch>
**    Date: Mon, 16 Jan 2006 18:43:40 +0100
**    Subject: __VA_NARG__
**
**    A year ago, I was asking here for an equivalent of __VA_NARG__ which
**    would return the number of arguments contained in __VA_ARGS__ before its
**    expansion. In fact my problem at that time (detecting for a third
**    argument) was solved by the solution of P. Mensonides. But I was still
**    thinking that the standard should have provided such a facilities rather
**    easy to compute for cpp.
**
**    This morning I had to face again the same problem, that is knowing the
**    number of arguments contained in __VA_ARGS__ before its expansion (after
**    its expansion can always be achieved if you can do it before). I found a
**    simple non-iterative solution which may be of interest here as an answer
**    to who will ask in the future for a kind of __VA_NARG__ in the standard
**    and I post it for archiving. May be some more elegant-efficient solution
**    exists?
**
**    Returns NARG, the number of arguments contained in __VA_ARGS__ before
**    expansion as far as NARG is >0 and <64 (cpp limits):
**
**    #define PP_NARG( ...) PP_NARG_(__VA_ARGS__,PP_RSEQ_N())
**    #define PP_NARG_(...) PP_ARG_N(__VA_ARGS__)
**    #define PP_ARG_N(_1,_2,_3,_4,_5,_6,_7,_8,_9,[..],_61,_62,_63,N,...) N
**    #define PP_RSEQ_N() 63,62,61,60,[..],9,8,7,6,5,4,3,2,1,0
**
**    [..] stands for the continuation of the sequence omitted here for
**    lisibility.
**
**    PP_NARG(A) -> 1
**    PP_NARG(A,B) -> 2
**    PP_NARG(A,B,C) -> 3
**    PP_NARG(A,B,C,D) -> 4
**    PP_NARG(A,B,C,D,E) -> 5
**    PP_NARG(A1,A2,[..],A62,A63) -> 63
**
** ======
**
**    Newsgroups: comp.std.c
**    From: Roland Illig <roland.il...@gmx.de>
**    Date: Fri, 20 Jan 2006 12:58:41 +0100
**    Subject: Re: __VA_NARG__
**
**    Laurent Deniau wrote:
**    > This morning I had to face again the same problem, that is knowing the
**    > number of arguments contained in __VA_ARGS__ before its expansion (after
**    > its expansion can always be achieved if you can do it before). I found a
**    > simple non-iterative solution which may be of interest here as an answer
**    > to who will ask in the future for a kind of __VA_NARG__ in the standard
**    > and I post it for archiving. May be some more elegant-efficient solution
**    > exists?
**
**    Thanks for this idea. I really like it.
**
**    For those that only want to copy and paste it, here is the expanded version:
**
** // Some test cases
** PP_NARG(A) -> 1
** PP_NARG(A,B) -> 2
** PP_NARG(A,B,C) -> 3
** PP_NARG(A,B,C,D) -> 4
** PP_NARG(A,B,C,D,E) -> 5
** PP_NARG(1,2,3,4,5,6,7,8,9,0,    //  1..10
**         1,2,3,4,5,6,7,8,9,0,    // 11..20
**         1,2,3,4,5,6,7,8,9,0,    // 21..30
**         1,2,3,4,5,6,7,8,9,0,    // 31..40
**         1,2,3,4,5,6,7,8,9,0,    // 41..50
**         1,2,3,4,5,6,7,8,9,0,    // 51..60
**         1,2,3) -> 63
**
**Note: using PP_NARG() without arguments would violate 6.10.3p4 of ISO C99.
*/

/* The PP_NARG macro returns the number of arguments that have been
** passed to it.
*/

#define PP_NARG(...) \
    PP_NARG_(__VA_ARGS__,PP_RSEQ_N())
#define PP_NARG_(...) \
    PP_ARG_N(__VA_ARGS__)
#define PP_ARG_N( \
     _1, _2, _3, _4, _5, _6, _7, _8, _9,_10, \
    _11,_12,_13,_14,_15,_16,_17,_18,_19,_20, \
    _21,_22,_23,_24,_25,_26,_27,_28,_29,_30, \
    _31,_32,_33,_34,_35,_36,_37,_38,_39,_40, \
    _41,_42,_43,_44,_45,_46,_47,_48,_49,_50, \
    _51,_52,_53,_54,_55,_56,_57,_58,_59,_60, \
    _61,_62,_63,  N, ...) N
#define PP_RSEQ_N() \
    63,62,61,60,                   \
    59,58,57,56,55,54,53,52,51,50, \
    49,48,47,46,45,44,43,42,41,40, \
    39,38,37,36,35,34,33,32,31,30, \
    29,28,27,26,25,24,23,22,21,20, \
    19,18,17,16,15,14,13,12,11,10, \
     9, 8, 7, 6, 5, 4, 3, 2, 1, 0

#endif /* JLSS_ID_NARG_H */
```
注释中也提到，当参数为空时需要编译器的特殊支持。但是我发现做适当的修改就能够避免这种尴尬的局面。
```c
#define PP_NARG(...)   (PP_NARG_(0, ##__VA_ARGS__, PP_RSEQ_N()) - 1)
```
只要参数不超过64个，它都能很好的工作，而且在预处理阶段就能计算出参数的个数。

## 总结
我参考了大量其他人的写法，其中的基本原理大都是基于这两种思想。在C++中我还看到其它的解法，但是我目前还没有想到在C++中需要统计不定参数个数的地方。
```cpp
#include <tuple>
#define MACRO(...) \
    std::cout <<"num args:" \
    << std::tuple_size<decltype(std::make_tuple(__VA_ARGS__))>::value \
    << std::endl;

/* Another way */
#define VA_COUNT(...) detail::va_count(__VA_ARGS__)
namespace detail
{
    template<typename ...Args>
    constexpr std::size_t va_count(Args&&...) { return sizeof...(Args); }
}
```

## 应用
能够计算不定参数个数对于实现函数静态优化有一定意义，在代码中判断参数个数调用不同的函数实现，相当于实现了函数多态的一种特性。
