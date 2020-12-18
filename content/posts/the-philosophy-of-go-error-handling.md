---
title: "Go Error 的设计哲学"
date: 2020-12-05T17:36:56+08:00
categories: ["go"]
---


最近阅读了不少关于 Go 错误处理的材料和讨论，总结出 Go Error 的设计哲学主要有两点：

- 处理所有潜在的错误
- Errors Are Value

 Go 对错误处理的最初设计和后续优化方案，以及编程过程中处理错误的各种方法，基本都是围绕着这些理念展开的。

<!--more-->

## 1 处理所有潜在的错误

 Go 从最初设计起就确定了一个原则：程序中的所有潜在的错误都必须被明确地处理。为什么要这么做？可以先看下面一段代码：

```go
func CopyFile(src, dst string) throws error {
	r := os.Open(src)
	defer r.Close()

	w := os.Create(dst)
	io.Copy(w, r)
	w.Close()
}
```

在这段代码中， `os.Open`  `os.Create` `io.Copy`等函数都可能失败引发错误，当错误发生时会抛出错误 `throws error` ，但我们并不能直接知道具体哪个函数出错，并依此做出相应的处理（例如，当 `io.Copy` 出错时，我们需要删除了 `dst` 文件）。

这种 `throws error`  这种处理方式，和很多其它编程语言里采用的 `try catch` 类似，都属于隐晦地处理错误的方法。而 Go 希望避免这种情况，它希望清楚地处理错误（error），而不是把它当作异常（exception） 。按照这个原则，代码应该如下：

```go
func CopyFile(src, dst string) error {
	r, err := os.Open(src)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	defer r.Close()

	w, err := os.Create(dst)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if _, err := io.Copy(w, r); err != nil {
		w.Close()
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if err := w.Close(); err != il {
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
}
```

这段代码基本上对潜在的错误都做了检查和处理，使程序更加健壮。

Go 对错误处理经常会提到两句话：

Allways  check your errors!

Do not treat errors like exception

其实就是在实践这一设计理念。

不过，目前这种对错误的处理方法并不是完美的，代码中重复出现很多次 `if err != nil` ，可读性较差，这也是 Go 社区近年来试图解决的一个问题。

Go2 的错误处理讨论草案曾提出引入  `check` /  `handle`  关键字，内置函数  `try()` 等新特性来避免重复的错误检查，因为担心这些方案不能带来显著的改善，目前还没有进一步的计划。

## 2 Errors Are Values

"Errors Are Values"，是 Go  Error 设计的第二个哲学，它的主要内涵是 error 是可以被赋值，即通过代码来自定义 `error` 。

在 Go 语言里，`error` 是一个接口类型，只要实现了 `Error()` 方法，就可以自定义一个错误结构，这是 Errors Are Values 的基础。

```go
type error interface {
    Error() string
}

// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

这样的设计，使得我们在处理错误时有很大的发挥空间。

以上述备受争议 `if err != nil` 为例，虽然很多人诟病这个语句会在代码中出现太多次，可读性很差。但 Rob Pike 就认为，这在很多情况下是可以避免的，很多人只是简单地了解了 Go 处理错误的范式，没有更深入地去领会它，每次处理错误时就打上 `if err != nil` ，才会重复很多次这个语句。如果真正领会了 "Errors Are Values" 的含义，就可以通过代码来改变这一情况。

以下面代码为例：

```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

 `if err != nil` 重复出现了多次，十分累赘，但我们可以根据这个场景自定义一个错误结构：

```go
type errWriter struct {
    w   io.Writer
    err error
}
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

上述的代码就可以改成：

```go
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}
```

经过调整后的代码更加整洁，同时也能清楚地处理错误。这种处理错误的方式，在官方标准库中也有不少应用，例如 bufio 的 [Write](https://golang.org/src/bufio/bufio.go?s=14996:15065#L548) 就使用了类似方法。Go 官方博客介绍错误处理的[文章](https://blog.golang.org/error-handling-and-go) 里，“Simplifying repetitive error handling“ 一节也介绍相似的处理方法。

Rob Pike 也强调，这种对错误的处理方式并非唯一的对策，Errors Are Values 的设计，给我们带来了更多可能性。



可以预见，未来 Go 对错误的更新迭代，仍然离不开这两点设计哲学。假如 Go 的错误处理未来不能变得足够好，那也可能跟这两点设计哲学。


## 参考资料

【 [Error handling and Go](https://blog.golang.org/error-handling-and-go)  】Go 错误的基础知识和使用介绍

【[Working with Errors in Go 1.13](https://blog.golang.org/go1.13-errors) 】Go 1.13 带来的新特性 `Uwarp` ， `errors.Is` ，`errors.As` 

【[errors-are-values](https://blog.golang.org/errors-are-values)】 Rob Pike 介绍 errors are values 的应用

【[error handling problem overview](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md)】Russ Cox 总结错误处理存在的问题

【[https://blog.golang.org/experiment#Errors](https://blog.golang.org/experiment#Errors)】 Russ Cox 介绍错误处理的改进进展

【[Proposal: A built-in Go error check function, "try"](https://github.com/golang/go/issues/32437)】"try" 提案讨论 

【[Go 2 Draft Designs](https://go.googlesource.com/proposal/+/master/design/go2draft.md)】Go 2 设计草案，关于错误部分占了不少，有的已经应用到 Go 1.13