
---
title: go channel，buffered 与 unbuffered 的区别
date: 2019-08-19 01:23:29
categories: ["go"]
tags: 
---

go 通过 goroutine 实现并发，而不同的 goroutine 之间需要通过 channel 来通讯，否则就是两段各行其是的代码。
<!--more-->


## channel 解决 goroutine 的通讯

看一个 [code](https://play.golang.org/p/5cvtWy36o8b)

```go
func main() {
    
    go func(){
      fmt.Println("I'm gotoutine") //不会执行
    }()
    fmt.Println("bye")
}
//bye
```

这里的 goroutine 最终不会打印信息，原因是 main （也是一个 goroutine）不会去管另一个 goroutine 的情况，只顾把自己部分执行完。main 结束后，整个程序也就结束了。

解决这个问题的办法，就是利用 channel 来传递消息，告诉 main 另一个 goroutine 什么时候结束。

 [code](https://play.golang.org/p/g9u4Z46tgz2)

```go
func main() {
    c := make(chan int)
    go func(){
      fmt.Println("I'm gotoutine")
      c <- 1
    }()
    fmt.Println("bye", <-c)
    fmt.Println("main")
}
//I'm gotoutine
//bye 1
//main
```

## buffered VS unbuffered

channel 在初始化时可以设置为 unbuffered 或 buffered ：

```go
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan int, 100)       // buffered channel of integers

```

channel 的阻塞功能（block）和它是否为 buffered 有很大关系。

阻塞是指暂停执行下一段代码，等待得到新的通知后再继续执行。它会在不同的情况出现：

- 接收端始终会引发阻塞

如上一段代码，`<-c` 是 channel 的接收端，如果接收端没有执行，`main` 这个 goroutine 就不会继续执行 `fmt.Println("main")`

- 发出端引发阻塞的时机与 channel 是否为 buffered 有关









**如果 channel 是 unbuffered  ，它的发出端会引发阻塞**
，如下面代码（[code](https://play.golang.org/p/-VeBs3jcmN7])）的  `c <- 1` ：

```go
func main() {
    c := make(chan int)    
    c <- 1   // 报错 deadlock, 下面的代码不会执行
    fmt.Println(<-c)
}

//fatal error: all goroutines are asleep - deadlock!
```


**如果 channel 是 buffered ，只有当 buffer 数组满了的时候，发出端才会引发阻塞**， 参考以下例子：

[code](https://play.golang.org/p/Ikok29aKdrz)

```go
func main() {
	c := make(chan int, 1)
	c <- 1
	fmt.Println(<-c)
	c <- 2
	c <- 3 // 这一行发生阻塞，因 buffer 已满，但没有释放
	fmt.Println(<-c)

}

//1
//fatal error: all goroutines are asleep - deadlock!
```

因为这个阻塞特效，buffer 的数量也可以用来控制程序的吞吐量。





*注：channel 是为不同 goroutine 之间的通讯而生的，通常情况下，他们的接收端和发出端一般都在不同的 goroutine 里，而不会在同一个 goroutine 里同时出现，像上面大多数的例子那样，仅仅是为了演示。*


**参考**：

- [Effective Go - channels](https://golang.org/doc/effective_go.html#channels)
- [why using unbuffered... stackoverflow](https://stackoverflow.com/questions/18660533/why-using-unbuffered-channel-in-the-same-goroutine-gives-a-deadlock)







