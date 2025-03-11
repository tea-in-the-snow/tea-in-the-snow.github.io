---
title: "Fuzzing简单了解"
description: 
date: 2025-03-11T09:14:29+08:00
categories:
    - Program Testing
tags:
    - Fuzzing
hidden: false
comments: true
draft: false
---

模糊测试（fuzzing）指的是通过反复运行程序并生成可能在语法和语义上存在错误的输入来进行的测试过程。模糊测试通过向系统引入意外的输入，并观察系统对这些输入是否产生任何负面反应，从而揭示出安全、性能或质量方面的差距或问题。

根据对目标程序的了解程度，Fuzzing可以被分为三种类型：

- Whitebox Fuzzing：在有目标程序的完整源代码或二进制代码的情况下进行，利用程序的结构信息来生成测试用例。其特点是更容易发现隐藏和深层的漏洞，但需要对代码或二进制进行深入分析，因此通常更为耗时。
- Greybox Fuzzing：介于Whitebox和Blackbox之间，具有部分程序内部知识。它结合了Blackbox的速度和Whitebox的深度；通常使用轻量级的代码覆盖率信息来指导测试。
- Blackbox Fuzzing：无需知道程序内部信息，只对程序的输入和输出进行测试，不依赖源代码或二进制分析。它速度更快，但可能错过一些深层次的漏洞。

根据测试数据的产生方式，Fuzzing可以被分为两种类型：

- Generation-based Fuzzing：即从零开始，根据某种预定义的规范或模型生成测试数据。
- Mutation-based Fuzzing：从现有的合法输入数据开始，通过应用一系列随机或半随机变化来产生新的测试用例。

根据输入生成的关注点和测试目标，Fuzzing可以被分为两种类型：

- Syntactic Fuzzing：语法模糊测试专注于生成符合输入格式（语法）但可能包含畸形成分或错误的输入。它不关心输入的实际语义或逻辑含义，生成的输入可能在语法上是有效的，但语义上可能是无效或无意义的。目标一般是测试程序的输入解析和错误处理逻辑；发现由于输入格式错误导致的崩溃或漏洞（如缓冲区溢出、空指针解引用等）。

- Semantic Fuzzing： 语义模糊测试不仅关注输入的语法正确性，还关注其语义有效性，即输入是否符合目标程序的逻辑和预期行为。。它尝试生成符合程序预期行为的输入，以测试核心逻辑。目标一般是测试程序的核心逻辑，而不仅仅是错误处理代码；发现由于输入语义错误导致的逻辑漏洞（如业务逻辑缺陷、安全漏洞等）。

随着软件复杂度的爆炸式增加，随机生成测试数据已经远不能满足需求。一种主流方案是用动态符号执行（concolic execution）辅助Fuzzing，实现上可以采用KLEE、Z3 Solver和LLVM等技术，自动解析条件代码语句生成输入.

后来，基于编译器插桩的coverage feedback driven Fuzzing开始受到关注，神器AFL横空出世（如今已停止维护，社区维护的版本是AFL++）。后继者有LLVM的libfuzzer、内核领域的syzkaller等，统称为Coverage-based Greybox Fuzzing（CGF）。