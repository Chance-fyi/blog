---
title: "Dockerfile编写注意事项与技巧"
date: 2022-01-26T09:08:11+08:00
draft: false
categories: ["Docker"]
tags: ["Docker","Dockerfile"]

---

### 1、多条指令应换行

```dockerfile
...
RUN apt-get clean apt-get update
...
```

多条指令写在一行会当做一条指令执行，有可能后面的指令不执行或者报错。

应改为：

```dockerfile
...
RUN apt-get clean \
	&& apt-get update
...
```

### 2、忽略错误继续运行

我们在使用`apt-get`安装一个包时，常常会因为缺少依赖而安装失败，我们可以使用`apt-get install -y -f --fix-missing`命令来安装上一次安装失败所需要的依赖包，可以很方便的管理所以依赖，而不用我们手动按照依赖顺序把所有依赖包安装一遍，但是必须在安装失败后执行。Dockerfile 在构建过程中如果出现报错会立即退出构建，我们可以使用`逻辑或||`来忽略错误继续执行后面的语句。

```dockerfile
...
# 下载Chrome安装包
RUN wget -P /tmp https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    # 安装Chrome失败
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb \
    # 安装Chrome的依赖
    && apt-get install -y -f --fix-missing \
    # 再次安装Chrome
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb 
...
```

上面的写法在 Chrome 安装失败时构建会退出，改为下面的写法就可以成功构建了。

```dockerfile
...
# 下载Chrome安装包
RUN 
    wget -P /tmp https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    # 安装Chrome失败之后安装Chrome的依赖
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb || apt-get install -y -f --fix-missing \
    # 再次安装Chrome
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb 
...
```

