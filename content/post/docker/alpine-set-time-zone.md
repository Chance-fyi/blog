---
title: "alpine镜像设置时区"
date: 2022-03-29T14:53:01+08:00
draft: false
categories: ["Docker"]
tags: ["Docker","Dockerfile","alpine","timezone"]
---

```dockerfile
FROM alpine

ENV TZ=Asia/Shanghai

RUN echo 'http://mirrors.aliyun.com/alpine/v3.4/main/' > /etc/apk/repositories \
    && apk --no-cache add tzdata zeromq \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo '$TZ' > /etc/timezone
```

