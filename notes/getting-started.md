# 启动一个最简单的 RESTful API 服务器
Go 语言 API 开发中常用的组合是 `gRPC + Protobuf` 和 `REST + JSON`。

## 什么是 REST
REST（REpresentational State Transfer）代表表现层状态转移。REST 是一种软件架构风格，不是技术框架，REST 有一系列规范，满足这
些规范的 API 均可称为 RESTful API。 REST 规范中有如下几个核心：

1. REST 中一切实体都被抽象成资源，每个资源有一个唯一的标识 —— URI，所有的行为都应该是在资源上的 CRUD 操作
2. 使用标准的方法来更改资源的状态，常见的操作有：资源的增删改查操作
3. 无状态：这里的无状态是指每个 RESTful API 请求都包含了所有足够完成本次操作的信息，服务器端无须保持 Session

在 HTTP 协议中通过 POST、DELETE、PUT、GET 方法来对应 REST 资源的增、删、改、查操作:

| HTTP 方法	| 行为 | URI | 示例说明 |
| ------ | ------ | ------ | ------ |
| GET | 获取资源列表	 | `/users` | 获取用户列表 |
| GET | 获取一个具体的资源 | `/users/admin` | 获取 admin 用户的详细信息 |
| POST | 创建一个新的资源 | `/users` | 创建一个新用户 |
| PUT | 以整体的方式更新一个资源	 | `/users/1` | 更新 id 为 1 的用户 |
| DELETE | 删除服务器上的一个资源 | `/users/1` | 删除 id 为 1 的用户 |


## 什么是 RPC
RPC（Remote Procedure Call，RPC）远程过程调用。是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程
序员无须额外地为这个交互作用编程。通俗来讲，就是服务端实现了一个函数，客户端使用 RPC 框架提供的接口，调用这个函数的实现，并获取返
回值。RPC 屏蔽了底层的网络通信细节，使得开发人员无须关注网络编程的细节，而将更多的时间和精力放在业务逻辑本身的实现上，从而提高开发效率。

RPC 的调用过程如下（图片来自 
[How RPC Works](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc738291(v=ws.10))）：

![RPC](images/rpc.jpg)

1. Client 通过本地调用，调用 Client Stub
2. Client Stub 将参数打包（也叫 Marshalling）成一个消息，然后发送这个消息
3. Client 所在的 OS 将消息发送给 Server
4. Server 端接收到消息后，将消息传递给 Server Stub
5. Server Stub 将消息解包（也叫 Unmarshalling）得到参数
6. Server Stub 调用服务端的子程序（函数），处理完后，将最终结果按照相反的步骤返回给 Client

> Stub 负责调用参数和返回值的流化（serialization）、参数的打包解包，以及负责网络层的通信。Client 端一般叫 Stub，Server 端一般
>叫 Skeleton。

## REST vs RPC
RPC 相比 REST 的优点主要有 3 点：
1. RPC+Protobuf 采用的是 TCP 做传输协议，REST 直接使用 HTTP 做应用层协议，这种区别导致 REST 在调用性能上会比 RPC+Protobuf 低
2. RPC 不像 REST 那样，每一个操作都要抽象成对资源的增删改查，在实际开发中，有很多操作很难抽象成资源，比如登录操作。所以在实际开发中
并不能严格按照 REST 规范来写 API，RPC 就不存在这个问题
3. RPC 屏蔽网络细节、易用，和本地调用类似

REST 相较 RPC 的优势：
1. 轻量级，简单易用，维护性和扩展性都比较好
2. REST 相对更规范，更标准，更通用，无论哪种语言都支持 HTTP 协议，可以对接外部很多系统，只要满足 HTTP 调用即可，更适合对外，RPC 会
有语言限制，不同语言的 RPC 调用起来很麻烦
3. JSON 格式可读性更强，开发调试都很方便
4. 在开发过程中，如果严格按照 REST 规范来写 API，API 看起来更清晰，更容易被大家理解

业界普遍采用的做法是，内部系统之间调用用 RPC，对外用 REST，因为内部系统之间可能调用很频繁，需要 RPC 的高性能支撑。对外用 REST 更
易理解，更通用些。当然以现有的服务器性能，如果两个系统间调用不是特别频繁，对性能要求不是非常高，REST 的性能完全可以满足。此外 REST 在
实际开发中，能够满足绝大部分的需求场景，所以 RPC 的性能优势可以忽略。

## HTTP API 服务器启动流程

![httpprocess](images/httpprocess.jpg)

注意，这里在启动 HTTP 端口之前，程序会 go 一个协程，来ping HTTP 服务器的 `/sd/health` 接口，如果程序成功启动，ping 协程在 timeout 
之前会成功返回，如果程序启动失败，则 ping 协程最终会 timeout，并终止整个程序。

