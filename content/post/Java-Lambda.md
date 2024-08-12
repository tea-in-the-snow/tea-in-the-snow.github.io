---
title: "Java中的lambda表达式"
date: 2024-07-18 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Java
    - Lambda
---

## 什么是lambda表达式

lambda表达式是一个简短的代码块，它接受参数并返回一个值。lambda 表达式与方法类似，但它们不需要名称，并且可以直接在方法中实现。

## lambda表达式有什么作用

- 允许将函数视为方法参数，或将代码视为数据。
- 可以创建不属于任何类的函数。
- lambda 表达式可以像对象一样传递并按需执行。

## lambda表达式的语法

```java
parameter -> expression

(parameter1, parameter2) -> expression

(parameter1, parameter2) -> { code block }
```

## 简单的例子

```java
import java.util.ArrayList;

public class Main {
    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
        numbers.add(4);
        numbers.add(5);
        // 打印ArrayList中所有元素
        numbers.forEach((n) -> System.out.println(n));
        // 打印ArrayList中所有偶数
        numbers.forEach((n) -> {
            if (n % 2 == 0) {
                System.out.println(n);
            }
        });
    }
}
```

如果变量的类型是只有一个方法的接口，则Lambda表达式可以存储在变量中。Lambda表达式应该具有与该方法相同数量的参数和相同的返回类型。Java 内置了许多此类接口，例如Consumer接口（表示接受单个输入参数且不返回结果的操作。）。

```java
import java.util.ArrayList;
import java.util.function.Consumer;

public class Main {
    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
        numbers.add(4);
        numbers.add(5);
        Consumer<Integer> method = (n) -> {
            if (n % 2 == 0) {
                System.out.println(n);
            }
        };
        numbers.forEach(method);
    }
}
```

lambda表达式可以作为函数的参数。

```java
import java.util.ArrayList;

interface NumFunction {
    boolean apply(int s);
}

public class Main {
    private static void printNumWithCondition(ArrayList<Integer> nums, NumFunction condition) {
        nums.forEach((num) -> {
            if (condition.apply(num)) {
                System.out.println(num);
            }
        });
    }

    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
        numbers.add(4);
        numbers.add(5);
        NumFunction isEven = (num) -> num % 2 == 0;
        printNumWithCondition(numbers, isEven);
    }
}
```

注：以上代码也可以用匿名内部类实现。即lambda可以简化一些匿名内部类的书写。

```java
import java.util.ArrayList;

interface NumFunction {
    boolean apply(int s);
}

public class Main {
    private static void printNumWithCondition(ArrayList<Integer> nums, NumFunction condition) {
        nums.forEach((num) -> {
            if (condition.apply(num)) {
                System.out.println(num);
            }
        });
    }

    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
        numbers.add(4);
        numbers.add(5);
        // 使用匿名内部类
        NumFunction isEven = new NumFunction() {
            @Override
            public boolean apply(int s) {
                return s % 2 == 0;
            }
        };
        printNumWithCondition(numbers, isEven);
    }
}
```
