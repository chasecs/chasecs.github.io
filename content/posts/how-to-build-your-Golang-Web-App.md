---
layout: post
title: 构建 Golang Web 应用的两种方法
categories: ["go"]
date: 2019-11-01 18:04:46
description: "对前后端分离的 Golang Web 项目，既可以用 `http.FileServer` 提供前端静态文件的访问，也可以使用 `go-bindata-assetfs` 将它打包和后端一起编译成一个二进制文件。"
---

对前后端分离的 web 项目，golang 后端可以以两种方式提供前端静态文件的访问：

1. 使用 `http.FileServer` 提供静态文件服务
2. 使用 `go-bindata-assetfs` 将前端静态文件打包成二进制数据，和后端构建到一起

<!--more-->

# 一、`http.FileServer` 提供静态文件服务

文件目录：

    go-web-prj/
    		static/
    				index.html
    		main.go

index.html :

    <!doctype html>
    <html>
      <body>
        <p>Hello World!</p>
      </body>
    </html>

main.go

    package main

    import (
        "log"
        "net/http"
    )

    func main() {
        // Create a fileServer handler that serves our static files.
        fileServer := http.FileServer(http.Dir("static/"))

        http.HandleFunc(
            "/",
            func(w http.ResponseWriter, r *http.Request) {
                fileServer.ServeHTTP(w, r)
            },
        )

        log.Fatal(http.ListenAndServe(":8080", nil))
    }

此时使用 `go run`  或  `go build` 将后端代码运行、构建，便可以在浏览器打开 `[localhost:8080](http://localhost:8080)` 访问。

不过，这种打包形式有一个缺点，就是必须保证 `static/`  目录的路径正确，如果修改或者不存在了，前端文件就不可以正常渲染。

使用 `go-bindata-assetfs` 打包前端静态文件可以解决这个问题。

## 二、使用 `go-bindata-assetfs` 打包前端文件

使用前先安装：

    $ go get github.com/go-bindata/go-bindata/...
    $ go get github.com/elazarl/go-bindata-assetfs/...

将前端文件打包：

    $ go-bindata-assetfs static/...

生成 `bindata_assetfs.go` 文件

修改 `main.go` 代码：

    package main

    import (
        "log"
        "net/http"
    )

    func main() {
        // Create a fileServer handler that serves our static files.
        http.Handle("/", http.FileServer(assetFS()))
        log.Fatal(http.ListenAndServe(":8080", nil))
    }

使用 `go mod` 方便接下来构建：

    go mod init go-web-prj

构建：

    go build main.go bin bindata_assetfs.go

然后运行生成的 `main` 文件，即可在浏览器打开 `[localhost:8080](http://localhost:8080)` 访问。

此时 `static/` 文件已打包进二进制文件，所以当前的 `static/` 修改甚至删除，都不影响网站的正常使用。



## 使用 Docker 部署

如果是打包好的二进制文件，使用 docker 部署十分简单， Dockerfille :

    FROM scratch

    ADD main /

    CMD ["/main"]

然后构建：

    dk build -t go-web-prj .

运行：

    dk run -p 8080:8080 web-test

如果出现：

    standard_init_linux.go:211: exec user process caused "exec format error"

之类的错误，那是因为二进制文件编译环境出错，对于 linux 的 docker 环境，通常要设置环境变量：

    env GOOS=linux GOARCH=arm64

更多的 go 编译环境参考：[Golang GOOS and GOARCH](https://gist.github.com/asukakenji/f15ba7e588ac42795f421b48b8aede63)


