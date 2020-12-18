---
title: '利用 swagger 管理 API 开发'
date: 2019-11-20 21:48:25
description: ""
tags:
---

swagger 可以通过 yaml 文件自动生成 api 文档，客户端代码等，减少了许多繁琐的工作和前后端沟通，从而提高开发效率。
<!--more-->

第一次知道 swagger 这个工具， 是在 docker 的源码里，docker 客户端和服务端的 API 就是使用 swagger 来管理。最近在一次技术交流会上，听人提到用 swagger 来自动生成 api 文档，省下了许多前后端沟通的工夫，便留心记了下来。

前几天抽空研究了一下，发现使用 swagger 进行 api 开发，确实可以减少了许多繁琐的工作，大大提高开发和前后端沟通的效率。使用 swagger ，只要在项目里写好 `swagger.yaml `文件，其他开发人员需要了解 api 的信息时，把它放到线上或下载下来的 swagger editor，就能自动生成 api 文档，调试接口，还可以生成多种语言的客户端代码、服务端代码。

swagger 最主要的优点，总结有以下几点：

- 记录 api 信息只要一个 yaml 文件，用的时候放到 editor 就能直接生成文档。yaml 容易编辑维护
- 可以生成多种客户端代码，客户端调接口只要传参数，简化使用方法
- 可以在 editor 请求调试接口



## Swagger `yaml`文件示例

Swagger 定义 api 的标准文件可以是 json 格式也可以是 yaml 格式，但是使用 yaml 可读性更高和更容易编写。

Swagger 目前最新版本是 OpenAPI 3.0 （ Swagger Specification 已经被捐献给 linux 基金会，更名为 [OpenAPI Specification](https://www.openapis.org/) (OAS) ，因此 Swagger 和 OpenAPI 基本可以理解为一个东西），虽然 3.0 版本也可以生成的客户端代码，但支持语言没有 2.0 的多。本文还是使用一个 2.0 的 yaml 文件作为示例，介绍 swagger 的基本语法。不过，2.0 的 yaml 可以在 swagger editor 里自动转成 3.0。

### api 文档的基本信息

Yaml 文件开头需要声明版本：

```yaml
swagger: "2.0"
```

紧跟着是项目信息

```yaml
info:
  version: 1.0.0
  title: Swagger Petstore
  license:
    name: MIT
```

然后是请求地址、格式等内容：

```yaml
host: petstore.swagger.io
basePath: /v1
schemes:
  - http
consumes:
  - application/json
produces:
  - application/json
```

consumes 和 produces 分别表示请求和返回的 `Content-Type` （本例是: `Content-Type: application/json`），接下的接口如果没有另外声明这两个字段，则都是这个格式。



### 在 `yaml  `定义一个 `GET `请求接口

 `paths` 字段可以定义具体接口请求方式、参数等，例如定义一个` GET  /pets` 接口：

```yaml
paths:
  /pets:
    get:
      summary: List all pets
      operationId: listPets
      tags:
        - pets
      parameters:
        - name: limit
          in: query
          description: How many items to return at one time (max 100)
          required: false
          type: integer
          format: int32
      responses:
      	.....

```

在上面 yaml 片段中：

- Summary 是接口介绍
- operationId 作为该接口的唯一标识
- tags 是给接口的归类
- parameters 即请求参数，此处只有一个 limit 的接口，以 query 方式传递



接上一个 yaml 片段，返回格式是：

```yaml
			responses:
        "200":
          description: A paged array of pets
          headers:
            x-next:
              type: string
              description: A link to the next page of responses
          schema:
          	type: array
          	items:
          		type: "object"
              required:
                - id
                - name
              properties:
                id:
                  type: integer
                  format: int64
                name:
                  type: string
                tag:
                  type: string
        default:
          description: unexpected error
          schema:
            type: "object"
            required:
              - code
              - message
            properties:
              code:
                type: integer
                format: int32
              message:
                type: string
```

在这个 `yaml` 片段里，分别定义**Status Code** 为 200 和其它的返回值。

schema 就是返回数据的格式，对应 200 的返回格式即是：

```json
[
  {
    "id": 0,
    "name": "string",
    "tag": "string"
  }
]
```



### `POST` 请求接口示例

因为请求路径相同， `POST /pets` 可以跟在 `GET /pets` 下方：

```yaml
paths:
  /pets:
    get:
      ...
      ...
		post:
      summary: Create a pet
      operationId: createPets
      tags:
        - pets
      parameters:
        - name: "pet"
          in: body
          description: creat a pet
          schema:
            type: "object"
            required:
              - id
              - name
            properties:
              id:
                type: integer
                format: int64
              name:
                type: string
              tag:
                type: string
      responses:
        "201":
          description: Null response
        default:
         ……
```

POST 请求的参数通过 body 传递，因此 `parameters` 写明 `in: body`,  最终生成的请求示例如下：

```json
{
  "id": 0,
  "name": "string",
  "tag": "string"
}
```



### 使用 `$ref`  复用数据类型

对比 get 和 post 的 yaml 片段，可以发现其中有不少重复的内容，比如 parameters 和 response 的 scheme 多次出现：

```yaml
				schema:
            type: "object"
            required:
              - id
              - name
            properties:
              id:
                type: integer
                format: int64
              name:
                type: string
              tag:
                type: string
```

这些可以定义一些可复用的数据格式，然后用 `$ref` 引入：

```yaml
definitions:
  Pet:
    type: "object"
    required:
      - id
      - name
    properties:
      id:
        type: integer
        format: int64
      name:
        type: string
      tag:
        type: string
  Pets:
    type: array
    items:
      $ref: '#/definitions/Pet'
  Error:
    type: "object"
    required:
      - code
      - message
    properties:
      code:
        type: integer
        format: int32
      message:
        type: string
```

swagger yaml 基本的写法大致就是这样，最终完整的 yaml 文件见：[**uber.yaml**](https://github.com/OAI/OpenAPI-Specification/blob/master/examples/v2.0/yaml/uber.yaml)

## 使用 swagger editor 生成文档、代码

swagger 有一些可以配套使用的工具，editor 就是其中之一。完成的 yaml ，任何时候想查看接口文档，只要把 yaml 文件放到 editor，就能自动生成 api 文档。

以下截图是放到 [online editor](https://editor.swagger.io/) 的效果:
![swagger-editor](/images/swagger-editor.png)


在 editor 里还可以根据自己需要生成客户端代码，服务端代码。客户端代码可以类似 sdk 那样接入项目，调用时只要传入参数即可。

服务端代码是一个模版，帮你定义好最终输出数据的格式，而中间的实现逻辑，还需要自己完成。



## swagger 的不足

swagger 总体上可以大大提高 api 开发的效率，不过试用的过程中，还是发现了美中不足之处。

- 无法通过 json 数据自动生成 schema。

  写 yaml 文件毕竟还是要花一些时间，尤其是定义返回数据的`schema`。假如可以导入接口返回的 json 数据，自动生成 schema 的 properties 等信息，又会方便了许多。

  

**参考文章：**

- [Intro to Swagger – A Structured Approach to Creating an API](https://spin.atomicobject.com/2018/08/30/swagger-api-intro/)
- [Writing OpenAPI (fka Swagger) Specification tutorial](http://apihandyman.io/writing-openapi-swagger-specification-tutorial-part-3-simplifying-specification-file/#writing-openapi-fka-swagger-specification-tutorial)

