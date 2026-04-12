---
title: "malloc 与 free：手动管理内存的艺术"
slug: "c-lang-heap-malloc"
date: 2026-04-12T20:00:00+08:00
draft: false
tags:
  - C
  - 堆
  - malloc
  - free
  - 内存管理
  - 系统编程
categories:
  - 技术
series:
  - C 语言深度入门
description: 'malloc 向堆管理器申请内存，free 归还它。理解堆的机制、三大经典错误（泄漏/悬垂指针/双重释放）以及所有权规则。'
showToc: true
TocOpen: false
mermaid: false
---

# malloc 与 free：手动管理内存的艺术

> 系列：C 语言深度入门 · 第 4 篇  
> 前置知识：[第 3 篇：指针不是魔法：地址、解引用与数组](/posts/c-lang-pointer-array/)

---

## 你以为你知道的

很多初学者学 `malloc` 和 `free` 的方式是这样的：

> "需要数组就 `malloc`，用完就 `free`，不然会'内存泄漏'。"

这没错，但它回避了三个核心问题：

1. `malloc` 到底向谁申请内存？怎么申请的？
2. `free` 之后，那块内存发生了什么？
3. "内存泄漏"仅仅是浪费内存吗？还是会更危险？

带着这三个问题，我们从机器视角重新看堆内存。

---

## 堆在哪里？

回顾进程内存布局（[第一篇](/posts/c-lang-memory-world/)），每个 C 程序的地址空间大致是这样的：

<img src="/images/c-lang-heap-layout.svg" alt="进程内存布局：堆（Heap）位于栈与BSS段之间，向高地址增长" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

堆是一片可增长的内存区域，由**堆管理器**（glibc 中的 `ptmalloc`）负责。当你调用 `malloc`，实际发生的是：

1. 堆管理器检查是否有可用的空闲块
2. 如果有，直接返回；如果没有，用 `brk` 或 `mmap` 系统调用向操作系统要更多内存
3. 返回指向可用内存的指针

---

## malloc：申请一块内存

### 基本用法

```c
#include <stdlib.h>

// 申请 n 个 int 大小的连续内存
int *arr = malloc(sizeof(int) * n);

// 检查是否成功（malloc 失败返回 NULL）
if (arr == NULL) {
    fprintf(stderr, "malloc failed\n");
    exit(1);
}
```

**为什么要 `sizeof(int) * n` 而不是直接写 `4 * n`？**

因为 `int` 的大小是平台相关的。在大多数 64 位系统上是 4 字节，但标准不保证。`sizeof(int)` 让代码跨平台可移植。

### malloc 返回的是什么？

```c
int *p = malloc(sizeof(int) * 3);
printf("p = %p\n", (void*)p);        // 堆上某个地址，如 0x561f2c3a02a0
printf("sizeof(p) = %zu\n", sizeof(p));  // 8（64位系统上指针是8字节）
```

`p` 只是一个保存了堆地址的变量。它本身在栈上（只有 8 字节），指向的内存（12 字节）在堆上：

<img src="/images/c-lang-heap-pointer.svg" alt="指针 p 在栈上，指向堆上分配的12字节内存块" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

### malloc 不初始化内存

`malloc` 返回的内存内容是**未定义的**（通常是上次使用留下的残留值）。如果需要初始化为零，用 `calloc`：

```c
// calloc：申请并清零
int *arr = calloc(n, sizeof(int));   // 所有元素初始化为 0

// 等价于：
int *arr = malloc(n * sizeof(int));
memset(arr, 0, n * sizeof(int));
```

---

## free：归还内存

### free 做了什么？

```c
int *p = malloc(sizeof(int) * 3);
p[0] = 10; p[1] = 20; p[2] = 30;

free(p);   // 归还给堆管理器
```

`free(p)` 之后：
- 堆管理器把这块内存**标记为可用**
- 下次 `malloc` 时可能分配给其他人
- **`p` 的值没有改变**，它仍然保存着那个地址
- **那块内存的内容也没有被清除**（可能还有 10, 20, 30）

用图来看：

<img src="/images/c-lang-heap-free.svg" alt="free(p) 前后对比：释放后内存归还堆管理器，p 的值不变但指向无效内存" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

---

## 三大经典错误

### 错误一：内存泄漏（Memory Leak）

```c
void process_data() {
    int *buf = malloc(1024 * sizeof(int));
    // ... 做了一些事 ...
    return;   // 忘记 free(buf)！
}

int main() {
    for (int i = 0; i < 100000; i++) {
        process_data();   // 每次调用泄漏 4KB
    }                     // 总计泄漏约 400MB
}
```

**后果：** 程序占用内存持续增长，最终被操作系统 OOM Killer 杀掉。服务器程序尤其致命。

**规则：** 每个 `malloc` 必须对应一个 `free`，"谁申请，谁释放"。

### 错误二：悬垂指针（Dangling Pointer）

```c
int *p = malloc(sizeof(int));
*p = 42;

free(p);   // 释放了

printf("%d\n", *p);   // 悬垂指针！UB！
```

`free` 后继续使用指针是**未定义行为**。可能的结果：
- 打印出 42（堆管理器还没覆盖）
- 打印出垃圾值（被其他 malloc 覆盖了）
- 崩溃（内存被操作系统回收）
- 产生安全漏洞（Use-After-Free 漏洞）

**修复：** `free` 后立即将指针置为 `NULL`：

```c
free(p);
p = NULL;   // 后续误用会在这里崩溃，而不是踩内存
```

