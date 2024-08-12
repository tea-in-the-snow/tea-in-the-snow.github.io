---
title: CSAPP Cache Lab
date: 2023-01-10 01:10:23+0800
categories:
    - Labs
    - CSAPP
tags:
    - CSAPP
    - cache
---

## 任务A

### 任务主要内容分析

- 编写一个 cache 仿真程序，使用valgrind的内存跟踪记录作为输入，模拟高速缓存的命中/未命中行为，然后输出总的命中次数，未命中次数和缓存块的替换次数。
- 实验给出了一个可执行程序 csim-ref，要求实现的仿真程序和 csim-ref 功能相同。
- 仿真程序模拟的 cache 的 s，e，b 参数以输入参数的形式输入。
- 忽略指令加载的操作，要处理的指令包括：L : read，S : write，M : read then write。
- 使用 LRU 算法作为缓存块的替换策略

仿真程序实现 LRU 算法的方式为为每个缓存行添加一个 log 字段，用来记录最近访问情况。每次有更新时将每个有效的缓存行的 log 的值加 1，在每次访问后将对应的缓存行 log 的值设置为 1，在需要替换时选择替换 log 的值最大的缓存块。

仿真程序如下：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "cachelab.h"

#define BUFF_SIZE 25
#define BEGIN_STATE 0
#define EMPTY_LINE 1
#define REPLACE_LINE 2
#define HIT_LINE 3

void printUsage()
{
    printf("Usage: ./csim-ref [-hv] -s <num> -E <num> -b <num> -t <file>\n");
}

void handleFormetError()
{
    fprintf(stderr, "./csim-ref: Missing required command line argument\n");
    fprintf(stderr, "Usage: ./csim-ref [-hv] -s <num> -E <num> -b <num> -t <file>\n");
}

// 缓存行
typedef struct cacheLine
{
    unsigned int valid; // 有效位
    unsigned int tag; // 标记位
    unsigned int log; // 记录最近访问情况
                      /*每次有更新时将每个有效的行的 log 的值加 1，
                      在每次访问后将对应行的 log 的值设置为 1，
                      需要替换时选择替换 log 的值最大的缓存块
                      */
} CacheLine;

// 缓存组
typedef struct cacheSet
{
    CacheLine *lines;
} CacheSet;

// 缓存模拟
typedef struct cache
{
    CacheSet *sets;
    int s, e, b;
} Cache;

typedef struct testLog
{
    int miss, hit, eviction;
} TestLog;

// 初始化模拟缓存
Cache *initCache(int s, int e, int b)
{
    int setNum = 1 << s;
    Cache *newCache = malloc(sizeof(Cache));
    newCache->s = s;
    newCache->e = e;
    newCache->b = b;
    newCache->sets = malloc(sizeof(CacheSet) * setNum);
    for (int i = 0; i < setNum; ++i)
    {
        newCache->sets[i].lines = malloc(sizeof(CacheLine) * e);
        for (int j = 0; j < e; ++j)
        {
            newCache->sets[i].lines[j].log = 0;
            newCache->sets[i].lines[j].tag = 0;
            newCache->sets[i].lines[j].valid = 0;
        }
    }
    return newCache;
}

