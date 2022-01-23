---
title: "从零完成一个带滑动验证的自动登陆"
date: 2022-01-21T14:02:22+08:00
draft: true
categories: ["Python"]
tags: ["Python","Selenium","爬虫","自动登录","滑动验证码"]
---

## 前言

最近帮女朋友写了一个监控九价订阅的小程序，对方接口需要登陆，Token 有效时间只有大概 2 个小时，每次都要手动去更新登陆状态，太过麻烦了。所以就想要解决一下自动登陆的问题，登陆有滑动验证码，查找了一些资料决定使用 **Python** + **Selenium** 来解决，可惜我也不会 Python ，只能一点点摸索了，顺便记录下来。

## 了解Python

因为以前没有使用过 Python ，所以先了解一下 Python 的基本语法。

+ [菜鸟教程](https://www.runoob.com/Python/Python-tutorial.html)
+ [廖雪峰的 Python 教程](https://www.liaoxuefeng.com/wiki/1016959663602400)

## 配置环境

了解完基础之后先安装一个环境吧，不想在电脑上在安装 Python ，而且完成之后肯定要部署服务器，所以还是使用 Docker 来统一环境。

### 安装 Python

先简单写一个 **docker-compose.yml** 的文件，拉取一个 Python镜像并创建容器。

**docker-compose.yml**

```yaml
version: "3.7"
services:
  Python:
    image: Python
    volumes:
      - .:/home
    working_dir: /home
    tty: true
```

使用 **PyCharm** 可以很方便的管理 Docker ，运行 docker-compose.yml 并进入容器。

![image-20220122090750540](https://image.chance.fyi/image-20220122090750540.png)

![image-20220122090813000](https://image.chance.fyi/image-20220122090813000.png)

### 安装 Chrome

打开 Chrome [官网](https://www.google.cn/chrome/)，滑动到最下面，点击**其他平台**，然后在弹出窗口选择 **Linux**，选择适用系统的安装包，获取下载[链接](https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb)，然后进入容器执行以下命令安装 Chrome 。

```bash
# 更换apt源
$ sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list
# 更新依赖
$ apt-get clean
$ apt-get update
# 下载Chrome安装包
$ wget -P /tmp https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
# 安装Chrome失败
$ dpkg -i /tmp/google-chrome-stable_current_amd64.deb 
# 安装Chrome的依赖
$ apt-get install -y -f --fix-missing
# 再次安装Chrome
$ dpkg -i /tmp/google-chrome-stable_current_amd64.deb 
```

### 安装 ChromeDriver

[https://chromedriver.chromium.org/](https://chromedriver.chromium.org/)

```bash
# 下载ChromeDriver
$ wget -P /tmp https://chromedriver.storage.googleapis.com/97.0.4692.71/chromedriver_linux64.zip
# 解压
$ unzip -d /usr/bin/ /tmp/chromedriver_linux64.zip 
```

### 安装 Selenium

```bash
pip install selenium
```

### 测试环境

编写一个 Python 脚本测试环境是否安装成功。

```Python
from selenium import webdriver

option = webdriver.ChromeOptions()
# 无界面模式运行
option.add_argument('headless')
option.add_argument('no-sandbox')
driver = webdriver.Chrome(chrome_options=option)

driver.get('http://www.baidu.com/')
print(driver.title)
driver.quit()
```

运行输出 **百度一下，你就知道** 成功获取到百度网页的标题，说明环境安装成功。如果输出乱码可以查看这篇[文章](/post/tool/ide/docker-terminal-chinese-garbled-code/)。

![image-20220121172834421](https://image.chance.fyi/image-20220121172834421.png)

### Dockerfile

环境安装成功之后整理成 Dockerfile ，因为 Dockerfile 构建时产生错误会直接退出构建，我们上面的安装 Chrome 时是有报错的，我们可以使用`逻辑或||`来忽略错误继续执行后面的语句。

```dockerfile
FROM python

ARG aly=https://mirrors.aliyun.com/pypi/simple/

# 更换apt源
RUN sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list \
    # 更新依赖
    && apt-get clean \
    && apt-get update \
    # 下载Chrome安装包
    && wget -P /tmp https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    # 安装Chrome失败之后安装Chrome的依赖
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb || apt-get install -y -f --fix-missing \
    # 再次安装Chrome
    && dpkg -i /tmp/google-chrome-stable_current_amd64.deb \
    # 下载ChromeDriver
    && wget -P /tmp https://chromedriver.storage.googleapis.com/97.0.4692.71/chromedriver_linux64.zip \
    # 解压
    && unzip -d /usr/bin/ /tmp/chromedriver_linux64.zip \
    # 安装Python包
    && pip install selenium -i ${aly}
```

## 调试

因为使用了无界面的方式，对页面的一些操作，比如输入账号密码，到底是否成功，我们是不得而知的。这时候我们可以使用截图的方式来查看当前的页面效果。

```python
driver.save_screenshot('./1.png')
```

当然也可以直接使用有界面的方式，具体如何请自行查找资料，我也不会。

### 页面乱码

截图显示的页面中文可能是一个个框框，这个是因为 Linux 没有中文字体的原因，因为不影响我要完成的功能，暂时不管了。
