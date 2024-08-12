---
title: "Ocaml Unit Testing"
date: 2024-08-05 06:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - OCaml
---

## Basic workflow for testing

- Write a function in a file f.ml.
- Write unit tests for that function in a separate file test.ml.
- Build and run test to execute the unit tests.

## Example

`sum.ml`

```OCaml
let rec sum = function
  | [] -> 0
  | x :: xs -> x + sum xs
```

`test.ml`

```OCaml
open OUnit2
open Sum

let tests = "test suite for sum" >::: [
    (*assert_equal checks whether its two arguments are equal*)
  "empty" >:: (fun _ -> assert_equal 0 (sum []));
  "singleton" >:: (fun _ -> assert_equal 1 (sum [1]));
  "two_elements" >:: (fun _ -> assert_equal 3 (sum [1; 2]));
]

let _ = run_test_tt_main tests
```

To improve output, we have

```OCaml
let tests = "test suite for sum" >::: [
  "empty" >:: (fun _ -> assert_equal 0 (sum []) ~printer:string_of_int);
  "singleton" >:: (fun _ -> assert_equal 1 (sum [1]) ~printer:string_of_int);
  "two_elements" >:: (fun _ -> assert_equal 3 (sum [1; 2]) ~printer:string_of_int);
]
```

To improve the code, we have

```OCaml
let make_sum_test name expected_output input =
  name >:: (fun _ -> assert_equal expected_output (sum input) ~printer:string_of_int)

let tests = "test suite for sum" >::: [
  make_sum_test "empty" 0 [];
  make_sum_test "singleton" 1 [1];
  make_sum_test "two_elements" 3 [1; 2];
]
```
