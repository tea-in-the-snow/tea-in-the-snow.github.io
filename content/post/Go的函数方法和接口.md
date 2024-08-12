---
title: "Go的函数，方法和接口"
date: 2024-01-30 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Go
---

## 函数 (function)

- 函数可以没有参数或者接受多个参数。

- 当连续两个或多个函数的已命名形参类型相同时，除最后一个类型以外，其它都可以省略。

```go
func add(x, y int) int {
    return x + y
}
```

- 函数（或者变量）的名称以大写字母开头时，它就是已导出的。

- 函数可以返回任意数量的字符串。

```go
func swap(x, y string) (string, string) {
    return y, x
}
```

- 函数的返回值可以被命名，它们会被视作在函数顶部定义的变量，没有参数的 return 返回已经被命名的返回值。

```go
func division(dividend, divisor int) (quotient, remainder int) {
    quotient = dividend / divisior
    remainder = dividend - quotient * divisor
    return
}
```

- 函数也是值，也可以用作函数的参数和返回值。

```go
// conpute 接受一个函数作为参数
// 调用 conpute 时传入不同的函数，返回对3和4作不同的操作的结果
func conpute(fn func(float64, float64) float64) float64 {
    return fn(3, 4)
}
```

## 函数的闭包 (closure)

- A closure is a record storing a function together with an environment.

- 闭包是由函数和环境组合而成的。闭包保存和记录了它产生时的外部环境——它的函数体之外的变量，并且可以访问和修改这些变量。

- 在闭包实际实现的时候，往往通过调用一个外部函数返回其内部函数来实现的。用户得到一个闭包，也等同于得到了这个内部函数，每次执行这个闭包就等同于执行内部函数。

- 如果外部函数的变量可见性是 local 的，即生命周期在外部函数结束时也结束的，那么闭包的环境就是封闭的。反之，那么闭包其实不再封闭，全局可见的变量的修改，也会对闭包内的这个变量造成影响。

```go
package main

import "fmt"

func test_1(x int) func() {
    return func() {
        x++
        fmt.Println(x)
    }
}

func test_2(x int) func() {
    sum := 0
    return func() {
        sum += x
        fmt.Println(x, sum)
    }
}

func test_3(x int) func(int) int {
    sum := 0
    return func(y int) int {
        sum += x * y
        return sum
    }
}

func main() {
    test_1(1)()

    test_2(1)()

    // 每个闭包事实上有着不同的外部环境
    // 即：对每个 for 循环，都会新建一个 test_3()
    // 所以每个闭包绑定的是不同的 sum 变量
    for i := 0; i < 5; i++ {
        fmt.Printf("%d ", test_3(1)(i))
    }
    fmt.Println()

    // 每个闭包的外部环境相同（tmp）
    // 即 for 循环中的闭包绑定的是同一个 sum 变量
    tmp := test_3(1)
    for i := 0; i < 5; i++ {
        fmt.Printf("%d ", tmp(i))
    }
    fmt.Println()
}
```

上面的程序输出结果是：

```bash
2
1 1
1 1
0 1 2 3 4 
0 1 3 6 10
```

## 方法 (method)

- Go 没有类，不过可以为结构体类型定义方法。方法就是一类带特殊的接收者参数的函数。方法接收者在它自己的参数列表内，位于 func 关键字和方法名之间。（非结构体类型也可以定义方法）

```go
type Vertex struct {
    X, Y float64
}

func (v Vertex) distance() float64 {
    return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

- 方法并不能修改指针接收者的值。只有指针接收者的方法能够修改接收者指向的值。（在这种情况下，方法也没有修改接收者的值——指针的内容，只是修改了指针指向的值，和用指针作为参数是一样的）

- 在很多意义上，方法的接收者和普通的参数是一样的。如果不使用指针。

- 不过，带指针参数的函数必须接受一个指针，而以指针为接受者的方法被调用时，接受者接收者既能为值又能为指针。

```go
package main

import "fmt"

type Vertex struct {
    X, Y float64
}

func (v *Vertex) Move_1(dx, dy float64) {
    v.X += dx
    v.Y += dy
}

func (v Vertex) Move_2(dx, dy float64) {
    v.X += dx
    v.Y += dy
}

func Move_3(v *Vertex, dx, dy float64) {
    v.X += dx
    v.Y += dy
}

func Move_4(v Vertex, dx, dy float64) {
    v.X += dx
    v.Y += dy
}

func main() {
    var v Vertex
    v.X = 0
    v.Y = 0.
    v.Move_1(1, 1)
    fmt.Println(v.X, v.Y)
    p := &v
    p.Move_1(1, 1)
    fmt.Println(v.X, v.Y)
    v.Move_2(1, 1)
    fmt.Println(v.X, v.Y)
    Move_3(&v, 1, 1)
    fmt.Println(v.X, v.Y)
    Move_4(v, 1, 1)
    fmt.Println(v.X, v.Y)
}
```

上面的程序输出结果是：

```bash
1 1
2 2
2 2
3 3
3 3
```

## 接口 (interface)

- 接口是一组方法签名的集合，接口类型的变量可以保存任何实现了这些方法的值。

- Go 语言中的接口是隐式实现的，也就是说，如果一个类型实现了一个接口定义的所有方法，那么它就自动地实现了该接口。没有 implements 关键字。

```go
type MyFloat float64

func (f MyFloat) Abs() float64 {
    if f < 0 {
        return float64(-f)
    }
    return float64(f)
}

type Vertex struct {
    X, Y float64
}

func (v *Vertex) Abs() float64 {
    return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

type Abser interface {
    Abs() float64
}

func main() {
    var a Abser
    f := MyFloat(-math.Sqrt2)
    v := Vertex{3, 4}

    a = f  // MyFloat 实现了 Abs()
    a = &v // *Vertex 实现了 Abs()
}
```

- 指定了零个方法的接口值被称为空接口，可以保存任何类型的值（因为每个类型都至少实现了零个方法）。空接口被用来处理未知类型的值。

- 在内部，接口值可以看做包含值和具体类型的元组，类型断言提供了访问接口值底层具体值的方式。

```go
package main

import "fmt"

func main() {
    var i interface{} = "hello"

    // 该语句断言接口值 i 保存了具体类型 string，
    // 并将其底层类型为 string 的值赋予变量 s。
    // 若 i 并未保存 string 类型的值，该语句就会触发 panic。
    s := i.(string)
    fmt.Println(s)

    // 为了判断一个接口值是否保存了一个特定的类型，
    // 类型断言可返回两个值：其底层值以及一个报告断言是否成功的布尔值。
    // 若 i 保存了一个 string，那么 s 将会是其底层值，而 ok 为 true。
    // 否则，ok 将为 false 而 s 将为 T 类型的零值，程序并不会产生 panic。
    s, ok := i.(string)
    fmt.Println(s, ok)

    f, ok := i.(float64)
    fmt.Println(f, ok)

    f = i.(float64) // 报错 (panic)
    fmt.Println(f)
}
```

上面的程序输出结果是：

```bash
hello
hello true
0 false
panic: interface conversion: interface {} is string, not float64
......
```