---
title: "OCaml Variants"
date: 2024-08-07 09:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

varientahi一个值为多种可能（一个有限的集合）的数据类型，类似于C和Java中的enum。

```OCaml
type day = Sun | Mon | Tue | Wed | Thu | Fri | Sat
let d = Tue
```

一个variant的值的名字在OCaml中叫做constructor，constructor的开头字母必须大写。

访问variant可以只用pattern matching：

```OCaml
let int_of_day d =
  match d with
  | Sun -> 1
  | Mon -> 2
  | Tue -> 3
  | Wed -> 4
  | Thu -> 5
  | Fri -> 6
  | Sat -> 7
```

## Variants that carry data

```OCaml
type point = float * float
type shape =
  | Point of point
  | Circle of point * float (* center and radius *)
  | Rect of point * point (* lower-left and upper-right corners *)
```

Here are a couple functions that use the `shape` type:

```OCaml
let area = function
  | Point _ -> 0.0
  | Circle (_, r) -> Float.pi *. (r ** 2.0)
  | Rect ((x1, y1), (x2, y2)) ->
      let w = x2 -. x1 in
      let h = y2 -. y1 in
      w *. h

let center = function
  | Point p -> p
  | Circle (p, _) -> p
  | Rect ((x1, y1), (x2, y2)) -> ((x2 +. x1) /. 2.0, (y2 +. y1) /. 2.0)
```

We could use this type to code up lists (e.g.) that contain either strings or ints:

```OCaml
type string_or_int_list = string_or_int list

let rec sum : string_or_int list -> int = function
  | [] -> 0
  | String s :: t -> int_of_string s + sum t
  | Int i :: t -> i + sum t

let lst_sum = sum [String "1"; Int 2]
```

## Catch-all Cases

Catch-all cases lead to buggy code. Avoid using them. Otherwise, if you add a constructor to the variant, it is possible that you forget to change sonm pattern matching code.

## Recursive Variants

Variant types may mention their own name inside their own body. For example, here is a variant type that could be used to represent something similar to `int list`:

```OCaml
type intlist = Nil | Cons of int * intlist

let lst3 = Cons (3, Nil)  (* similar to 3 :: [] or [3] *)
let lst123 = Cons(1, Cons(2, lst3)) (* similar to [1; 2; 3] *)

let rec sum (l : intlist) : int =
  match l with
  | Nil -> 0
  | Cons (h, t) -> h + sum t

let rec length : intlist -> int = function
  | Nil -> 0
  | Cons (_, t) -> 1 + length t

let empty : intlist -> bool = function
  | Nil -> true
  | Cons _ -> false
```

## Parameterized Variants

Variant types may be parameterized on other types. For example, the `intlist` type above could be generalized to provide lists (coded up ourselves) over any type:

```OCaml
type 'a mylist = Nil | Cons of 'a * 'a mylist

let lst3 = Cons (3, Nil)  (* similar to [3] *)
let lst_hi = Cons ("hi", Nil)  (* similar to ["hi"] *)
```
