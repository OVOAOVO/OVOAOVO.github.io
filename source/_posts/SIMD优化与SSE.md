---
title: SIMD优化与SSE
date: 2024-06-03 22:13:54
tags: 学习
categories: 游戏技术 
---
# SIMD
Single Instruction Multiple Data，单指令多数据流，可以使用一条指令同时完成多个数据的运算操作。传统的指令架构是SISD就是单指令单数据流，每条指令只能对一个数据执行操作。
>简单举个例子一次可以算4次加法

## 使用方法1：内联汇编

下面的写法是 GCC 和 Clang 等编译器支持的特性。Visual Studio 使用的 MSVC 编译器并不支持 GCC 风格的内联汇编

```C++
#include <stdio.h>
#include <stdlib.h>    
int main()
{
    float a[4] = { 1,2,3,4 };
    float b[4] = { 5,6,7,8 };
    float res[4];
    __asm__ __volatile__( 
        "movups %1,%%xmm0\n\t"            // 将a所指内存的128位数据放入xmm0寄存器
        "movups %2,%%xmm1\n\t"            // 将b所指内存的128位数据放入xmm0寄存器
        "mulps  %%xmm1,%%xmm0\n\t"        // 计算 xmm0 * xmm1，结果放入 xmm0
        "movups %%xmm0,%0\n\t"            // 将xmm0的数据放入res所指内存
        :"=m"(res)                        //output
        :"m"(a),"m"(b)                    //input
    );
    for(int i=0;i<4;i++)
      printf("%.2f\n",res[i]);
  return 0;
}
```
movups指令，分为四个部分：

1. mov，表示数据移动。
2. u，表示 unaligned，内存未对齐。a，表示 aligned，内存已对齐。
3. p，表示 packed，打包数据，对128位所有数据执行操作。如果是s，则表示 scalar，标量数据，仅对128位内第一个数执行操作。
4. s表示 single precision floating point，将数据视为32位单精度浮点数，一组4个。d，表示 double precision floating point，将数据视为64位双精度浮点，一组两个。

从内存中向寄存器加载数据时，必须区分数据是否对齐。SSE指令要求数据按16字节对齐，未对齐数据必须使用movups，已对齐数据可以任意使用movups或者movaps。

## 使用方法2：Intrinsics内在函数

Intel提供的封装过的函数库。

```C++
#include <stdio.h>
#include <stdlib.h>
#include <emmintrin.h>    
int main()
{
    float a[4] = { 1,2,3,4 };
    float b[4] = { 5,6,7,8 };
    float res[4];
    __m128 A = _mm_loadu_ps(a);
    __m128 B = _mm_loadu_ps(b);
    __m128 RES = _mm_mul_ps(A, B);
    _mm_storeu_ps(res, RES);
    for(int i=0;i<4;i++)
      printf("%.2f\n",res[i]);
  return 0;
}
```

1. _mm，表示这是一个64位/128位的指令，_mm256和_mm512则表示是256位或是512位的指令。
2. _loadu，表示unaligen的load指令，不带u后缀的为aligen。
3. _ps，同上面汇编指令。

一些值得注意的事：

>_m128变量本身是占128位内存的结构体，不开编译器优化的时候会频繁发生回写内存再读取，性能很差。理论上_mm_load_ps编译生成的指令是movaps，_mm_loadu_ps生成的是movups，但经测试发现MSVC会把这两者都统一编译成movups，也许是对于现在的处理器来说，数据是否对齐带来的性能影响真的是可以忽略了。

SSE实际上是SIMD的扩展，然后还有一堆的指令集扩展。。。

>参考引用：https://zhuanlan.zhihu.com/p/583326378