### 错误三：重复释放（Double Free）

```c
int *p = malloc(sizeof(int));
free(p);
free(p);   // double free！
```

`free` 同一指针两次会破坏堆管理器的内部数据结构，可能导致：
- 程序崩溃
- 堆元数据损坏（可被攻击者利用）

**修复：** `free` 后置 `NULL`，`free(NULL)` 是安全的空操作。

---

## 动手验证：用 Valgrind 检测泄漏

### 实验代码

```c
// leak_demo.c
#include <stdlib.h>
#include <stdio.h>

int *make_array(int n) {
    return malloc(sizeof(int) * n);
}

int main() {
    int *a = make_array(10);
    int *b = make_array(20);

    a[0] = 42;
    b[0] = 99;

    printf("a[0] = %d, b[0] = %d\n", a[0], b[0]);

    free(a);
    // 忘记 free(b)！

    return 0;
}
```

### 编译与检测

```bash
gcc -g -O0 leak_demo.c -o leak_demo
valgrind --leak-check=full --show-leak-kinds=all ./leak_demo
```

### Valgrind 输出

```
a[0] = 42, b[0] = 99

==12345== LEAK SUMMARY:
==12345==    definitely lost: 80 bytes in 1 blocks
==12345==      indirectly lost: 0 bytes in 0 blocks
==12345==        possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 0 bytes in 0 blocks
==12345==
==12345== 80 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x...: malloc (vg_replace_malloc.c:...)
==12345==    by 0x...: make_array (leak_demo.c:5)
==12345==    by 0x...: main (leak_demo.c:11)
```

Valgrind 精确告诉你：第 5 行的 `malloc`（在 `make_array` 中，由 `main` 第 11 行调用）产生了 80 字节（`sizeof(int) * 20`）的确定性泄漏。

---

## realloc：改变已有内存的大小

当你需要扩大或缩小一块堆内存时：

```c
int *arr = malloc(sizeof(int) * 5);
// ...填充数据...

// 扩大到 10 个 int
int *new_arr = realloc(arr, sizeof(int) * 10);
if (new_arr == NULL) {
    // realloc 失败：原 arr 仍然有效！不要 free(arr) 两次
    free(arr);
    exit(1);
}
arr = new_arr;   // 更新指针（realloc 可能移动了数据）
```

**重要：** `realloc` 可能在原地扩展，也可能分配新块并复制数据。不要假设地址不变。

---

## 所有权规则

堆内存没有自动管理。C 程序员需要自己维护"所有权"：

<img src="/images/c-lang-heap-ownership.svg" alt="所有权规则：谁申请的内存，谁负责释放" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

### 所有权清晰的写法

```c
// 申请者释放（最简单）
char *buf = malloc(128);
// ... 使用 ...
free(buf);

// 函数内部申请，返回给调用者（转移所有权）
// 调用者负责释放
char *create_message(const char *text) {
    char *msg = malloc(strlen(text) + 1);
    strcpy(msg, text);
    return msg;   // 调用者拥有这块内存
}

// 使用
char *m = create_message("Hello");
printf("%s\n", m);
free(m);   // 调用者负责释放
```

注释中写清"调用者需要 free"是良好习惯，否则极易泄漏。

---

## 栈 vs 堆：如何选择？

| 特性 | 栈（Stack） | 堆（Heap） |
|------|------------|-----------|
| 管理方式 | 自动 | 手动 |
| 生命周期 | 函数结束时自动销毁 | 显式 free 才释放 |
| 大小限制 | 通常 1-8 MB | 受系统内存限制 |
| 速度 | 极快（移动栈指针） | 较慢（堆管理器开销） |
| 适用场景 | 大小已知的局部数据 | 大数据、动态大小、跨函数生命周期 |

**实践原则：**
- 能用栈就用栈（小结构体、固定大小数组）
- 以下情况用堆：
  - 大小在编译时未知（用户输入决定）
  - 需要在函数返回后继续存活
  - 大于 1-2KB 的数据（避免栈溢出）

---

## 常见误区

| 误区 | 真相 |
|------|------|
| `free` 会把内存还给操作系统 | 通常只还给堆管理器，进程内存不会立刻减少 |
| `free` 后指针变成 NULL | 不会！必须手动赋 NULL |
| `malloc(0)` 是错误 | 标准允许，但返回的指针不能解引用 |
| 内存泄漏程序崩溃时就消失了 | 是的，但进程长期运行时会OOM |
| 不调用 `free` 程序会直接崩溃 | 不会立即崩溃，但内存占用持续增长 |

---

## 小结

| 操作 | 作用 | 注意 |
|------|------|------|
| `malloc(n)` | 申请 n 字节，返回指针（或NULL） | 返回的内存未初始化 |
| `calloc(n, size)` | 申请 n×size 字节，清零 | 比 malloc 稍慢 |
| `realloc(p, n)` | 调整已有块大小 | 地址可能改变 |
| `free(p)` | 释放内存 | 之后必须将 p 置 NULL |

**核心规则：申请和释放成对出现，free 后置 NULL，用 Valgrind 验证。**

---

## 下一篇预告

我们已经覆盖了 C 语言的四大核心领域：内存布局、栈机制、指针、堆管理。但还有一类风险潜藏在每一行代码里——**未定义行为（Undefined Behavior）**。

下一篇：《危险边界：UB 与安全防线》——我们会看到，为什么一个"看起来正确"的 C 程序，在编译器优化下可能产生完全出乎意料的行为，以及如何用工具链构建防线。
