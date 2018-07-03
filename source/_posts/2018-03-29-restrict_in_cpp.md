---
layout: post
category: C++
date: 2018-03-29
title: C/C++ 中的 restrict 关键字
---

　　restrict 是 C 的关键字(出现于 C99 标准)，用在指针声明中，C++ 的标准中没有 restrict ，但是现代的主流编译器都保留了对 restrict 的支持，如 VC++, GCC, Clang, Icc 等，不同编译器的书写形式或许不同，可能是 restrict, \_\_restrict\_\_, __restrict 等.<br>
　　restrict 的含义是由编程者向编译器声明，在这个指针的生命周期中，只有这个指针本身或者直接由它产生的指针(如 ptr + 1)能够用来访问该指针指向的对象．他的作用是限制指针别名，帮助编译器做优化．如果该指针与另外一个指针指向同一对象，那么结果是未定义的．

<!-- more -->

　　例如在下列代码中

```C++
int add (int* a, int* b)
{
    *a = 10;
    *b = 12;
    return *a + *b;
}
```

　　这个函数直观的作用便是将 a 和 b 指向的 int 对象赋值为 10 和 12, 然后返回他们两个的和，可以看出最后 `return *a + *b` 这条语句中是不需要的访问内存单元的，a 中一定是 10, b 中一定是 12, 所以编译器可以在这里做优化直接返回 22.<br>
　　但是如果 a 和 b 指向的是同一个 int 对象呢? 这时编译器就不应该做优化，实际上编译器的优化策略是保守的，当无法判断这里能不能优化时，便不做优化，所以可得如下汇编代码，这已经是编译器在已知信息下所能做的最好的优化，优化等级是 -O3

```asm
add(int*, int*):
  mov DWORD PTR [rdi], 10
  mov DWORD PTR [rsi], 12
  mov eax, DWORD PTR [rdi]
  add eax, 12
  ret
```

　　但是如果加上 `restrict` 修饰(编译器 GCC)，代码如下

```C++
int add (int* __restrict__ a, int* __restrict__ b)
{
    *a = 10;
    *b = 12;
    return *a + *b;
}
```

　　编译器已经得知 a, b 两指针不会指向同一单元(此点应由编程者保证)，所以在这里会减少一次内存的访问，可以得到如下的汇编代码

```asm
add(int*, int*):
  mov DWORD PTR [rdi], 10
  mov DWORD PTR [rsi], 12
  mov eax, 22
  ret
```

　　可见少了一条内存访问的指令

　　C++ snippet 的编译生成的指令可见于 [godbolt](https://godbolt.org/)