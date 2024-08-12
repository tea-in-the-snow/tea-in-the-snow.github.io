---
title: "KMP算法"
date: 2022-11-12 11:37:42+0800
comments: true
categories:
    - Algorithms
tags:
    - KMP
---

KMP算法是一个字符串匹配算法，或者说是一个子字符串匹配算法

算法在字符串str中搜索子串pattern，如果存在，返回这个子串的起始索引，如果不存在，返回-1

## 暴力枚举匹配

暴力的字符串匹配算法非常简单，就是枚举所有可能的起始索引 i，对于每一个索引，都通过与pattern逐个字符地进行比较来检查是否匹配，代码如下：

```java
static int matchString(String str, String pattern) {
  for (int i = 0; i <= str.length() - pattern.length(); ++i) {
      for (int j = 0; j < pattern.length(); ++j) {
          if (str.charAt(i + j) != pattern.charAt(j)) {
              break;
          }
          if (j == pattern.length() - 1) {
              return i;
          }
      }
  }
  return -1;
}
```

显而易见，这种算法在一些比较极端的情况下的运行情况非常不理想

如 str = "aaaabcacbab"，pattern = "aaaac" 时

```
在 i = 0 时，i 和 j 增加到4发现不匹配
        i
a a a a b c b a b
a a a a c
        j

接着 i 回退到 1，又经过了3次比较发现不匹配
        i
a a a a b c b a b
  a a a a c
        j
```

显然做了很多冗余的比较，而KMP就是一种对于这些极端情况也能够高效处理的算法

## 什么是next数组

在上面的例子中，如果是人进行模拟，在 i = 0 的4次比较发现不匹配后，显然没有必要将 i 回退到1，j 回退到0再开始比较，而是可以不回退i，直接从 j = 3 开始比较。

这实际上是由于在第一次的比较中，我们得到了 str[0 ~ 3] = pattern[0 ~ 3] 的信息。因此，不需要 i 再回退到1去依次访问和比较 str[1]，str[2]，str[3]，而可以保持 i 不动，通过 pattern 自身的性质让 j 进行跳转，继续进行比较

而对于每一个 j，当出现 str[i] != pattern[j] 时，j 进行跳转后的下一个位置就是 next[j]

## 如何计算 next 数组

首先根据上面的定义，i 和 j 满足 str[i - j ~ i - 1] = pattern[0 ~ j - 1], 将 next[j] 暂时记为 k，i 和 k 要满足 str[i - k ~ i] = pattern[0 ~ k - 1]

这样一看，j 和 k 岂不是相等的？那是因为还少了几个约束。首先，由于 i 和 j 取旧的值时已经确定不能匹配，因此 k（即新的 j）的值不能和j相等。而如果有 k > j 满足 str[i - k ~ i] = pattern[0 ~ k -1]，在之前一定被处理过。因此，k 一定会小于 j。并且，为了不漏过可能的子串，k 应该是满足条件的最大值

那么，根据 str[i - j ~ i - 1] = pattern[0 ~ j - 1]，可以得到 str[i - k ~ i - 1] = pattern[j - k ~ j - 1]

再根据 str[i - k ~ i - 1] = pattern[0 ~ k - 1]，得到 pattern[0 ~ k - 1] = pattern[j - k ~ j - 1]

也就是说，对于每一个 j，k = next[j] 是满足 pattern[0 ~ k - 1] = pattern[j - k ~ j - 1]的最大的k，直观来描述，就是在j之前的串中，前 k 个位置和后 k 个位置相同，k的最大值

我们把在一个字符串中前 k 个位置和后 k 个位置相同，k的最大值记为字符串的 X 长度，相同的前 k 个位置和后 k 个位置记为 X 前缀和 X 后缀

要求满足条件的 k，就只需要考虑pattern了，首先考虑 k 和 next[j - 1]的关系，t = next[j - 1]是满足 pattern[0 ~ t - 1] = pattern[j - 1 - t ~ j - 2] 的最大值，就是j - 1之前的串的 X 长度，显然，当 pattern[j - 1] = pattern[t] 时，k = t + 1

那么当 pattern[j - 1] ！= pattern[t]的时候呢，可以通过向前迭代 t 来继续进行比较，令 t = next[t]，如果 pattern[j - 1] = pattern[t] 时，k = t + 1，如果不等，则继续迭代，令 t = next[t]

上面的迭代为什么是正确的呢，因为当 pattern[j - 1] ！= pattern[t] 时，我们考虑 j - 1 之前的串的 X 后缀的 X 前缀和 X 后缀，j - 1之前的串的 X 前缀的 X 前缀和 j - 1 之前的串的 X 后缀的X 后缀是相等的，又因为 X 的最长特性，显然是下一个检查的对象，也就是令 t = next[t] 进行迭代，这样的迭代可以通过递归来考虑，终止条件就是当 t 迭代到 0 的时候

```
X 前缀
--------- t           j - 1
XXX***XXX   XXX***XXX
            ---------
            X 后缀
//X 前缀中***部分第一个位置即为 next[t]
```

结合上面的算法，考虑一些边界情况，最终写出代码如下:

```java
static int matchString(String str, String pattern) {
  if (str.length() < pattern.length() || str.isEmpty()) {
      return -1;
  }
  int[] next = new int[pattern.length()];
  //计算next数组的值
  next[0] = -1;
  for (int i = 1; i < next.length; ++i) {
      int k = i - 1;
      while (k > 0 && pattern.charAt(i - 1) != pattern.charAt(next[k])) {
          k = next[k];
      }
      next[i] = next[k] + 1;
  }
  //进行匹配
  int i = 0, j = 0;
  while (i < str.length()) {
      while (i < str.length() && str.charAt(i) == pattern.charAt(j)) {
          i++;
          j++;
          if (j == pattern.length()) {
              return i - j;
          }
      }
      j = next[j];
      //终止条件，j 需要移动到 pattern 的第一个位置，那么 i 就需要向后移动一位
      if (j < 0) {
          j = 0;
          i++;
      }
  }
  return -1;
}
```
