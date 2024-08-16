---
title: "抽象解释(Abstract Interpretation)"
description: 
date: 2024-08-16T12:30:57+08:00
hidden: false
comments: true
draft: false
categories:
  - Program Analysis
tags:
  - Theory
---

## 背景：程序分析(Program Analysis)

程序分析指的是分析程序行为事实的算法，比如查找哪些变量是常量、是否会出现空指针错误，或者每个“lock”操作是否紧接着一个“unlock”等。

程序分析主要分为两大类：静态分析和动态分析。动态分析的意思基本上就是测试：在一个或多个测试用例上运行程序并观察其行为。例如，你可以运行程序并查看是否出现任何空指针错误。动态分析的主要优点是可以获得有关程序行为的准确信息，因为正在实际运行它。缺点是只能获得所提供的测试输入的信息，并且只能获得这些测试输入所执行的代码路径的信息。因此，动态分析无法证明优化是安全的，也无法证明程序没有某些类型的安全漏洞。静态分析的目标是通过实现对程序的完全覆盖来解决此问题：使用特殊算法分析所有可能输入的行为，但代价是结果只是近似的。但结果只需在一个方向上近似即可。例如，你可以创建一个会发生误报（会在实际上没有错误的地方报告错误）但是会发现所有错误的程序分析，也可以创建一个不会出现误报但是会错过一些真正的bug的程序分析。

## 代码

下面是一段C++的示例代码，用来进行分析。我们通过一个程序分析“constant propagation”来理解抽象分析，目标是发现汉室每个点处使用的变量是否具有常量值。

```C++
int foo(int a, int b) {
  int k = 1;          // k is constant: 1
  int m, n;
  if (a == 0) {
    ++k;              // k is constant: 2
    m = a;
    n = b;
  } else {
    k = 2;            // k is constant: 2
    m = 0;
    n = a + b;
  }
  return k + m + n;   // k is constant: 2
}
```

## 具体分析

为了理解抽象分析，先从具体分析开始。

我们可以通过使用不同的输入来调用这个函数来试图猜测那些变量是常量。例如，输入“a = 0, b = 7”:

```terminal
 (instruction)        (interpreter state after instruction)
 ENTRY                a = 0, b = 7
 k := 1               a = 0, b = 7, k = 1
 COND a == 0          (TRUE)
 k := k + 1           a = 0, b = 7, k = 2
 m := a               a = 0, b = 7, k = 2, m = 0
 n := b               a = 0, b = 7, k = 2, m = 0, n = 7
 t1 := k + m          a = 0, b = 7, k = 2, m = 0, n = 7, t1 = 2
 ret := t1 + n        a = 0, b = 7, k = 2, m = 0, n = 7, t1 = 2, ret = 9
 RETURN ret
 EXIT
```

可以看到在k的值被使用的语句“t1 := k + m”之前，k的值是2。我们可以运行很多组输入而得到相同的结果，但是这样的具体测试不能证明对于所有的输入都是这样。

## 接近抽象分析

观察这个函数，会发现对于k来说，实际上只有两种重要的情况：“a == 0”和“a != 0”，b的值与k无关。所以输入可以归结为两种，“a = 0, b = ?”和“a = NZ, b = ?”，其中，NZ表示不是0的数(anay nonzero value)，?表示任何值。

对于“a = 0, b = ?”：

```terminal
 (instruction)        (interpreter state after instruction)
 ENTRY                a = 0, b = ?
 k := 1               a = 0, b = ?, k = 1
 COND a == 0          (TRUE)
 k := k + 1           a = 0, b = ?, k = 2
 m := a               a = 0, b = ?, k = 2, m = 0
 n := b               a = 0, b = ?, k = 2, m = 0, n = ?
 t1 := k + m          a = 0, b = ?, k = 2, m = 0, n = ?, t1 = 2
 ret := t1 + n        a = 0, b = ?, k = 2, m = 0, n = ?, t1 = 2, ret = ?
 RETURN ret
 EXIT
```

对于“a = NZ, b = ?”：