// 进行一次更新（读或写操作）
void update(Cache *cache, unsigned int address, unsigned int length, TestLog *log, int version)
{
    unsigned int setNum, tag, blockOffset;
    tag = address >> (cache->b + cache->s);
    blockOffset = (address << (64 - cache->b)) >> (64 - cache->b);
    setNum = (address << (64 - cache->s - cache->b)) >> (64 - cache->s);
    CacheLine *lines = cache->sets[setNum].lines;
    int lineNum = 0;
    int state = BEGIN_STATE;
    for (int i = 0; i < cache->e; ++i)
    {
        if(lines[i].valid)
        {
            lines[i].log++;
            if(lines[i].tag == tag)
            {
                lineNum = i;
                state = HIT_LINE;
            }
        }
    }
    // 如果命中
    if (state == HIT_LINE)
    {
        log->hit++;
        if(version)
            printf("hit ");
        lines[lineNum].log = 0;
    }
    // 如果没有命中
    else
    {
        log->miss++;
        if(version)
            printf("miss ");
        // 确定要替换的行（或空行）
        for (int i = 0; i < cache->e; ++i)
        {
            if(lines[i].valid)
            {
                if(state == BEGIN_STATE)
                {
                    lineNum = i;
                    state = REPLACE_LINE;
                }
                else if(state == REPLACE_LINE)
                {
                    if(lines[lineNum].log < lines[i].log)
                        lineNum = i;
                }
            }
            else
            {
                if(state != EMPTY_LINE)
                {
                    lineNum = i;
                    state = EMPTY_LINE;
                }
            }
        }
        // 进行替换
        if (state == REPLACE_LINE)
        {
            lines[lineNum].log = 0;
            lines[lineNum].tag = tag;
            log->eviction++;
            if(version)
                printf("eviction ");
        }
        // 填入空行
        else if(state == EMPTY_LINE)
        {
            lines[lineNum].log = 0;
            lines[lineNum].tag = tag;
            lines[lineNum].valid = 1;
        }
    }
    // 判断要更新的内存长度是否超出一个缓存块
    unsigned int maxOffset = 1;
    for (int i = 0; i < cache->b; ++i)
        maxOffset = (maxOffset << 1) + 1;
    unsigned int temp = maxOffset - blockOffset + 1;
    if (maxOffset < blockOffset + length - 1)
        update(cache, address + temp, length - temp, log, version);
}

void testCache(FILE *fp, TestLog *log, int s, int e, int b, int version)
{
    Cache *cache = initCache(s, e, b);
    char command[BUFF_SIZE] = {0};
    char operation;
    unsigned int address, length;
    while (fgets(command, BUFF_SIZE, fp))
    {
        if (command[0] != ' ')
            continue;
        sscanf(command, " %c%x,%x\n", &operation, &address, &length);
        if(version)
            printf("%c %x,%x ", operation, address, length);
        switch (operation)
        {
        case 'L':
            update(cache, address, length, log, version);
            break;

        case 'S':
            update(cache, address, length, log, version);
            break;

        case 'M':
            update(cache, address, length, log, version);
            update(cache, address, length, log, version);
            break;

        default:
            break;
        }
        if(version)
            printf("\n");
        memset(command, 0, BUFF_SIZE);
    }
}

int main(int argc, char const *argv[])
{
    int s; // 序号所占的位宽
    int e; // 每组的缓存块个数
    int b; // 偏移量所占的位宽
    int t; // argv[t]为跟踪文件
    int version = 0; // 指示 -v 参数
    int se = 0, ee = 0, be = 0, te = 0; // 判断 s,e,b,t 参数是否给出
    FILE *fp; // 跟踪文件
    if (argc == 2)
    {
        if (strcmp(argv[1], "-h"))
        {
            handleFormetError();
            return EXIT_FAILURE;
        }
        printUsage();
        return EXIT_SUCCESS;
    }
    else if (argc == 9 || argc == 10)
    {
        for (int i = 1; i < argc; ++i)
        {
            if (!strcmp(argv[i], "-s"))
            {
                s = atoi(argv[++i]);
                se = 1;
            }
            else if (!strcmp(argv[i], "-E"))
            {
                e = atoi(argv[++i]);
                ee = 1;
            }
            else if (!strcmp(argv[i], "-b"))
            {
                b = atoi(argv[++i]);
                be = 1;
            }
            else if (!strcmp(argv[i], "-t"))
            {
                t = ++i;
                te = 1;
            }
            else if (!strcmp(argv[i], "-v"))
                version = 1;
        }
        if (!(se && ee && be && te))
        {
            handleFormetError();
            return EXIT_FAILURE;
        }
        if (argc == 10 && !version)
        {
            handleFormetError();
            return EXIT_FAILURE;
        }
    }
    else
    {
        handleFormetError();
        return EXIT_FAILURE;
    }
    // 打开跟踪文件
    fp = fopen(argv[t], "r");
    if (fp == NULL)
    {
        fprintf(stderr, "%s: No such file or directory\n", argv[t]);
        return EXIT_FAILURE;
    }
    // 初始化测试记录
    TestLog *log = malloc(sizeof(TestLog));
    log->eviction = 0;
    log->hit = 0;
    log->miss = 0;
    // 进行模拟测试
    testCache(fp, log, s, e, b, version);
    printSummary(log->hit, log->miss, log->eviction);
    return 0;
}
```

测试结果：

```json
❯ ./test-csim
                        Your simulator     Reference simulator
Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
     3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
     3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
     3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
     3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
     3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
     3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
     3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
     6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
    27