## 目录结构
```
├── admin.sh                     # 进程的start|stop|status|restart控制文件
├── conf                         # 配置文件统一存放目录
│   ├── config.yaml              # 配置文件
│   ├── server.crt               # TLS配置文件
│   └── server.key
├── config                       # 专门用来处理配置和配置文件的Go package
│   └── config.go
├── db.sql                       # 在部署新环境时，可以登录MySQL客户端，执行source db.sql创建数据库和表
├── docs                         # swagger文档，执行 swag init 生成的
│   ├── docs.go
│   └── swagger
│       ├── swagger.json
│       └── swagger.yaml
├── handler                      # 类似MVC架构中的C，用来读取输入，并将处理流程转发给实际的处理函数，最后返回结果
│   ├── handler.go
│   ├── sd                       # 健康检查handler
│   │   └── check.go
│   └── user                     # 核心：用户业务逻辑handler
│       ├── create.go            # 新增用户
│       ├── delete.go            # 删除用户
│       ├── get.go               # 获取指定的用户信息
│       ├── list.go              # 查询用户列表
│       ├── login.go             # 用户登录
│       ├── update.go            # 更新用户
│       └── user.go              # 存放用户handler公用的函数、结构体等
├── main.go                      # Go程序唯一入口
├── Makefile                     # Makefile文件，一般大型软件系统都是采用make来作为编译工具
├── model                        # 数据库相关的操作统一放在这里，包括数据库初始化和对表的增删改查
│   ├── init.go                  # 初始化和连接数据库
│   ├── model.go                 # 存放一些公用的go struct
│   └── user.go                  # 用户相关的数据库CURD操作
├── pkg                          # 引用的包
│   ├── auth                     # 认证包
│   │   └── auth.go
│   ├── constvar                 # 常量统一存放位置
│   │   └── constvar.go
│   ├── errno                    # 错误码存放位置
│   │   ├── code.go
│   │   └── errno.go
│   ├── token
│   │   └── token.go
│   └── version                  # 版本包
│       ├── base.go
│       ├── doc.go
│       └── version.go
├── README.md                    # API目录README
├── router                       # 路由相关处理
│   ├── middleware               # API服务器用的是Gin Web框架，Gin中间件存放位置
│   │   ├── auth.go
│   │   ├── header.go
│   │   ├── logging.go
│   │   └── requestid.go
│   └── router.go
├── service                      # 实际业务处理函数存放位置
│   └── service.go
├── util                         # 工具类函数存放目录
│   ├── util.go
│   └── util_test.go
└── vendor                         # vendor目录用来管理依赖包
    ├── github.com
    ├── golang.org
    ├── gopkg.in
    └── vendor.json
```