```terminal
 (instruction)        (interpreter state after instruction)
 ENTRY                a = NZ, b = ?
 k := 1               a = NZ, b = ?, k = 1
 COND a == 0          (FALSE)
 k := 2               a = NZ, b = ?, k = 2
 m := 0               a = NZ, b = ?, k = 2, m = 0
 n := a + b           a = NZ, b = ?, k = 2, m = 0, n = ?
 t1 := k + m          a = NZ, b = ?, k = 2, m = 0, n = ?, t1 = 2
 ret := t1 + n        a = NZ, b = ?, k = 2, m = 0, n = ?, t1 = 2, ret = ?
 RETURN ret
 EXIT
```

它看起来很像具体的情况，只是我们现在有一些抽象值，NZ 和“？”，它们代表具体值的集合。

到此为止就分析了全部的可能情况，但这并不是一个通用的方案。因为这只是一个简单的函数，所以我们确切地知道要使用那些抽象值进行测试，但这样的方法并不适用于复杂的函数，并且这不是程序自动的。

另一个问题是，我们正在单独分析函数中的每条路径。但是，具有 k 个 if 语句的函数最多可以有 2^k 条路径，而具有循环的函数则有无数条路径。因此，在实际程序中，我们几乎不可能获得完整的覆盖。

## 抽象解释的例子

在上面的分析方法中，一个问题是我们不知道应该使用那些值的集合，所以我们使用“a = ?, b = ?”进行测试：

```terminal
 (instruction)        (interpreter state after instruction)
 ENTRY                a = ?, b = ?
 k := 1               a = ?, b = ?, k = 1
 COND a == 0
```

由于我们不知道关于a的情况，所以让我们分析两个分支，首先是条件为真的情况：

```terminal
 (instruction)        (interpreter state after instruction)
                      a = ?, b = ?, k = 1
 k := k + 1           a = ?, b = ?, k = 2
 m := a               a = ?, b = ?, k = 2, m = ?
 n := b               a = ?, b = ?, k = 2, m = ?, n = ?
```

然后是条件为假的情况：

```terminal
 (instruction)        (interpreter state after instruction)
                      a = ?, b = ?, k = 1
 k := 2               a = ?, b = ?, k = 2
 m := 0               a = ?, b = ?, k = 2, m = 0
 n := a + b           a = ?, b = ?, k = 2, m = 0, n = ?
```

我们可以继续分别分析这两个分支，但这样在分支很多的情况下是不可行的，所以我们要通过合并状态的方法来合并分支。我们需要涵盖下面两个状态的一个解释器状态：

```terminal
a = ?, b = ?, k = 2, m = ?, n = ?
a = ?, b = ?, k = 2, m = 0, n = ?
```

可以通过合并每个变量的状态来合并状态，合并的结果是：

```terminal
a = ?, b = ?, k = 2, m = ?, n = ?
```

由此继续分析程序：

```terminal
 t1 := k + m          a = ?, b = ?, k = 2, m = ?, n = ?, t1 = ?
 ret := t1 + n        a = ?, b = ?, k = 2, m = ?, n = ?, t1 = ?, ret = ?
 RETURN ret
 EXIT
```

可以得到k是常量2。一些关键的idea是：

- 使用抽象值作为输入“运行”程序。
- 一个抽象值代表一个具体值的集合。
- 在控制流的分支处，两个分支分别分析。
- 在控制流的合并处，合并状态。

## 创建抽象解释

想要创建一个抽象解释，我们要指定抽象值、流函数和初始状态。通过改变这些，我们可以用相同的框架执行无数次的不同分析。

### 抽象值

每一个抽象解释都需要一个定义好的抽象值的集合。在一个标准的整形常量的传播分析(propagation analysis)中，抽象值有：

- T(top)，所有整数的集合
- 1，2，...，整数值的单例集
- ⊥(bottom)， 空集

要合并两个抽象值v1和v2，应该找到一个同时覆盖v1和v2的最小集合，实际上就是一个近似的并集。

### 流函数

抽象解释要求我们维护一个抽象解释器状态，然后在当前的状态下解释每一条指令。

流函数就是做这种解释的函数，输入是抽象状态和指令，输出是执行指令后的抽象状态。因此，流函数描述了每种指令类型的抽象语义。

例如，对于语句“x := K”，其中K是任何整数，很明显结果是x获得抽象值K。

### 初始状态

初始状态是函数最初进入时保存的值，对于常数传播分析(constant propagation)，初始状态是T，也即所有值。一些分析会需要其他的初始状态，例如，outparams分析是关于函数内部是否写入outparam，因此outparams分析的初始状态为NOT_WRITTEN。
