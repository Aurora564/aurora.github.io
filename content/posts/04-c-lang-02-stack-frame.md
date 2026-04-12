---
title: "函数调用的秘密：栈帧与值传递"
slug: "c-lang-stack-frame"
date: 2026-04-12T18:00:00+08:00
draft: false
tags:
  - C
  - 栈
  - 函数调用
  - 系统编程
categories:
  - 技术
series:
  - C 语言深度入门
description: '每次函数调用都会在栈上开辟一块"专属工作台"——栈帧。理解它，彻底搞懂值传递、悬空指针和栈溢出三个 C 语言经典困惑。'
showToc: true
TocOpen: false
mermaid: false
---

# 函数调用的秘密：栈帧与值传递

> 系列：C 语言深度入门 · 第 2 篇  
> 前置知识：[第 1 篇：你的代码住在哪里](/posts/c-lang-memory-world/)

---

上一篇，我们画出了 C 程序的内存地图：代码段、数据段、BSS 段、堆、栈。

这一篇，我们放大镜对准**栈**，看清每一次函数调用背后究竟发生了什么。

理解这一点，你就能彻底搞懂三个让初学者困惑的问题：
- 为什么修改函数参数，调用者的变量不变？
- 为什么不能返回局部变量的地址？
- 为什么递归太深会崩溃？

---

## 一、函数调用的本质：三个动作

在你写下 `foo(x)` 的那一刻，CPU 其实执行了三件事：

1. **把参数的值放到特定位置**（寄存器或压栈）
2. **把"下一条指令的地址"压入栈**（这叫**返回地址**，用于函数返回后继续执行）
3. **跳转到 `foo` 函数的代码段执行**

函数执行完毕，再把返回地址弹出，跳回去继续运行。

听起来简单，但魔鬼在于：**为了执行这些操作，系统在栈上为每次函数调用分配了一块专属空间——栈帧（Stack Frame）**。

---

## 二、栈帧：函数的"专属工作台"

每次调用函数，系统会在栈上开辟一个栈帧，里面存放：

<img src="/images/c-lang-stack-frame.svg" alt="栈帧结构：参数副本、返回地址、帧指针、局部变量" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

函数返回时，这块栈帧**整个被释放**——不是清零，只是"归还"，之后可能被下一次函数调用覆盖。

---

## 三、用地址亲眼看见栈帧

光说不练，我们让程序自己展示栈帧：

```c
#include <stdio.h>

void bar() {
    int c = 300;
    printf("bar  中 c 的地址: %p\n", (void*)&c);
}

void foo() {
    int b = 200;
    printf("foo  中 b 的地址: %p\n", (void*)&b);
    bar();
}

int main() {
    int a = 100;
    printf("main 中 a 的地址: %p\n", (void*)&a);
    foo();
    return 0;
}
```

编译运行（`gcc stack.c -o stack && ./stack`），你会看到：

```
main 中 a 的地址: 0x7ffd5a8c1c3c
foo  中 b 的地址: 0x7ffd5a8c1c18   ← 比 a 的地址小
bar  中 c 的地址: 0x7ffd5a8c1bf4   ← 比 b 的地址更小
```

调用越深，地址越小。这不是巧合——**栈向低地址增长**，每次调用都在当前栈顶的更低位置开辟新栈帧。

如果画成图：

<img src="/images/c-lang-call-stack.svg" alt="调用栈：main/foo/bar 三层栈帧与地址" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

---

## 四、值传递：传的是"影印件"

现在来解开第一个困惑：**为什么修改参数不影响原变量？**

```c
#include <stdio.h>

void try_double(int n) {
    n = n * 2;  // 修改的是影印件
    printf("函数内 n = %d，地址 %p\n", n, (void*)&n);
}

int main() {
    int x = 10;
    printf("调用前 x = %d，地址 %p\n", x, (void*)&x);
    try_double(x);
    printf("调用后 x = %d，地址 %p\n", x, (void*)&x);
    return 0;
}
```

运行结果：

```
调用前 x = 10，地址 0x7ffd...a4
函数内 n = 20，地址 0x7ffd...7c   ← 注意！地址不同
调用后 x = 10，地址 0x7ffd...a4   ← x 没变
```

`x` 和 `n` 的地址不同，说明**它们是两块独立的内存**。

调用 `try_double(x)` 时，C 把 `x` 的值（10）复制了一份，放进 `try_double` 的栈帧里，取名叫 `n`。之后对 `n` 做的任何操作，都只影响这份拷贝，`x` 纹丝不动。

