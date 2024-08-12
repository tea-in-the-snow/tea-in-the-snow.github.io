---
title: "OCaml Options"
date: 2024-08-03 07:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

Suppose you want to write a function that usually returns a value of type t, but sometimes returns nothing. For example, you might want to define a function `list_max` that returns the maximum value in a list, but there’s not a sensible thing to return on an empty list:

```OCaml
let rec list_max = function
  | [] -> ???
  | h :: t -> max h (list_max t)
```

You can think of an option as being like a closed box. Maybe there’s something inside the box, or maybe box is empty. We don’t know which until we open the box. If there turns out to be something inside the box when we open it, we can take that thing out and use it.

Thus, options provide a kind of “maybe type”, which ultimately is a kind of one-of type: the box is in one of two states, full or empty.

In `list_max` above, we’d like to metaphorically return a box that’s empty if the list is empty, or a box that contains the maximum element of the list if the list is non-empty.

Here’s how we create an option that is like a box with 42 inside it:

```OCaml
Some 42
```

And here’s how we create an option that is like an empty box:

```OCaml
None
```

Like `list`, we call option a type constructor: given a type, it produces a new type; but, it is not itself a type. So for any type `t`, we can write `t option` as a type. But option all by itself cannot be used as a type.

Values of type `t` option might contain a value of type `t`, or they might contain nothing. `None` has type `'a option` because it’s unconstrained what the type is of the thing inside — as there isn’t anything inside.

Here’s a function that extracts an `int` from an option, if there is one inside, and converts it to a string:

```OCaml
let extract o =
  match o with
  | Some i -> string_of_int i
  | None -> "";;
```

Here’s how we can write `list_max` with options:

```Ocaml
let rec list_max = function
  | [] -> None
  | h :: t -> begin
      match list_max t with
        | None -> Some h
        | Some m -> Some (max h m)
      end
```