TEST_CSIM_RESULTS=27
```

## 任务B

### 任务主要内容分析

- 实验要求优化矩阵转置程序，尽量减少程序对高速缓存访问的未命中次数。
- 实验已经给定了 cache 的相关参数：s = 5，e = 1，b = 5；因此，cache 共有32 组，每组包括一个缓存行，每行能够存储32字节的数据即 8 个 int 类型的数据。
- 实验要求转置的矩阵共有三种，分别是 32 *32，64* 64，61 * 67，每一种情况需要分别考虑。
- 实验要求定义的 int 局部变量总数最多不能超过 12 个。
- 实验给出了优化提示：使用分块技术。

### 32 * 32

根据 cache 的参数，矩阵的 8 行恰好占满整个 cache。

结合使用分块技术的提示，每次处理一个 8 *8 的矩阵。A 中的一个 8* 8 的矩阵 a 需要在转置后存储在 B 中一个 8 * 8 的矩阵 b 的位置。

代码实现如下：

```C
int temp;
for(int i = 0; i < 4; ++i)
{
  for(int j = 0; j < 4; ++j)
  {
    for(int k = 0; k < 8; ++k)
    {
      for(int t = 0; t < 8; ++t)
      {
        temp = A[i * 8 + k][j * 8 + t];
        B[j * 8 + t][i * 8 + k] = temp;
      }
    }
  }
}
```

测试结果：

```json
❯ ./test-trans -M 32 -N 32

Function 0 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:1709, misses:344, evictions:312

Function 1 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:869, misses:1184, evictions:1152

Summary for official submission (func 0): correctness=1 misses=344

TEST_TRANS_RESULTS=1:344
```

实验要求 miss 次数要小于 300，因此还需要继续优化。

注意到在矩阵 a 主对角线上的元素与矩阵 b 主对角线上的元素会映射到 cache 中的同一行，如果采用上面的写法，在读取完 a 主对角线上的元素后写入 b 时会将 cache 中 a 的一行替换掉，这样在接着读取 a 这一行后面的元素时就会产生未命中。如果利用局部变量一次性将 a 的一行全部读出来，再进行后续的写入，就可以避免这个问题。

修改后的代码如下：

```C
int x0, x1, x2, x3, x4, x5, x6, x7;
for (int i = 0; i < 4; ++i)
{
  for (int j = 0; j < 4; ++j)
  {
    for (int k = 0; k < 8; ++k)
    {
      x0 = A[i * 8 + k][j * 8];
      x1 = A[i * 8 + k][j * 8 + 1];
      x2 = A[i * 8 + k][j * 8 + 2];
      x3 = A[i * 8 + k][j * 8 + 3];
      x4 = A[i * 8 + k][j * 8 + 4];
      x5 = A[i * 8 + k][j * 8 + 5];
      x6 = A[i * 8 + k][j * 8 + 6];
      x7 = A[i * 8 + k][j * 8 + 7];
      B[j * 8][i * 8 + k] = x0;
      B[j * 8 + 1][i * 8 + k] = x1;
      B[j * 8 + 2][i * 8 + k] = x2;
      B[j * 8 + 3][i * 8 + k] = x3;
      B[j * 8 + 4][i * 8 + k] = x4;
      B[j * 8 + 5][i * 8 + k] = x5;
      B[j * 8 + 6][i * 8 + k] = x6;
      B[j * 8 + 7][i * 8 + k] = x7;
    }
  }
}
```

测试结果达到要求：

```json
❯ ./test-trans -M 32 -N 32

