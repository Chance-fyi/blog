---
title: "Go标准库time.Parse方法的时区问题"
date: 2022-03-30T13:37:03+08:00
draft: false
categories: ["Go"]
tags: ["Go", "time"]
---

`time.Parse`方法解析时间字符串得到的 time 的时区为`UTC`，并不会自动使用系统的默认时区。

```go
parse, _ := time.Parse("2006-01-02 15:04:05", "2022-03-30 13:37:03")
//UTC 0
fmt.Println(parse.Zone())
//CST 28800
fmt.Println(time.Now().Zone())
```

如果我们不想要`UTC`时区，需要别的时区。可以使用`time.ParseInLocation`方法自行设置时区。

```go
loc, _ := time.LoadLocation("Asia/Shanghai")
parse, _ := time.ParseInLocation("2006-01-02 15:04:05", "2022-03-30 13:37:03", loc)
//CST 28800
fmt.Println(parse.Zone())
//CST 28800
fmt.Println(time.Now().Zone())
```
