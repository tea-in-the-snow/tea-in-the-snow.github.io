---
title: "Ocaml List"
date: 2024-07-28 01:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

## 创建list

```OCaml
[]
e1 :: e2 :: [e3; e4; e5; e6]
[e1; e2; e3]
```

- list的所有元素类型要相同。

## Pattern Matching

```OCaml
match not true with | true -> "true" | false -> "false"
```

下划线表示任何值。

```Ocaml
match "foo" with | "bar" -> "foo" | _ -> "others"
```

与list进行匹配。

注意：对于 a::b，a的类型是int，b的类型是int list。

```OCaml
match [1; 2] with | [] -> -1 | a :: b -> a
```

定义一个递归函数sum，参数是lst，对list求和。

```OCaml
let rec sum lst = 
    match lst with
    |   [] -> 0
    |   h :: t -> h + sum t
```

求list的长度(Ocaml的库中有对应函数List.length)。

```Ocaml
let rec length lst =
  match lst with
  | [] -> 0
  | _ :: t -> 1 + length t
```

## :: vs @

### ::

- "cons"
- 在一个list的前面加上一个元素
- 'a -> 'a list -> 'a list
- O(1)

### @

- "append"
- 将两个list合并
- 'a list -> 'a list -> 'a list
- O(length lst1) (对于lst1 @ lst2)

## 不可修改

list中元素的值是不可修改的，想要修改list中元素的值，要新建一个list。

使用这种方法，编译器会让新老list共享相同的部分（正是得益于list的不可修改的特性）。

## function 关键字

```OCaml
let rec sum lst = match lst with
    | [] -> 0
    | h :: t -> h + sum t
```

等价于：

```OCaml
let rec sum = function
    | [] -> 0
    | h :: t -> h + sum t
```
