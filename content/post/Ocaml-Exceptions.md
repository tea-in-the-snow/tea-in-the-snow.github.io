---
title: "Ocaml Exceptions"
date: 2024-08-04 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

OCaml has an exception mechanism similar to many other programming languages. A new type of OCaml exception is defined with this syntax:

```OCaml
exception E of t
```

To create an exception value, use the same syntax you would for creating a variant value. Here, for example, is an exception value whose constructor is `Failure`, which carries a `string`:

```OCaml
Failure "something went wrong"
```

To catch an exception, use this syntax:

```OCaml
try e with
| p1 -> e1
| ...
| pn -> en
```

The expression `e` is what might raise an exception. If it does not, the entire try expression evaluates to whatever `e` does. If `e` does raise an exception value `v`, that value `v` is matched against the provided patterns, exactly like match expression.

## Pattern Matching

There is a pattern form for exceptions. Here’s an example of its usage:

```OCaml
match List.hd [] with
  | [] -> "empty"
  | _ :: _ -> "non-empty"
  | exception (Failure s) -> s
```

Exception patterns are a kind of syntactic sugar. Consider this code for example:

```OCaml
match e with
  | p1 -> e1
  | exception p2 -> e2
  | p3 -> e3
  | exception p4 -> e4
```

We can rewrite the code to eliminate the wxception pattern:

```OCaml
try 
  match e with
    | p1 -> e1
    | p3 -> e3
with
  | p2 -> e2
  | p4 -> e4
```

In general if there are both exception and non-exception patterns, evaluation proceeds as follows:

- Try evaluating `e`. If it produces an exception packet, use the exception patterns from the original match expression to handle that packet.
- If it doesn’t produce an exception packet but instead produces a non-exception value, use the non-exception patterns from the original match expression to match that value.
