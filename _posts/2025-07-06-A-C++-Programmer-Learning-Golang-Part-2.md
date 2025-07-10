---
title: C++程序员学Golang Part.2
description: >-
  
author: Dante.Cloud
date: 2025-07-06 00:00:00 +0800
tags: [Golang, C++]
---
这部分记录些稍微复杂些的内容。

## routine
routine是go的特色，使用起来简单优雅：
```go
func fn() { 
    //do something
}

go fn() // go + function
```
routine的含义自然是协程，协程一般的定义是包含专属寄存器和栈的执行流，多个routine可以分享单个**thread**的时间片，大家轮流执行。   
但**golang routine**并不会局限在一个**thread**之内，即一段代码中多次执行go，runtime可能会根据调度策略生成多个**thread**，此逻辑对用户透明。go不提供操作**thread**的库，用**golang routine**屏蔽并发细节。

## channel
channel是routine间同步的工具，用于传输特定类型的数据
```go
ch := make(chan int)        // 创建无缓冲channel
```
用箭头表示数据传输的方向
```go
ch <- 42      // 发送数据到channel
x := <-ch     // 从channel接收数据，并赋值给x
```
默认生成的channel是无缓冲的，即没有缓冲区用于保存需要传输的数据。使用这样的channel，接收routine如果不接收数据，发送routine会阻塞；反之，发送routine如果不发送数据，接收routine会阻塞。
可以定义有缓冲区的channel：
```go
ch := make(chan int, 3)     // 创建容量为3的缓冲channel
```
这样发送routine在缓冲区有空间是会是异步发送，接收routine在缓冲区有数据时不会阻塞。   
channel在内部实现会使用锁，无需我们在外部显式使用锁。

## select
select是golang的多路复用工具，类似于linux中针对fd的epoll/select，select针对的是channel：
```go
select {
    case <-ch1:
        // 当ch1可读时执行
    case ch2 <- data:
        // 当ch2可写时执行
    case val := <-ch3:
        // 当ch3可读时执行，并获取值
    case <-time.After(1 * time.Second):
        // 超时处理
    default:
        // 当所有case都不满足时执行（可选）
}
```
特点：
- 阻塞：若无 default 且所有 case 都阻塞，select 会阻塞直到至少一个 case 就绪。
- 随机选择：若多个 case 同时就绪，会随机选择一个执行。
- 非阻塞：若有 default 且所有 case 都阻塞，立即执行 default。




## 锁与同步
sync包中包含了routine之间同步的工具。   
Mutex是最常用的，悲观锁，用于routine之间的阻塞同步。   
WaitGroup 用于等待一组 goroutine 完成，无需显式传递 channel。   
三个方法：

- Add(delta int)：增加等待计数。
- Done()：减少等待计数（等价于 Add(-1)）。
- Wait()：阻塞直到计数为 0。

```go
import (
    "fmt"
    "sync"
)

var (
    counter int
    mutex   sync.Mutex
)

func increment() {
    mutex.Lock()
    defer mutex.Unlock()
    counter++ // 临界区
)

func main() {
    var wg sync.WaitGroup
    wg.Add(1000)

    for i := 0; i < 1000; i++ {
        go func() {
            defer wg.Done()
            increment()
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter) // 输出: 1000
}
```
Cond类似于c++中的condition_variable，用于通知唤醒机制。
```go
func producer(cond *sync.Cond, ch chan int) {
    for i := 0; i < 5; i++ {
        cond.L.Lock()
        ch <- i
        cond.Signal() // 通知消费者
        cond.L.Unlock()
        time.Sleep(200 * time.Millisecond)
    }
}

func consumer(id int, cond *sync.Cond, ch chan int) {
    cond.L.Lock()
    defer cond.L.Unlock()

    for len(ch) == 0 {
        cond.Wait() // 等待生产者通知
    }

    fmt.Printf("Consumer %d got: %d\n", id, <-ch)
}
```
sync包中还有atomic、RWMutex等工具，可按需使用。


## interface
golang的interface有两种使用场景，一种是包含成员函数的非空接口，这类似于c++的虚基类。可另外定义struct并实现接口的方法，这样就可以把该struct的指针赋值给interface，由interface来调用struct的方法，以实现运行时多态，这跟c++虚函数的使用方法几乎一致。
```go
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    Width, Height float64
}

// 实现Shape接口的方法
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

func main() {
    rect := Rectangle{Width: 3, Height: 4}
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", rect.Area(), rect.Perimeter()) // 由接口调用函数
}
```
另一种使用场景是不含成员函数的空接口interface{}，可以被赋值为任意类型，这种功能可以在很多场景灵活应用，可以用类型断言来确定空接口的具体类型：
```go
func PrintAnything(v interface{}) {
    switch t := v.(type) { //断言具体类型
    case string:
        fmt.Println("String:", t)
    case int:
        fmt.Println("Int:", t)
    default:
        fmt.Println("Unknown type")
    }
}

func main() {
    PrintAnything(42)       //可传入任意类型
    PrintAnything("hello")  
}
```

## Context
Context在C++中没有对应的概念，主要是用来协调树形routine群中routine之间行为，例如控制子routine取消，传递信息。   
context.Context 是一个接口，定义了四个核心方法：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 返回截止时间
    Done() <-chan struct{}                   // 返回一个channel，当context被取消或超时关闭
    Err() error                              // 返回context被取消的原因
    Value(key any) any                       // 返回与key关联的值
}
```
特点：
- 不可变：创建后不能修改，只能通过 With* 函数派生新的 context。
- 层级结构：通过 With* 函数创建的 context 形成一棵树，父 context 的取消会传播到所有子 context。
有几个具体的类型用于各种使用场景

### 根Context
因为Context是树形结构，最上层的routine会初始化根部的Context：
```go
ctx := context.Background()
```

### WithCancel
WithCancel函数传入一个上级的Context，会返回一个子Context外加一个Cancel函数。这个子Context可以传递给子routine，父routine调用Cancel，会触发子Context的Done：
```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():  //父routine调用Cancel的结果
            fmt.Println("Worker cancelled:", ctx.Err())
            return
        default:
            fmt.Println("Working...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go worker(ctx)
    
    // 3秒后取消
    time.Sleep(3 * time.Second)
    cancel()    //触发子routine的Done
    
    // 等待所有goroutine完成
    time.Sleep(1 * time.Second)
}
```

### WithTimeout
WithTimeout相比于WithCancel，会多一个让子routine取消的方式，即超时。超过了预设的时间，会触发子Context的Done。
```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel() // 确保资源释放

select {
case <-ctx.Done():
    fmt.Println("Timeout:", ctx.Err()) // 输出: context deadline exceeded
case <-time.After(3 * time.Second):
    fmt.Println("Operation completed")
}
```
### WithDeadline
与WithTimeout类似，不过超时时间的设置方式不是时长而是特定时间点。
```go
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
```

### WithValue
用于传递信息，不用于控制子routine的结束。
```go
ctx := context.WithValue(context.Background(), "userID", 123)
userID := ctx.Value("userID").(int) // 获取值
```