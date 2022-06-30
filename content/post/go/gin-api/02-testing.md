---
title: "构建自己的go-gin-api脚手架(二)"
date: 2022-01-11T13:41:12+08:00
draft: false
categories: ["Go"]
tags: ["Go", "Gin", "testing", "Testify", "GoConvey", "httptest", "单元测试"]
---

在前文中我们已经成功的使用 Gin 框架搭建了一个简易的 API 服务，但是在一个项目中，所有的业务逻辑都写在 **main.go** 一个文件中显然是不合理的。所以接下来我们会对其进行重构，在整个搭建 go-gin-api 脚手架的过程中，会经常对其重构。

> 每当我要进行重构的时候，第一个步骤永远相同：我得确保即将修改的代码拥有一组可靠的测试。这些测试必不可少，因为尽管遵循重构手法可以使我避免绝大多数引入 bug 的情形，但我毕竟是人，毕竟有可能犯错。程序越大，我的修改不小心破坏其他代码的可能性就越大——在数字时代，软件的名字就是脆弱。 —— 《重构：改善既有代码的设计》

所以在此之前我们需要编写一套可靠的单元测试，来确保代码重构之后还能良好的运行。

## Go 单元测试框架

### testing

**testing** 是 Go 标准库中提供的自动化测试支持，通过 `go test` 命令，能够自动执行如下形式的任何函数：

```go
func TestXxx(*testing.T)
```

注意：Xxx 可以是任何字母数字字符串，但是第一个字母不能是小写字母。

在这些函数中，使用 `Error`、`Fail` 或相关方法来发出失败信号。

要编写一个新的测试套件，需要创建一个名称以 \_test.go 结尾的文件，该文件包含 `TestXxx` 函数，如上所述。 将该文件放在与被测试文件相同的包中。该文件将被排除在正常的程序包之外，但在运行 `go test` 命令时将被包含。 有关详细信息，请运行 `go help test` 和 `go help testflag` 了解。

标准库的 **testing** 实现比较简单，并不支持断言，需要写 if 判断。所以不考虑使用。

### Testify

