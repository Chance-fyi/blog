---
title: "构建自己的go-gin-api脚手架(三)"
date: 2022-02-27T09:34:23+08:00
draft: false
categories: ["Go"]
tags: ["Go", "Gin", "Viper", "Config"]
---

## Viper

> 地址：[https://github.com/spf13/viper](https://github.com/spf13/viper)![](https://img.shields.io/github/stars/spf13/viper?style=social)

Viper 是适用于 Go 应用程序的完整配置解决方案。它被设计用于在应用程序中工作，并且可以处理所有类型的配置需求和格式。

### 文档

- 英文：可查看 Github 仓库的 [README](https://github.com/spf13/viper/blob/master/README.md) 文件
- 中文：可查看这篇[文章](https://www.cnblogs.com/you-men/p/14694780.html#_label0)的翻译

### 安装

```shell
go get github.com/spf13/viper
```

### 使用

新建配置文件`/config/config.toml`

```toml
aaa = "111"
bbb = "222"
```

Viper 只需简单设置就可以读取到配置信息。

```go
//设置文件路径
viper.AddConfigPath("/config")
//设置文件名
viper.SetConfigName("config")
//设置文件类型
viper.SetConfigType("toml")
//读取配置文件
_ = viper.ReadInConfig()
//监控配置文件变动
viper.WatchConfig()

for {
    //输出所有配置
    fmt.Println(viper.AllSettings())
    time.Sleep(time.Second)
}
```

### 多配置文件

随着项目开发，配置文件中的信息可能会越来越多，全部集中到一个文件中会变的难以管理。按功能分为不同的配置文件更易于管理，我们来自己实现一个多配置文件的方法。

#### 方法一

使用`MergeInConfig()`方法将多个配置文件内容合并到一起。

在新建一个`/config/router.toml`配置文件

```toml
port = 8000
```

```go
//设置文件路径
viper.AddConfigPath("/config")
//设置文件名
viper.SetConfigName("config")
//设置文件类型
viper.SetConfigType("toml")
//读取配置文件
_ = viper.ReadInConfig()

//设置第二个配置文件名称
viper.SetConfigName("router")
//读取并合并已有配置
_ = viper.MergeInConfig()

//输出所有配置
fmt.Println(viper.AllSettings())
//map[aaa:111 bbb:222 port:8000]
```

这种方法存在一个问题，多个配置文件中不能存在相同的 key 值，不然后一个配置文件会覆盖掉前一个配置文件相同 key 的值。

#### 方法二

使用配置文件名作为 key ，其中配置信息作为 value ，这样即使不同文件中存在相同的值也无妨了。

新建一个 config 包对 Viper 配置文件进行处理

**/pkg/config/config.go**

```go
package config

import (
	"github.com/spf13/viper"
	"io/ioutil"
	"path"
	"strings"
)

var cfg *viper.Viper

func Init() {
	dirname := "config"
    //创建一个新的Viper实例
	cfg = viper.New()

	viper.AddConfigPath(dirname)

    //循环读取路径下的所有配置文件
    //将文件名作为key配置信息作为value
    //set到创建的cfg实例中
	for _, cf := range readConfigFile(dirname) {
		viper.SetConfigType(cf["ext"])
		viper.SetConfigName(cf["name"])
		err := viper.ReadInConfig()
		if err != nil {
			panic(err)
		}
		s := viper.AllSettings()
		if len(s) > 0 {
			cfg.Set(cf["name"], s)
		}
	}
}

//读取路径下的所有文件
func readConfigFile(dirname string) (configFile []map[string]string) {
	dir, err := ioutil.ReadDir(dirname)
	if err != nil {
		panic(err)
	}

	for _, fileInfo := range dir {
		fileName := fileInfo.Name()
		ext := path.Ext(fileName)

		configFile = append(configFile, map[string]string{
			"name": fileName[0 : len(fileName)-len(ext)],
			"ext":  strings.Trim(ext, "."),
		})
	}
	return
}

//因为cfg实例设置的是私有变量
//所以要访问Viper的方法需要重新定义一下
func AllSettings() map[string]interface{} {
	return cfg.AllSettings()
}
```

然后在`/boot/boot.go`的`Init()`方法中调用`config.Init()`进行配置的初始化操作。

**读取所有配置信息**

```go
fmt.Println(config.AllSettings())
//map[config:map[aaa:111 bbb:222] router:map[port:8000]]
```

### 监控文件变动

多配置文件方法要实现监控文件的变动，就不能只是简单的添加一个`viper.WatchConfig()`方法就可以了，因为我们创建了一个新的 Viper 实例来保存多文件的配置，这个方法并不能影响到这个实例。但我们可以对每一个配置文件进行监控，然后使用`viper.OnConfigChange()`方法进行回调处理。

```go
func configChange(v *viper.Viper) {
    //监控文件变化
	v.WatchConfig()
    //文件变化后的回调处理
	v.OnConfigChange(func(in fsnotify.Event) {
		fileName := filepath.Base(in.Name)
		ext := path.Ext(fileName)
		name := fileName[0 : len(fileName)-len(ext)]
		v.SetConfigType(strings.Trim(ext, "."))
		v.SetConfigName(name)
		s := v.AllSettings()
		if len(s) > 0 {
			cfg.Set(name, s)
		}
	})
}
```

然后在`Init()`方法的 for 循环中，读取完配置信息之后调用`configChange(viper.GetViper())`。
