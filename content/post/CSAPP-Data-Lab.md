---
title: CSAPP Data Lab
date: 2023-01-10 21:10:14+0800
categories:
    - Labs
    - CSAPP
tags:
    - CSAPP
---

## isAsciiDigit

### 任务

判断 x 的值是否在 0 和 9 对应的ASCII码（0x30 和 0x39）之间。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 15

### 思路

计算 x – 0x30 和 0x39 – x 的值，如果两个值都大于0，则说明 x 的值在两个数之间。

### 代码实现

```C
int isAsciiDigit(int x)
{
  int i = ~(0x30) + 1;  //i = -(0x30)
  int j = ~x + 1;  //j = -x
  int a = x + i;
  int b = 0x39 + j;
  //if a,b are not nagetive, return 1
  return (!(a >> 31)) &(!(b >> 31));
}
```

## anyEvenBit

### 任务

判断 x 的偶数位是否有 1 。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 12

### 思路

构造一个所有偶数位都为 1 的数，与 x 进行 & 操作，一次获取 x 偶数位的相关信息。

### 代码实现

```C
int anyEvenBit(int x)
{
  int a = 85; // a = 01010101;
  // x 偶数位上有 1，b 是正数
  // x 偶数位上没有 1，b 是 0
  int b = x & ((a << 24) + (a << 16) + (a << 8) + a);
  b = b + (~1 + 1); // b = b - 1;
  // b 是负数，返回 0，否则返回 1
  return !(b >> 31);
}
```

## copyLSB

### 任务

将 x 的所有位设置为 x 的最低位的值。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 5

### 思路

利用算数移位的特性，将符号位置为 x 的最低位，再右移 31 位。

### 代码实现

```C
int copyLSB(int x)
{
  x = x << 31;
  x = x >> 31;
  return x;
}
```

## leastBitPos

### 任务

返回一个标记着 x 最低位的 1 的掩码。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 6

### 思路

令 a 的值等于 x 取反加一，若 x 从右往左数第 i 位是第一个 1，则 a 的从右往左第 i 位也会是 1，第 a 与 x 的第 i 位向左的所有位的值都是 0，第 i 位向右的所有位都是相反的。因此，最终返回 x & a 。

### 代码实现

```C
int leastBitPos(int x) {
  int a = ~x + 1;
  return x & a;
}
```

## divpwr2

### 任务

计算 x / (2 ^ n) ，向 0 取整，0 <= n <= 30。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 15

### 思路

首先直接将 x 向右移 n 位，这样得到 x / (2 ^ n) 向下取整的结果。如果 x 是负数，并且 2 ^ n 不能整除 x，则需要将结果加一。

### 代码实现

```C
int divpwr2(int x, int n)
{
  // 如果 x 是非负数，op 为 1
  // 如果 x 是负数，op 为 0
  int op = (x >> 31) + 1;
  int a = x >> n;
  // 判断是否整除
  int b = a << n;
  int is_exact_division = !!(b ^ x);
  // 若 x 是负数且不能整除，结果需要加一
  return (a + ((!op) & is_exact_division));
}
```

## bitCount

### 任务

计算 x 中 1 的个数。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 40

### 思路

使用类似分治的思想，先分别求出 x 中每两位的 1 的个数，再求出 x 中每四位的 1 的个数，再求出 x 中每八位 的 1 的个数，最后将每个八位的 1 的个数相加得到 x 中 1 的个数。

### 代码实现

```C
int bitCount(int x)
{
  int a = 0x55;
  int b = 0x33;
  int c = 0x0f;
  int ans = 0;
  // a = 0101 0101 ... 0101 0101
  a = a | (a << 8);
  a = a | (a << 16);
  // b = 0011 0011 ... 0011 0011
  b = b | (b << 8);
  b = b | (b << 16);
  // c = 0000 1111 ... 0000 1111
  c = c | (c << 8);
  c = c | (c << 16);
  // 每两位的值代表 x 中两位的 1 的个数
  ans = (x & a) + ((x >> 1) & a);
  // 每两位的值代表 x 中四位的 1 的个数
  ans = (ans & b) + ((ans >> 2) & b);
  // 每两位的值代表 x 中八位的 1 的个数
  ans = (ans & c) + ((ans >> 4) & c);
  // 将每个八位的值求和得出 x 中 1 的个数
  ans = (ans + (ans >> 8) + (ans >> 16) + (ans >> 24)) & 0xff;
  return ans;
}
```

## conditional

### 任务

返回 x ? y : z 的值。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 16

### 思路

利用算数移位，在 x 不为 0 时将 x 设置为 0xfff...ff，x 为0 时 x 就为 0，则返回 (x & y) | ((~x) & z) 即可。

### 代码实现

