# Go 语言常见的坑

- [Go 语言常见的坑](#go-语言常见的坑)
  - [++ 和 -- 运算符](#-和----运算符)
  - [可变参数是空接口类型](#可变参数是空接口类型)
  - [数组是值传递](#数组是值传递)
  - [Map 遍历是顺序不固定](#map-遍历是顺序不固定)
  - [返回值被局部变量屏蔽](#返回值被局部变量屏蔽)
  - [切片会导致整个底层数组被锁定](#切片会导致整个底层数组被锁定)
  - [空指针和空接口不等价](#空指针和空接口不等价)
  - [内存地址会变化](#内存地址会变化)
  - [Goroutine 泄露](#goroutine-泄露)

## ++ 和 -- 运算符

`i++` 和 `i--` 在 Go 语言中是语句，不是表达式，因此不能赋值给另外的变量。此外没有 `++i` 和 `--i`。

## 可变参数是空接口类型

当参数的可变参数是空接口类型时，传入空接口的切片时需要注意参数展开的问题。

```go
func main() {
    var a = []interface{}{1, 2, 3}

    fmt.Println(a)
    fmt.Println(a...)
}
```

```
[1 2 3]
1 2 3
```

## 数组是值传递

在函数调用参数中，数组是值传递，无法在函数修改外部数组的成员。

```go
func main() {
    x := [3]int{1, 2, 3}

    func(arr [3]int) {
        arr[0] = 7
        fmt.Println(arr)
    }(x)

    fmt.Println(x)
}
```

可以传入数组指针或者用切片。

## Map 遍历是顺序不固定

## 返回值被局部变量屏蔽

在局部作用域中，命名的返回值内同名的局部变量屏蔽：

```go
func Foo() (err error) {
    if err := Bar(); err != nil {
        return // 返回局部变量 err
    }
    return
}
```

## 切片会导致整个底层数组被锁定

切片会导致整个底层数组被锁定，底层数组无法释放内存。如果底层数组较大会对内存产生很大的压力。

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = data[:1]
    }

    // do some thing
}
```

解决的方法是将结果克隆一份，这样可以释放底层的数组：

```go
func main() {
    headerMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/path/to/file"
        data, err := ioutil.ReadFile(name)
        if err != nil {
            log.Fatal(err)
        }
        headerMap[name] = append([]byte{}, data[:1]...)
    }

    // do some thing
}
```

## 空指针和空接口不等价

## 内存地址会变化

```go
func main() {
    var x int = 42
    var p uintptr = uintptr(unsafe.Pointer(&x))

    runtime.GC()
    var px *int = (*int)(unsafe.Pointer(p))
    println(*px)
}
```

## Goroutine 泄露

Go 语言是带内存自动回收的特性，因此内存一般不会泄漏。但是 Goroutine 确存在泄漏的情况，同时泄漏的 Goroutine 引用的内存同样无法被回收。

```go
func main() {
    ch := func() <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                ch <- i
            }
        } ()
        return ch
    }()

    for v := range ch {
        fmt.Println(v)
        if v == 5 {
            break
        }
    }
}
```

上面的程序中后台 Goroutine 向管道输入自然数序列，`main` 函数中输出序列。但是当 `break` 跳出 `for` 循环的时候，后台 Goroutine 就处于无法被回收的状态了。

可以通过 `context` 包来避免这个问题：

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    ch := func(ctx context.Context) <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                select {
                case <- ctx.Done():
                    return
                case ch <- i:
                }
            }
        } ()
        return ch
    }(ctx)

    for v := range ch {
        fmt.Println(v)
        if v == 5 {
            cancel()
            break
        }
    }
}
```
