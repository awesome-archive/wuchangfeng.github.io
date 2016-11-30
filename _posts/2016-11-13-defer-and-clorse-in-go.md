---
layout: post
title: Go: Go 中的 Defer 和 Closure
date: 2016-11-13 10:28:01 +0800
categories: Go

---

学习 Go 中的 defer 相关知识疑惑之处以及闭包的概念和用法。

<!-- more -->

### Defer 的理解

按照官方的解释，defer 后面的表达式会被放入一个列表中，在当前方法返回的时候，列表中的表达式就会被执行。一个方法中可以在一个或者多个地方使用defer表达式。文档中一个例子如下：

```go
func a() {
    i := 0
    defer fmt.Println(i)
// defer fmt.Println(2)
// defer fmt.Println(3)
    i++
    return
}
```

打印出来的数值为 0 ，这就很诧异了，即使按照 defer 最后被执行，那 i 的数值也应该是 1 啊。仔细看了官方文档原来原因：上面的这段代码，defer 表达式中用到了i这个变量，i 在初始化之后的值为 0，接着程序执行到 defer 表达式这一行，表达式所用到的i的值就为 0 了，接着，表达式被放入 list，等待在 return 的时候被调用。所以，后面尽管有一个 i++ 语句，仍然不能改变表达式 fmt.Println(i) 的结果。接着我们在上述代码中加上注释的代码，最后返回的结果分别是 3，2，0。这就验证了一个：**defer 表达式的调用顺序是按照先进后出的方式**。

在 defer 表达式中可以修改**函数中的命名返回值**，如下代码所示：

```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}
```

返回值变量名为 i，在 defer 表达式中可以修改这个变量的值。所以，虽然在 return 的时候给返回值赋值为 1，后来 defer 修改了这个值，让 i 自增了 1，所以，函数的返回值是 2 而不是 1。**这个可以解释下面代码中的疑问一。**

### 匿名函数

一旦我们暂时还不想给一个函数取名字，可以用如下方式定义函数：

```go
func(x, y int) int { return x + y }
```

相应的调用方式如下，**注意最后一对小括号，就是表示匿名函数的调用**：

```go
func(x, y int) int { return x + y } (3, 4)
```

下面这个例子就是展示了如何将**匿名函数赋值给变量**并对其进行调用：

```go
package main

import "fmt"

func main() {
    f()
}
func f() {
    for i := 0; i < 4; i++ {
      	// 匿名函数赋值给局部变量 g
        g := func(i int) { fmt.Printf("%d ", i) } 
        // 利用变量调用匿名函数
      	g(i)
        fmt.Printf(" - g is of type %T and has value %v\n", g, g)
    }
}
```

书上更进一步给我们一个实例如下,匿名函数跟 defer 结合的：

```go
package main

import "fmt"
// 疑问一
func f() (ret int) {
    defer func() {
        ret++
    }()
    return 1
}
func main() {
    fmt.Println(f())
}
```

最终返回的是 2，因为 ret ++ 是在执行 return 1 语句之后发生的。其实个人有点疑惑，return 返回的数字，就是给 ret 作为初始变量了吗？或者说 return 返回的语句立马就进入到 defer 定义的函数中了？

### 闭包与函数

函数只是一段可执行代码，编译后就“固化”了，每个函数在内存中只有一份实例，得到函数的入口点便可以执行函数了。在函数式编程语言中，函数是一等公民（First class value）：第一类对象，我们不需要像命令式语言中那样借助函数指针，委托操作函数，函数可以作为另一个函数的参数或返回值，可以赋给一个变量。函数可以嵌套定义，即在一个函数内部可以定义另一个函数，有了嵌套函数这种结构，便会产生闭包问题。而此时需要注意，闭包并不是函数。由于闭包存在上下文的问题，也就是引用环境，不同的引用环境与同一个闭包能够成不同的实例。在 OO 中，我们习惯将**对象**传来传去，而在函数式编程中，我们就需要转换思想了，我们传来传去的是 **函数**。

### 应用闭包：将函数作为返回值

如下两个函数 Add2 和 Adder 都会返回签名为 func(b int) int 的函数：

```go
func Add2() (func(b int) int)
func Adder(a int) (func(b int) int)
```

对应的实例如下所示：

```go
package main

import "fmt"

func main() {
    // make an Add2 function, give it a name p2, and call it:
    p2 := Add2()
    fmt.Printf("Call Add2 for 3 gives: %v\n", p2(3))
    // make a special Adder function, a gets value 3:
    TwoAdder := Adder(2)
    fmt.Printf("The result is: %v\n", TwoAdder(3))
}
// 将函数作为返回值
func Add2() func(b int) int {
  	// 注意在这里的结构
    return func(b int) int {
        return b + 2
    }
}

func Adder(a int) func(b int) int {
    return func(b int) int {
        return a + b
    }
}
```

返回的数值如下：

```go
 Call Add2 for 3 gives: 5
  The result is: 5
```

接着书上讲了一个**比较重要的例子**，通过这个例子跟我们介绍一个概念：闭包函数保存并积累其中的变量的值，不管外部函数退出与否，它都能够继续操作外部函数中的局部变量。

实例如下：

```go
package main

import "fmt"

func main() {
    var f = Adder()
    fmt.Print(f(1), " - ")
    fmt.Print(f(20), " - ")
    fmt.Print(f(300))
}
// Adder() 函数中，函数作为返回值
func Adder() func(int) int {
    var x int
  	// 闭包函数，对于连续的 return 用简单的思维去理解
    return func(delta int) int {
        x += delta
        return x
    }
}
```

按照一般的思维，肯定会觉得返回值：1-20-300。可以程序执行出来的结果却为：1-21-321。这？我们按照思路理一下：三次调用函数 f 的过程中函数 Adder() 中变量 delta 的值分别为：1、20 和 300。对比输出的结果，我们发现每一次 x 这个**外部函数的局部变量并没有在闭包中被清零！！！**，也就是说变量 x 的数值在**闭包**中是被保留的。无论外部函数的状态如何！！！

另外注意一点，闭包中使用到的变量可以在**闭包函数体内**也可以在**闭包函数体外**声明。

### 计算函数执行时间

这一章中的最后一小节，讲述了如何计算函数的执行时间,主要就是 time 包中的 Now() 和 Sub() 函数，对于函数优化，我们就可以通过函数的计算时间来进行比较：

```go
start := time.Now()
longCalculation()
end := time.Now()
delta := end.Sub(start)
fmt.Printf("longCalculation took this amount of time: %s\n", delta)
```