## 选择 web 框架
[RESTful Web 框架 性能对比](https://github.com/gin-gonic/gin/blob/master/BENCHMARKS.md)。

我们这里使用 [Gin](https://github.com/gin-gonic/gin)，[简介](https://www.jianshu.com/p/a31e4ee25305)。

## 启动 HTTP 服务
在 `main()` 函数中主要做一些配置文件解析、程序初始化和路由加载之类的事情，最终调用 `http.ListenAndServe()` 在指定端口启动一
个 HTTP 服务器。

`main.go`：
```go
package main

import (
	"log"
	"net/http"

	"apiserver/router"

	"github.com/gin-gonic/gin"
)

func main() {
	// Create the Gin engine.
	g := gin.New()

    // gin middlewares
	middlewares := []gin.HandlerFunc{}

	// Routes.
	router.Load(
		// Cores.
		g,

		// Middlewares.
		middlewares...,
	)

	log.Printf("Start to listening the incoming requests on http address: %s", ":8080")
	log.Printf(http.ListenAndServe(":8080", g).Error())
}
```

### 加载路由
通过调用 `router.Load` 函数来加载路由，
[demo01/router/router.go](https://github.com/lexkong/apiserver_demos/blob/master/demo01/router/router.go)：
```go
"apiserver/handler/sd"

    ....

// The health check handlers
svcd := g.Group("/sd")
{
    svcd.GET("/health", sd.HealthCheck)
    svcd.GET("/disk", sd.DiskCheck)
    svcd.GET("/cpu", sd.CPUCheck)
    svcd.GET("/ram", sd.RAMCheck)
}
```
定义了一个叫 `sd` 的分组，在该分组下注册了 `/health`、`/disk`、`/cpu`、`/ram` HTTP 路径，分别路由到 `sd.HealthCheck`、
`sd.DiskCheck`、`sd.CPUCheck`、`sd.RAMCheck` 函数。`sd` 分组主要用来检查 API Server 的状态：健康状况、服务器硬盘、CPU 和内
存使用量。
[demo01/handler/sd/check.go](https://github.com/lexkong/apiserver_demos/blob/master/demo01/handler/sd/check.go)

### 设置 HTTP Header
通过 `g.Use()` 来为每一个请求设置 `Header`，在 `router/router.go` 文件中设置 `Header`：
```go
    // 在处理某些请求时可能因为程序 bug 或者其他异常情况导致程序 panic，这时候为了不影响下一次请求的调用，
    // 需要通过 gin.Recovery() 来恢复 API  服务器
    g.Use(gin.Recovery())
    // 强制浏览器不使用缓存
    g.Use(middleware.NoCache)
    // 浏览器跨域 OPTIONS 请求设置
    g.Use(middleware.Options)
    // 一些安全设置
    g.Use(middleware.Secure)
```

### 健康状态自检
有时候 API 进程起来不代表 API 服务器正常，如：API 进程存在，但是服务器却不能对外提供服务。因此在启动 API 服务器时，如果能够最后做
一个自检会更好些。在 apiserver 中添加自检程序，在启动 HTTP 端口前 `go` 一个 `pingServer` 协程，启动 HTTP 端口后，该协程不
断地 `ping/sd/health` 路径，如果失败次数超过一定次数，则终止 HTTP 服务器进程。通过自检可以最大程度地保证启动后的 API 服务器处于
健康状态。自检部分代码位于 `main.go` 中：
```go
func main() {
    ....

    // Ping the server to make sure the router is working.
    go func() {
        if err := pingServer(); err != nil {
            log.Fatal("The router has no response, or it might took too long to start up.", err)
        }
        log.Print("The router has been deployed successfully.")
    }()
    ....
}

// pingServer pings the http server to make sure the router is working.
func pingServer() error {
    for i := 0; i < 10; i++ {
        // Ping the server by sending a GET request to `/health`.
        resp, err := http.Get("http://127.0.0.1:8080" + "/sd/health")
        if err == nil && resp.StatusCode == 200 {
            return nil
        }

        // Sleep for a second to continue the next ping.
        log.Print("Waiting for the router, retry in 1 second.")
        time.Sleep(time.Second)
    }
    return errors.New("Cannot connect to the router.")
}
```
`http.Get` 向 `http://127.0.0.1:8080/sd/health` 发送 HTTP GET 请求，如果函数正确执行并且返回的 HTTP StatusCode 为 200，则
说明 API 服务器可用，`pingServer` 函数输出部署成功提示；如果超过指定 10 次，`pingServer` 直接终止 API Server 进程。

### 编译源码
下载 [源码](https://github.com/lexkong/apiserver_demos)。将 `apiserver_demos/demo01` 复制为 `$GOPATH/src/apiserver`。

首次编译需要下载 vendor（依赖管理工具） 包:
```bash
$ cd $GOPATH/src
$ git clone https://github.com/lexkong/vendor
```

编译：
```bash
$ cd $GOPATH/src/apiserver
# 每次编译前对 Go 源码进行格式化和代码静态检查
$ gofmt -w .
$ go tool vet .
$ go build -v .
```
### 安装 Dep
Dep 也是 Go 的依赖管理工具。
```bash
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
```

保证 `$GOPATH/bin` 存在。建议配置环境变量 `GOBIN` 为 `$GOPATH/bin`。

### 测试API
```bash
# 启动 api server
$ ./apiserver

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /sd/health                --> apiserver/handler/sd.HealthCheck (5 handlers)
[GIN-debug] GET    /sd/disk                  --> apiserver/handler/sd.DiskCheck (5 handlers)
[GIN-debug] GET    /sd/cpu                   --> apiserver/handler/sd.CPUCheck (5 handlers)
[GIN-debug] GET    /sd/ram                   --> apiserver/handler/sd.RAMCheck (5 handlers)
Start to listening the incoming requests on http address: :8080
The router has been deployed successfully.

# 测试
$ curl -XGET http://127.0.0.1:8080/sd/health
OK

$ curl -XGET http://127.0.0.1:8080/sd/disk
OK - Free space: 16321MB (15GB) / 51200MB (50GB) | Used: 31%

$ curl -XGET http://127.0.0.1:8080/sd/cpu
CRITICAL - Load average: 2.39, 2.13, 1.97 | Cores: 2

$ curl -XGET http://127.0.0.1:8080/sd/ram
OK - Free space: 455MB (0GB) / 8192MB (8GB) | Used: 5%
```
