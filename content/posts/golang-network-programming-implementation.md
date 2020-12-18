---
title: Go 网络编程的实现
date: 2020-07-22 00:59:17
categories: ["go"]
---

提要：

- TCP 服务端、客户端示例
- Network system calls
- TCP 编程是如何实现的

Go 网络编程主要通过 net 包（package）实现，它支持 TCP/IP, UDP, domain name resolution, Unix domain sockets 等连接，此外，还通过 net/http ，net/rpc 等提供了 http，rpc 等主流应用层的连接协议。

<!--more-->

# 1 TCP 服务示例

TCP 是最常用的网络连接方式，以 TCP 连接为例，一个简单的 TCP 连接代码示例：

TCP Client

```go
conn, err := net.Dial("tcp", "golang.org:80")
if err != nil {
	// handle error
}
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
status, err := bufio.NewReader(conn).ReadString('\n')
// ...D
```

TCP server

```go
ln, err := net.Listen("tcp", ":8080")
if err != nil {
	// handle error
}
for {
	conn, err := ln.Accept()
	if err != nil {
		// handle error
	}
	go handleConnection(conn)
}
```

那么 net 包背后是如何实现 tcp 连接的？

# 2. TCP 连接在系统调用层面的实现

包括 TCP/IP 在内的各种网络连接，在类 unix 的操作系统里，都是通过网络系统调用（network system calls） 实现的，相关系统调用接口主要有：

`socket()`，`bind()`， `listen()`， `accept()`，`send()`  

使用系统调用创建 TCP 服务器的核心流程是：

- 创建 socket 连接 `sockfd = socket(AF_INET, SOCK_STREAM, 0);`
- 绑定 地址   `bind(sockfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr));`
- 启动监听 `listen(sockfd,5);`
- 接收客户端的连接 `newsockfd = accept(sockfd, (struct sockaddr *)&cli_addr, &clilen);`
- 接收数据 `n = read( newsockfd,buffer,255 );`
- 发送数据，`n = write(newsockfd,"I got your message",18);`

创建 TCP 客户端的核心流程：

- 创建 socket 连接 `sockfd = socket(AF_INET, SOCK_STREAM, 0);`
- 连接服务器地址：`connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr))`
- 接收数据 `n = read( newsockfd,buffer,255 );`
- 发送数据，`n = write(newsockfd,"I got your message",18);`

 `socket()` 等系统调用的实现，是 kernel 层面完成的。

对于 net 包而言，包括 TCP 在内网络连接，底层上是通过这些系统调用实现的，它的任务则是对这个工作流程做了封装。

# 3. Golang TCP 连接的实现

- go 在调用 syscall 之前，做了什么工作
- 这些工作有什么必要性？

## 3.1  核心方法

以客户端连接为例，`conn, err := net.Dial("tcp", "golang.org:80")`  的核心实现方法是 dial.go 的

```go
func (d *Dialer) DialContext(ctx context.Context, network, address string) (Conn, error)
```

这个方法的主要功能是：

### 3.1.1 context 处理

### 3.1.2 网络和地址解析， `d.resolver().resolveAddrList(.....)`

- 解析 network 类型；unix socket, tcp, udp 等，通过 `parseNetwork(...)`
- 解析 address ，IP 或者 hostname ，`r.internetAddrList(ctx, afnet, addr)`
    - `r.lookupIPAddr(ctx, net, host)`，先解析 IP，或者 DNS 解析 ，`r.lookupIPAddr(...)`

### 3.1.3 sysDialer ，拨号连接

连接的两种方式：

- `sd.dialParallel(ctx, primaries, fallbacks)`
- `sd.dialSerial(ctx, primaries)`

`sd.dialSingle(dialCtx, ra)` 支持四种类型：

```go
sd.dialTCP(ctx, la, ra)

sd.dialUDP(ctx, la, ra)

sd.dialIP(ctx, la, ra)

sd.dialUnix(ctx, la, ra)
```

以 TCP 为例：

`sd.dialTCP(ctx, la, ra)`  以及  `sl.listenTCP(ctx, la)`  的下层实现都可以追溯到 sock_posix.go 的

```go
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
```

 即实现系统调用`socket()` 。

如果是服务端，调用 `fd.listenStream(laddr, listenerBacklog(), ctrlFn)`

- 实现 `bind()` ： `syscall.Bind(fd.pfd.Sysfd, lsa)`
- 实现 `listen()` : `listenFunc(fd.pfd.Sysfd, backlog)`

如果是客户端，调用 `fd.dial(ctx, laddr, raddr, ctrlFn)`

- 实现 `connect()` : `fd.connect(ctx, lsa, rsa)`

```c
if laddr != nil {
		if lsa, err = laddr.sockaddr(fd.family); err != nil {
			return err
		} else if lsa != nil {
			if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
				return os.NewSyscallError("bind", err)
			}
		}
	}
	var rsa syscall.Sockaddr  // remote address from the user
	var crsa syscall.Sockaddr // remote address we actually connected to
	if raddr != nil {
		if rsa, err = raddr.sockaddr(fd.family); err != nil {
			return err
		}
		if crsa, err = fd.connect(ctx, lsa, rsa); err != nil {
			return err
		}
		fd.isConnected = true
	} else {
		if err := fd.init(); err != nil {
			return err
		}
	}
```

## 3.2 小结

net 在调用syscall 之前，主要是增加 context 控制，封装 udp，tcp, uninx domain socket  等不同的连接类型，DNS 查找等，同时在有需要的地方引入 goroutine 提高处理效率。

*问题：*

*proxy 是如何实现的*

*TCP 如何保持长连接*

```c
 *int setsockopt(int sockfd, int level, int optname,const void *optval, socklen_t optlen);*
```

# 4 TCP 数据发送与接收

## 4.1 服务端和客户端创建连接

TCP 连接建立后，如果是服务端，每当有客户端发来请求时，会建立新的连接。

```go
conn, err := ln.Accept()
```

这个方法的下层实现是，tcpsock.go 

```go
func (l *TCPListener) Accept() (Conn, error)
```

最终可以追溯到 fd_unix.go

```go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) 
```

而最底层的实现则是系统调用：hook_unix.go， `syscall.Accept`

## 4.2 读写数据

在 unix 的设计哲学里，everything is a file，延伸到网络连接设计，发送和接收数据对应也就是写入数据和读取数据。

网络连接建立后的  `Conn`  类型，其实也是一个文件描述符（file discreptor），故发送数据就是向 conn 写入数据

 

```go
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
```

接收数据即是从 `conn` 读取数据

```go
status, err := bufio.NewReader(conn).ReadString('\n')
```

# 5 Http 的实现

go http 连接就是先实现 http 协议，然后再调用上述的 TCP 连接。

```go
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

参考：

[https://www.tutorialspoint.com/unix_sockets/socket_server_example.htm](https://www.tutorialspoint.com/unix_sockets/socket_server_example.htm)

[TCP Keepalive HOWTO](https://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)

[http://www.haifux.org/lectures/217/netLec5.pdf](http://www.haifux.org/lectures/217/netLec5.pdf)

[http://linasm.sourceforge.net/docs/syscalls/network.php](http://linasm.sourceforge.net/docs/syscalls/network.php)

[https://www.kernel.org/doc/html/latest/networking/rds.html#socket-interface](https://www.kernel.org/doc/html/latest/networking/rds.html#socket-interface)