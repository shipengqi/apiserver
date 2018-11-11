# 用户业务逻辑处理
本小节源码下载路径：[demo07](https://github.com/lexkong/apiserver_demos/tree/master/demo07)

## 配置路由信息
需要先在`router/router.go`文件中，配置路由信息：
```go
func Load(g *gin.Engine, mw ...gin.HandlerFunc) *gin.Engine {
    ...
	// 用户路由设置
	u := g.Group("/v1/user")
	{
		u.POST("", user.Create)         // 创建用户
		u.DELETE("/:id", user.Delete)   // 删除用户
		u.PUT("/:id", user.Update)      // 更新用户
		u.GET("", user.List)            // 用户列表
		u.GET("/:username", user.Get)   // 获取指定用户的详细信息
	}
    ...
	return g
}
```

> 在 RESTful API 开发中，API 经常会变动，为了兼容老的 API，引入了版本的概念，比如上例中的`/v1/user`，说明该 API 版本是 v1。

## 注册新的错误码
在`pkg/errno/code.go`文件中（详见 [demo07/pkg/errno/code.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/pkg/errno/code.go)），新增如下错误码：
```go
var (
	// Common errors
        ...

	ErrValidation       = &Errno{Code: 20001, Message: "Validation failed."}
	ErrDatabase         = &Errno{Code: 20002, Message: "Database error."}
	ErrToken            = &Errno{Code: 20003, Message: "Error occurred while signing the JSON web token."}

	// user errors
	ErrEncrypt           = &Errno{Code: 20101, Message: "Error occurred while encrypting the user password."}
	ErrTokenInvalid      = &Errno{Code: 20103, Message: "The token was invalid."}
	ErrPasswordIncorrect = &Errno{Code: 20104, Message: "The password was incorrect."}
)
```

## 新增用户
更新`handler/user/create.go`中`Create()`的逻辑，
更新后的内容见 [demo07/handler/user/create.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/handler/user/create.go)。

创建用户逻辑：
1. 从 HTTP 消息体获取参数（用户名和密码）
2. 参数校验
3. 加密密码
4. 在数据库中添加数据记录
5. 返回结果（这里是用户名）

从 HTTP 消息体解析参数，前面小节已经介绍了。

参数校验这里用的是`gopkg.in/go-playground/validator.v9`包（详见 [go-playground/validator](https://github.com/go-playground/validator)），实际开发过程中，
该包可能不能满足校验需求，这时候可在程序中加入自己的校验逻辑，比如在`handler/user/creater.go`中添加校验函数`checkParam`：
```go
package user

import (
    ...
)

// Create creates a new user account.
func Create(c *gin.Context) {
	log.Info("User Create function called.", lager.Data{"X-Request-Id": util.GetReqID(c)})
	var r CreateRequest
	if err := c.Bind(&r); err != nil {
		SendResponse(c, errno.ErrBind, nil)
		return
	}

	if err := r.checkParam(); err != nil {
		SendResponse(c, err, nil)
		return
	}
        ...
}

func (r *CreateRequest) checkParam() error {
	if r.Username == "" {
		return errno.New(errno.ErrValidation, nil).Add("username is empty.")
	}

	if r.Password == "" {
		return errno.New(errno.ErrValidation, nil).Add("password is empty.")
	}

	return nil
}
```
例子通过`Encrypt()`对密码进行加密：
```go
// Encrypt the user password.
func (u *UserModel) Encrypt() (err error) {
    u.Password, err = auth.Encrypt(u.Password)
    return
}
```
`Encrypt()`函数引用`auth.Encrypt()`来进行密码加密，
具体实现见 [demo07/pkg/auth/auth.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/pkg/auth/auth.go)。

最后例子通过`u.Create()`函数来向数据库中添加记录，ORM 用的是`gorm`，`gorm`详细用法请参考[GORM 指南](http://gorm.io/)。
在`Create()`函数中引用的数据库实例是`DB.Self`，该实例在 API 启动之前已经完成初始化。DB 是个全局变量，可以直接引用。

## 删除用户
删除用户代码详见 [demo07/handler/user/delete.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/handler/user/delete.go)。

删除时，首先根据 URL 路径`DELETE http://127.0.0.1/v1/user/1`解析出 id 的值 1，该 id 实际上就是数据库中的 id 索引，调用`model.DeleteUser()`函数删除，
函数详见 [demo07/model/user.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/model/user.go)。

## 更新用户
更新用户代码详见 [demo07/handler/user/update.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/handler/user/update.go)。

更新用户逻辑跟创建用户差不多，在更新完数据库字段后，需要指定 gorm model 中的 id 字段的值，因为 gorm 在更新时默认是按照 id 来匹配记录的。
通过解析 `PUT http://127.0.0.1/v1/user/1`来获取 id。

## 查询用户列表
查询用户列表代码详见 [demo07/handler/user/list.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/handler/user/list.go)。

一般在`handler`中主要做解析参数、返回数据操作，简单的逻辑也可以在`handler`中做，像新增用户、删除用户、更新用户，代码量不大，所以也可以放在`handler`中。
有些代码量很大的逻辑就不适合放在`handler`中，因为这样会导致`handler`逻辑不是很清晰，这时候实际处理的部分通常放在`service`包中。比如本例的`LisUser()`函数：
```go
package user

import (
    "apiserver/service"
    ...
)

// List list the users in the database.
func List(c *gin.Context) {
    ...
    infos, count, err := service.ListUser(r.Username, r.Offset, r.Limit)
    if err != nil {
        SendResponse(c, err, nil)
        return
    }
    ...
}
```
查询一个 REST 资源列表，通常需要做分页，如果不做分页返回的列表过多，会导致 API 响应很慢，前端体验也不好。本例中的查询函数做了分页，收到的请求中传入的`offset`和`limit`参数，
分别对应于 MySQL 的`offset`和`limit`。

`service.ListUser()`函数用来做具体的查询处理，代码详见 [demo07/service/service.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/service/service.go)。

在`ListUser()`函数中用了`sync`包来做并行查询，以使响应延时更小。在实际开发中，查询数据后，通常需要对数据做一些处理，比如`ListUser()`函数中会对每个用户记录返回一
个`sayHello`字段。`sayHello`只是简单输出了一个`Hello shortId`字符串，其中`shortId`是通过`util.GenShortId()`
来生成的（`GenShortId`实现详见 [demo07/util/util.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/util/util.go)）。像这类操作通常会增加 API
的响应延时，如果列表条目过多，列表中的每个记录都要做一些类似的逻辑处理，这会使得整个 API 延时很高，所以实际开发中通常会做并行处理。

在`ListUser()`实现中，有`sync.Mutex`和`IdMap`等部分代码，使用`sync.Mutex`是因为在并发处理中，更新同一个变量为了保证数据一致性，通常需要做锁处理。

使用`IdMap`是因为查询的列表通常需要按时间顺序进行排序，一般数据库查询后的列表已经排过序了，但是为了减少延时，程序中用了并发，这时候会打乱排序，
所以通过`IdMap`来记录并发处理前的顺序，处理后再重新复位。

## 获取指定用户的详细信息
代码详见 [demo07/handler/user/get.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/handler/user/get.go)。

获取指定用户信息时，首先根据 URL 路径`GET http://127.0.0.1/v1/user/admin`解析出`username`的值`admin`，然后调用`model.GetUser()`函数查询该用户的数据库记录并返回，
函数详见 [demo07/model/user.go](https://github.com/lexkong/apiserver_demos/blob/master/demo07/model/user.go)。



