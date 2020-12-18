---
title: gRPC 入门，如何用 go 开发 grpc 服务端/客户端应用
date: 2019-08-20 14:37:37
tags: 
categories: ["go"]
---

demo 的代码仓库：[github.com/chasecs/grpc-sorter](https://github.com/chasecs/grpc-sorter)

## 一、基础知识

### rpc

RPC 是一种用于主要用于服务器通讯的 API 协议，于 2005 年左右发布，在它出现之前，常用的服务器 API 协议是 REST 和 SOAP 。
<!--more-->

gRPC 是一个 RPC 框架，同类的还有 HTTP RPC，JSON RPC，TCP RPC，关于其它 RPC 协议的介绍见 [RPC](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/08.4.md)。

### protocal buffers

gRPC 最大的特点是使用 protocal buffers ，简称 protobuf。 protobuf 是 google 开源的一种交换数据机制，有些类似 XML 、JSON，但多了定义远程方法等内容。

gRPC 开发的第一项工作就是先写 .proto 文件，然后用 protoc 编译器生成相应的代码，供 gPRC 的服务端和客户端调用。**protobuf 跟语言无关，它可以生成不同语言的代码，供不同语言的项目使用。**

因为 protocbuf 是 gRPC 开发的必备知识，所以事先有必要了解它的基本语法，入门教程见官网：[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)

## 二、用 go 开发一个 gRPC demo

这个 demo 将开发一个提供快速排序接口的服务端，以及一个发出排序请求的客户端，排序算法在服务端实现，两者之间通过 gPRC 通讯。

### 1. 配置开发环境

安装 gRPC-go，[国内方法](https://grpc.io/docs/quickstart/go/)

安装 protocol buffer [方法](https://medium.com/@erika_dike/installing-the-protobuf-compiler-on-a-mac-a0d397af46b8)

项目目录

```
grpc-sorter
|
├── protobuf // 存放 protobuf 生成的 go 代码
|   └── sorter
├── sorter_server // 服务端项目目录
├── sorter_client // 客户端项目目录
└── sorter.proto 
```
### 2. 编写 `sorter.proto` 文件

`sorter.proto` 如下：

```
syntax = "proto3";
package sorter;

message Numbers {
  repeated int64  numbers = 1;
}

service Sorter {
  rpc QuickSort(Numbers) returns (Numbers) {}
}

```

使用 protoc 命令生成 go 代码到 `protobuf/sorter` 目录：

```sh
protoc --go_out=plugins=grpc:protobuf/sorter sorter.proto
```

### 3. 实现服务端代码

`sorter_server/server.go` 主要代码如下

```go
const (
	port = ":50051"
)

type server struct{}

func (s *server) QuickSort(ctx context.Context, in *pb.Numbers) (*pb.Numbers, error) {
	qsort.QuickSort(in.Numbers)
	return in, nil
}
func main() {
	lis, _ := net.Listen("tcp", port)
	s := grpc.NewServer()
	pb.RegisterSorterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

完整的服务端代码见：[github.com/chasecs/grpc-sorter/sorter_server](https://github.com/chasecs/grpc-sorter/tree/master/sorter_server)

### 4. 客户端开发


`sorter_client/main.go` 主要代码:

```go
import (
	"log"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	pb "grpc-sorter/protobuf/sorter"
)
func main() {
	var conn *grpc.ClientConn
	conn, err := grpc.Dial(":50051", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %s", err)
	}
	defer conn.Close()
	c := pb.NewSorterClient(conn)

	var raw = []int64{1, 8, 3, 5, 6}
	response, err := c.QuickSort(context.Background(), &pb.Numbers{Numbers: raw})
	if err != nil {
		log.Fatalf("Error when calling QuickSort: %s", err)
	}
	log.Printf("Response from server: %v", response.Numbers)
}
```


至此，这组 grpc 即开发完成。

### 5. 运行

先运行服务端：

```sh
go run sorter_server/server.go
```

再运行客户端，输出排序结果:

```
go run sorter_client/main.go 
...Response from server: [1 3 5 6 8]

```