**这就是"值传递（Pass by Value）"**：C 函数调用传的永远是值的副本，不是原变量本身。

<img src="/images/c-lang-value-passing.svg" alt="值传递：main 的 x 与 try_double 的 n 是两块独立内存" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

如果你想让函数"修改"调用者的变量，就必须传地址（指针）——这是第 3 篇要讲的内容。

---

## 五、危险的悬空指针：别把栈帧的地址带出去

既然栈帧函数返回就销毁，如果你试图返回局部变量的地址……

```c
int* danger() {
    int x = 42;
    return &x;  // ⚠️ 返回了局部变量的地址！
}

int main() {
    int *p = danger();
    printf("%d\n", *p);  // 未定义行为！
}
```

`danger()` 返回时，它的栈帧被释放。`x` 的地址（现在保存在 `p` 里）指向一块"已归还"的内存——随时可能被下一次函数调用覆盖成别的东西。

你通过 `*p` 读到的，可能是 42，可能是垃圾值，也可能直接 Segfault。**这是未定义行为（Undefined Behavior）**，程序的一切表现都不可预期。

现代编译器会警告这个错误：

```
warning: function returns address of local variable [-Wreturn-local-addr]
```

**规则：永远不要返回局部变量的指针。** 需要在函数外使用的数据，用堆（`malloc`）分配，或者通过参数传出去。

---

## 六、栈溢出：把工作台堆满

最后，来看递归太深为什么崩溃：

```c
#include <stdio.h>

int depth = 0;

void recurse() {
    depth++;
    if (depth % 10000 == 0)
        printf("递归深度: %d\n", depth);
    recurse();  // 无终止条件
}

int main() {
    recurse();
    return 0;
}
```

运行结果：

```
递归深度: 10000
递归深度: 20000
递归深度: 30000
...
Segmentation fault (core dumped)
```

每次调用 `recurse()`，都会在栈上创建一个新的栈帧。栈的大小是有限的（Linux 默认 **8MB**）：

<img src="/images/c-lang-stack-overflow.svg" alt="栈溢出：递归层数不断增加直到越界 Segfault" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

当栈空间耗尽，下一次分配栈帧就会越界访问不属于进程的内存——操作系统立刻终止进程，这就是**栈溢出（Stack Overflow）**，表现为 Segfault。

用 `ulimit -s` 可以查看当前栈大小限制（单位 KB）：

```bash
$ ulimit -s
8192
```

---

## 七、把调用栈看成一摞盘子

理解栈帧最直觉的比喻：**函数调用就像摞盘子**。

- 每次调用函数，放一个新盘子在最上面
- 函数返回，拿走最上面的盘子
- 只能操作最上面的盘子（当前正在执行的函数）
- 摞太高（递归太深），盘子倒了（栈溢出）

<img src="/images/c-lang-stack-plates.svg" alt="调用栈盘子比喻：bar/foo/main 依次叠放" style="max-width:100%; height:auto; display:block; margin:1.5rem auto;" />

每个盘子（栈帧）有自己独立的局部变量，互不干扰。`foo` 里的 `b` 和 `bar` 里的 `c`，虽然名字不同，地址也不同，确实是两块完全独立的内存。

---

## 小结：三个困惑，一张地图

| 困惑 | 原因 | 结论 |
|------|------|------|
| 改了参数，原变量没变 | 参数是副本，存在不同栈帧 | C 永远值传递；要改原变量，传指针 |
| 不能返回局部变量的地址 | 函数返回时栈帧销毁，内存失效 | 需要跨函数共享的数据，放堆上 |
| 递归太深崩溃 | 栈空间有限，每帧占用固定空间 | 深递归改迭代，或增大栈限制 |

---

## 动手练习

1. **观察栈帧地址**：写一个 `main → A → B → C` 的三层调用链，在每层打印局部变量地址，验证地址单调递减。

2. **值传递验证**：写 `swap(int a, int b)` 和 `main` 里调用它，打印调用前后的地址和值。为什么交换失败？要怎么修改才能真正交换？（提示：这是第 3 篇的引子）

3. **（进阶）测量栈帧大小**：两次相邻的递归调用里，打印 `&depth` 的地址差值，这就是每帧的大致大小（包含对齐和元数据）。

---

**下一篇：** 指针不是魔法：地址、解引用与数组  
→ 既然传地址能修改原变量，指针到底是什么？数组名又是什么？