> 地址：[https://github.com/stretchr/testify](https://github.com/stretchr/testify) ![](https://img.shields.io/github/stars/stretchr/testify?style=social)

Testify 是基于 testing 编写的，所以测试文件与执行方式与其完全相同，并且支持断言方法。

#### 编写测试

首先创建一个 **main_test.go** 的文件，写入以下代码并执行 `go mod tidy`更新依赖。

**main_test.go**

```go
package main

import (
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestTestify(t *testing.T) {
    // 断言是否相等 其他断言请自行查询文档
	assert.Equal(t, 1, 1)
}
```

#### 执行测试

打开 Goland 的 Terminal 命令行工具，执行`go test`

##### 正确断言

![image-20220112145057246](https://image.chance.fyi/image-20220112145057246.png)

##### 错误断言

然后将断言修改成错误看一下，可以看到错误信息很清晰并且可以点击跳转到对应的错误位置。

![image-20220112145257507](https://image.chance.fyi/image-20220112145257507.png)

Go 中推荐将单元测试与源码放置在一起，不过我更喜欢将所有单元测试集中到 **tests** 目录中去。接下来创建一个 **tests** 目录，将 **main_test.go** 移至 **test** 目录中去，**注意包名**，再次执行`go test`发现找不到测试文件了。需要使用`go test ./...`递归执行当前目录下所有的测试。

![image-20220112150411570](https://image.chance.fyi/image-20220112150411570.png)

![image-20220112164806365](https://image.chance.fyi/image-20220112164806365.png)

#### 使用 Goland 执行单元测试

![image-20220112160452891](https://image.chance.fyi/image-20220112160452891.png)

![image-20220112160512007](https://image.chance.fyi/image-20220112160512007.png)

### GoConvey

> 地址：[https://github.com/smartystreets/goconvey](https://github.com/smartystreets/goconvey) ![](https://img.shields.io/github/stars/smartystreets/goconvey?style=social)

#### 安装

```bash
$ go get github.com/smartystreets/goconvey
```

#### 编写测试

修改 **main_test.go** 文件并执行测试。

**main_test.go**

```go
package tests

import (
	. "github.com/smartystreets/goconvey/convey"
	"testing"
)

func TestConvey(t *testing.T) {
	Convey("判断两值相等", t, func() {
		So(1, ShouldEqual, 1)
		Convey("判断两数不等", func() {
			So(1, ShouldNotEqual, 2)
		})
	})
}
```

每个测试用例必须使用 **Convey** 函数包裹起来，一个 **Convey** 就是一个测试用例，并且可以嵌套使用

**Convey** 参数

- 第一个参数为测试用例的描述
- 第二个参数为测试函数的参数类型为`*testing.T`最外层必须传入，内层 **Convey** 可不传
- 第三个参数为无参数、无返回值的函数，通常使用闭包，内部使用 **So** 函数进行断言

**So** 参数

- actual 输入
- assert 断言
- expected 期望值

#### 执行测试

##### 正确断言

![image-20220112170509891](https://image.chance.fyi/image-20220112170509891.png)

##### 错误断言

![image-20220112170536905](https://image.chance.fyi/image-20220112170536905.png)

可以看到 **GoConvey** 相比 **Testify** 输出内容多了些，好像还显得有些杂乱，作者说的是有彩色输出的，但我测试的并没有，更重要的是不能直接点击跳转到错误位置。但是 **GoConvey** 有一个很好用的功能**Web UI** 。

#### Web UI

控制台执行命令`goconvey`，然后打开 [http://127.0.0.1:8080/](http://127.0.0.1:8080/)

```bash
$ goconvey
2022/01/12 17:24:12 goconvey.go:116: GoConvey server:
2022/01/12 17:24:12 goconvey.go:121:   version: v1.7.2
2022/01/12 17:24:12 goconvey.go:122:   host: 127.0.0.1
2022/01/12 17:24:12 goconvey.go:123:   port: 8080
2022/01/12 17:24:12 goconvey.go:124:   poll: 250ms
2022/01/12 17:24:12 goconvey.go:125:   cover: true
2022/01/12 17:24:12 goconvey.go:126:
2022/01/12 17:24:12 tester.go:19: Now configured to test 10 packages concurrently.
2022/01/12 17:24:12 goconvey.go:243: Serving HTTP at: http://127.0.0.1:8080
...
```

###### 正确断言

![image-20220112172619292](https://image.chance.fyi/image-20220112172619292.png)

###### 错误断言

![image-20220112172726527](https://image.chance.fyi/image-20220112172726527.png)

GoConvey 会在你修改代码之后会自动执行单元测试，同时 web 页面会自动更新，如果有错误断言，就会变成 FAIL 并显示出错误位置，还可以打开 H5 的通知功能，在单元测试运行失败后，通过桌面弹窗通知提醒你。

![image-20220112173217342](https://image.chance.fyi/image-20220112173217342.png)

##### 生成代码

![image-20220113131536143](https://image.chance.fyi/image-20220113131536143.png)

## 对比

testing 功能太过简单，而且没有断言，就不考虑了。Testify 相比 GoConvey 控制台输出效果要好一些，但是 GoConvey 可以使用 WebUI 很方便清晰的管理测试用例，自动执行测试用例，并且如果不想要频繁切换页面查看还可以使用桌面弹窗来通知错误。

如果你不喜欢 GoConvey 的写法，但是又想要自动测试加错误弹窗提醒，其实 Testify 是可以和 GoConvey 一起用的。你可以使用 Testify 来编写测试用例，同时启动 GoConvey 来享受它的一些特性，比如自动运行与错误弹窗提醒，但是 WebUI 的功能会有些缺失。

![image-20220113131103708](https://image.chance.fyi/image-20220113131103708.png)

## http 测试

测试框架最终考虑使用 GoConvey ，对我们的接口进行测试，因为 GoConvey 使用的是`8080`端口，所以 **main.go** 我们改成监听`8000`端口，将 `r.Run()`改为`r.Run(":8000")`，然后启动服务编写测试用例进行测试。

**main_test.go**

```go
package tests

import (
	. "github.com/smartystreets/goconvey/convey"
	"io/ioutil"
	"net/http"
	"testing"
)

func TestPingRoute(t *testing.T) {
	Convey("api ping", t, func() {
		get, err := http.Get("http://127.0.0.1:8000/ping")
		So(err, ShouldEqual, nil)
		So(get.StatusCode, ShouldEqual, 200)
		body, _ := ioutil.ReadAll(get.Body)
		So(string(body), ShouldEqual, "{\"message\":\"pong\"}")
	})
}
```

单元测试通过我们就可以放心大胆的重构代码了，但是每次单元测试接口都要启动服务，修改了还要重启，太过麻烦。不要急我们先进行重构，创建以下三个文件。

- boot/boot.go
- boot/router.go
- routers/api.go

**boot/boot.go**

```go
package boot

type boot struct{}

var Boot = boot{}

func (*boot) Init() {
	// 初始化路由
	Route.Init()
}
```

**boot/router.go**

```go
package boot

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"go-gin-api/routers"
	"net/http"
)

type route struct{}

var Route = route{}

func (*route) Init() {
	err := SetRouter().Run(":8000")
	if err != nil {
		fmt.Println(err.Error())
	}
}

// SetRouter 设置路由与服务启动分开方便单元测试
func SetRouter() *gin.Engine {
	r := gin.Default()
	routers.Init(r)
	setup404Handler(r)
	return r
}

// 处理404请求
func setup404Handler(r *gin.Engine) {
	r.NoRoute(func(ctx *gin.Context) {
		ctx.JSON(http.StatusNotFound, gin.H{
			"code":    404,
			"message": "not found",
		})
	})
}
```

**routers/api.go**

```go
package routers

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Init(r *gin.Engine) {
	r.GET("/ping", func(ctx *gin.Context) {
		ctx.JSON(http.StatusOK, gin.H{
			"message": "pong",
		})
	})
}
```

修改 **main.go**

```go
package main

import "go-gin-api/boot"

func main() {
	boot.Boot.Init()
}
```

### httptest

使用 Go 标准库中的`net/http/httptest`包测试 http 请求，就可以发送一个 http request 而不用真正的去启动一个 http server。

修改 **main_test.go**

```go
package tests

import (
	. "github.com/smartystreets/goconvey/convey"
	"go-gin-api/boot"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestPingRoute(t *testing.T) {
	router := boot.SetRouter()

	Convey("api ping", t, func() {
		w := httptest.NewRecorder()
		req, _ := http.NewRequest("GET", "/ping", nil)
		router.ServeHTTP(w, req)

		So(w.Code, ShouldEqual, http.StatusOK)
		So(w.Body.String(), ShouldEqual, "{\"message\":\"pong\"}")
	})

	Convey("404 request", t, func() {
		w := httptest.NewRecorder()
		req, _ := http.NewRequest("GET", "/404", nil)
		router.ServeHTTP(w, req)

		So(w.Code, ShouldEqual, http.StatusNotFound)
		So(w.Body.String(), ShouldEqual, "{\"code\":404,\"message\":\"not found\"}")
	})
}
```

现在将 http 服务关闭，再次运行单元测试之后发现依然是成功的。