Function 0 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:1765, misses:288, evictions:256

Function 1 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:869, misses:1184, evictions:1152

Summary for official submission (func 0): correctness=1 misses=288

TEST_TRANS_RESULTS=1:288
```

### 64 * 64

由于矩阵大小的变化，矩阵的 4 行就会占满整个 cache。在一个 8 *8 的分块中，第 0，1，2，3 行和第 4，5，6，7 行会映射到 cache 的同一行上，在写 cache 时会出现 conflict miss，因此，不能再像前面那样分成 8* 8 的矩阵进行处理。

一个显然的想法是将 8 *8 的分块变成 4* 4 的分块进行处理，代码如下：

```C
int tmp[4];
for (int i = 0; i < 16; ++i)
{
  for (int j = 0; j < 16; ++j)
  {
    for (int k = 0; k < 4; ++k)
    {
      x0 = A[i * 4 + k][j * 4];
      x1 = A[i * 4 + k][j * 4 + 1];
      x2 = A[i * 4 + k][j * 4 + 2];
      x3 = A[i * 4 + k][j * 4 + 3];4
      B[j * 4][i * 4 + k] = x0;
      B[j * 4 + 1][i * 4 + k] = x1;
      B[j * 4 + 2][i * 4 + k] = x2;
      B[j * 4 + 3][i * 4 + k] = x3;
    }
  }
}
```

测试结果：

```json
❯ ./test-trans -M 64 -N 64

Function 0 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:6497, misses:1702, evictions:1670

Function 1 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:3473, misses:4724, evictions:4692

Summary for official submission (func 0): correctness=1 misses=1702

