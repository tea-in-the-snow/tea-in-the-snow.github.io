---
title: "Ocaml Records and Tuples"
date: 2024-08-02 01:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

## Records

A record is a composite of other types of data, each of which is named. OCaml records are much like structs in C.

```OCaml
type ptype = TNormal | TFire | TWater
type mon = {name : string; hp : int; ptype : ptype}
```

To build a value of a record type, we write a record expression, which looks like this:

```OCaml
{name = "Charmander"; hp = 39; ptype = TFire}
```

To access a record and get a field from it, we use the dot notation that you would expect from many other languages. For example:

```OCaml
let c = {name = "Charmander"; hp = 39; ptype = TFire};;
c.hp
```

It’s also possible to use pattern matching to access record fields:

```OCaml
match c with {name = n; hp = h; ptype = t} -> h
```

The `n`, `h`, and `t` here are pattern variables. There is a syntactic sugar provided if you want to use the same name for both the field and a pattern variable:

```OCaml
match c with {name; hp; ptype} -> hp
```

## Tuples

Like records, tuples are a composite of other types of data. But instead of naming the components, they are identified by position. Here are some examples of tuples:

```OCaml
(1, 2, 10)
(true, "Hello")
([1; 2; 3], (0.5, 'X'))
```

A tuple with two components is called a pair. A tuple with three components is called a triple. Beyond that, we usually just use the word tuple instead of continuing a naming scheme based on numbers.