```C
int conditional(int x, int y, int z)
{
  int a, b;
  // x 不为 0 时将 x 设为 1
  // x 为 0 时 x 还会是 0
  x = !!x;
  // x 如果为 1，x 将会等于 0xfff...ff
  // x 如果为 0，x 将会等于 0
  x = x << 31;
  x = x >> 31;
  a = x & y;
  b = (~x) & z;
  return a | b;
}
```

## isNonNegative

### 任务

如果 x >= 0，返回1，否则返回 0。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 6

### 思路

将 x 右移 31 位得到 x 的符号位，如果 x >= 0，右移后结果为 0，否则右移后结果为 1。

### 代码实现

```C
int isNonNegative(int x)
{
  //取符号位
  x = x >> 31;
  x = !x;
  return x;
}
```

## isGreater

### 任务

如果 x > y 返回 1, 否则返回 0。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 24

### 思路

根据 x - y 的结果判断 x，y 之间的大小关系。如果 x，y 同号，则 x - y 结果不会溢出，x - y 大于 0 时返回 1，否则返回 0；如果 x，y 不同号，y 小于 0 时返回 1，否则返回 0。

### 代码实现

```C
int isGreater(int x, int y)
{
  int flagX, flagY;  //x，y 符号,大于等于 0 为 0 ，小于 0 为 1
  int flag;  //x, y 符号相同为 0，不同为 1
  int a;  //y - x 大于等于 0 为 0， 小于 0 为 1
  flagX = (x >> 31) << 31;
  flagX = !!flagX;
  flagY = (y >> 31) << 31;
  flagY = !!flagY;
  flag = flagX ^ flagY;
  a = ((y + (~x) + 1) >> 31) << 31;
  a = !!a;
  return ((~flag) & a) | (flag & flagY);
}
```

## absVal

### 任务

返回 x 的绝对值。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 10

### 思路

利用算数移位，在 x 不为 0 时将 x 设置为 0xfff...ff。

### 代码实现

```C
int absVal(int x)
{
  int flag;
  // 符号表示，x 大于等于 0 为 0，小于 0 为 0xfff...ff
  flag = x >> 31;
  return (x & (~flag)) | (flag & ((~x) + 1));
}
```

## isPower2

### 任务

如果 x 是 2 的次幂，返回 1，否则返回 0。

### 要求

Legal ops: ! ~ & ^ | + << >>
Max ops: 20

### 思路

满足条件的 x 满足 x & (x - 1) = 0，但需要排除 0 和 0x80000000。

### 代码实现

```C
int isPower2(int x)
{
  int a,b;
  // a = !(x & (x - 1))
  a = !(x & (x + ~0));
  // 如果 x 是 0 和 0x80000000，值为 0
  b = ~(x >> 31) & (!!x);
  return a & b;
}
```

## float_neg

### 任务

将 unsigned 视为存储的 float 类型的数，返回 x 的相反数。如果 x 是 NaN，原样返回 x。

### 要求

Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
Max ops: 10

### 思路

判断是否是 NaN：取出阶码和尾数，如果阶码全为 1 且尾数不等于 0，则是一个 NaN，原样返回。否则，将符号位与 1 异或后返回。

### 代码实现

```C
unsigned float_neg(unsigned uf)
{
  unsigned exp;
  unsigned frac;
  exp = uf << 1 >> 24;
  frac = uf << 9 >> 9;
  // 如果是 NaN
  if (exp == 255 && frac != 0)
    return uf;
  return uf ^ (1 << 31);
}
```

## float_i2f

### 任务

返回 (float) x 在二进制编码所对应的无符号数。即为把 (float) x 的位表示放在一个无符号变量中返回

### 要求

Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
Max ops: 30

### 思路

根据浮点数规则进行操作。

### 代码实现

```C
unsigned float_i2f(int x)
{
  unsigned sign = 0;
  unsigned exp;
  unsigned frac;
  unsigned shift = 0, tmp, rounding = 0;
  if(x == 0)
    return x;
  if(x < 0){
    x = -x;
    sign = 1;
  }
  frac = x;
  //获得 x 的最高有效位，即确定 frac 的位数。
  while(1){
    tmp = frac;
    shift ++;
    frac <<= 1;
    if(tmp & 0x80000000) break;
  }
  // 计算出 exp 的值
  exp = (159 - shift) << 23;
  // 出现在规定位置后大于0x100的情况就进1
  if((frac & 0x1ff) > 0x100) rounding = 1;
  // 出现最后一位为1，且下一位为1的情况也进1(向偶取整)
  if((frac & 0x3ff) == 0x300) rounding = 1;
  frac = (frac >> 9) + rounding;
  return (sign << 31) + exp + frac;
}
```