TEST_TRANS_RESULTS=1:1702
```

实验要求 miss 的次数小于 1300，简单地将 8 *8 的分块改为 4* 4 并不能达到要求。

分析一下，cache 每一行能够存储 8 个数，而上面的程序中每次只读取 4 个数，对 cache 的利用率比较低。要继续优化，应该想办法提高 cache 的利用率。

实验限制了局部变量的个数，但可以将需要利用 B 矩阵进行暂存。

要最大限度利用 cache，应该要尽量读取连续的 8 个数进行处理，因此必定需要对 8 *8 的矩阵进行分块处理。结合 4 行就占满了 cache 的性质，可以将 8* 8 的矩阵划分为 4 个 4 * 4 的矩阵进行处理。

首先，对于前四行的，将 A 左上角的转置矩阵存入 B 的左上角，将 A 右上角的转置矩阵存入 B 的右上角。可以通过每次读取一行 8 个数，将它们存到 B 矩阵的对应位置实现。

然后，取出 A 左下角的一列和 B 右上角的一行，将 A 左下角的一列存入 B 右上角的一行，将 B 右上角的一行存入 B 左下角的一行。

最后，将 A 右下角转置矩阵存入 B 右下角。

代码实现如下：

```C
int x0, x1, x2, x3, x4, x5, x6, x7;
for (int i = 0; i < 64; i += 8)
{
  for (int j = 0; j < 64; j += 8)
  {
    // 前 4 行处理
    // A 左上角转置矩阵存入 B 左上角
    // A 右上角转置矩阵存入 B 右上角
    for (int k = i; k < i + 4; ++k)
    {
      // 取出一行 8 个 int（即一个缓存行）
      x0 = A[k][j];
      x1 = A[k][j + 1];
      x2 = A[k][j + 2];
      x3 = A[k][j + 3];
      x4 = A[k][j + 4];
      x5 = A[k][j + 5];
      x6 = A[k][j + 6];
      x7 = A[k][j + 7];
      // A 左上角转置矩阵存入 B 左上角
      B[j][k] = x0;
      B[j + 1][k] = x1;
      B[j + 2][k] = x2;
      B[j + 3][k] = x3;
      // A 右上角转置矩阵存入 B 右上角
      B[j][k + 4] = x4;
      B[j + 1][k + 4] = x5;
      B[j + 2][k + 4] = x6;
      B[j + 3][k + 4] = x7;
    }
    // 取出 A 左下角的一列和 B 右上角的一行
    // A 左下角的一列存入 B 右上角的一行
    // B 右上角的一行存入 B 左下角的一行
    for (int k = 0; k < 4; ++k)
    {
      // 取出 B 右上角的一行
      x0 = B[j + k][i + 4];
      x1 = B[j + k][i + 5];
      x2 = B[j + k][i + 6];
      x3 = B[j + k][i + 7];
      // 取出 A 左下角的一列
      x4 = A[i + 4][j + k];
      x5 = A[i + 5][j + k];
      x6 = A[i + 6][j + k];
      x7 = A[i + 7][j + k];
      // A 左下角的一列存入 B 右上角的一行
      B[j + k][i + 4] = x4;
      B[j + k][i + 5] = x5;
      B[j + k][i + 6] = x6;
      B[j + k][i + 7] = x7;
      // B 右上角的一行存入 B 左下角的一行
      B[j + k + 4][i] = x0;
      B[j + k + 4][i + 1] = x1;
      B[j + k + 4][i + 2] = x2;
      B[j + k + 4][i + 3] = x3;
    }
    // A 右下角转置矩阵存入 B 右下角
    for (int k = i + 4; k < i + 8; ++k)
    {
      x4 = A[k][j + 4];
      x5 = A[k][j + 5];
      x6 = A[k][j + 6];
      x7 = A[k][j + 7];
      // A 右下角转置矩阵存入 B 右下角
      B[j + 4][k] = x4;
      B[j + 5][k] = x5;
      B[j + 6][k] = x6;
      B[j + 7][k] = x7;
    }
  }
}
```

测试结果达到要求：

```json
❯ ./test-trans -M 64 -N 64

Function 0 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:9017, misses:1228, evictions:1196

Function 1 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:3473, misses:4724, evictions:4692

Summary for official submission (func 0): correctness=1 misses=1228

TEST_TRANS_RESULTS=1:1228
```

### 61 * 67

和上面的 64 *64 相比，要求 miss 次数小于 2000，条件明显宽松，使用简单的 8* 8 分块尝试一下：

```C
for (int i = 0; i < N; i += 8)
{
  for (int j = 0; j < M; j += 8)
  {
    for (int k = i; k < i + 8 && k < N; k++)
    {
      for (int l = j; l < j + 8 && l < M; l++)
      {
        B[l][k] = A[k][l];
      }
    }
  }
}
```

测试结果已经达到了要求：

```json
❯ ./test-trans -M 61 -N 67

Function 0 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 0 (Transpose submission): hits:6186, misses:1993, evictions:1961

Function 1 (2 total)
Step 1: Validating and generating memory traces
Step 2: Evaluating performance (s=5, E=1, b=5)
func 1 (Simple row-wise scan transpose): hits:3755, misses:4424, evictions:4392

Summary for official submission (func 0): correctness=1 misses=1993

TEST_TRANS_RESULTS=1:1993
```

## 综合测试结果

```json
❯ ./driver.py
Part A: Testing cache simulator
Running ./test-csim
                        Your simulator     Reference simulator
Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
     3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
     3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
     3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
     3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
     3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
     3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
     3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
     6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
    27


Part B: Testing transpose function
Running ./test-trans -M 32 -N 32
Running ./test-trans -M 64 -N 64
Running ./test-trans -M 61 -N 67

Cache Lab summary:
                        Points   Max pts      Misses
Csim correctness          27.0        27
Trans perf 32x32           8.0         8         288
Trans perf 64x64           8.0         8        1228
Trans perf 61x67          10.0        10        1993
          Total points    53.0        53
```
