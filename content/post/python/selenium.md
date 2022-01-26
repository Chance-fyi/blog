---

title: "从零完成一个带滑动验证的自动登陆"
date: 2022-01-21T14:02:22+08:00
draft: false
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

![image-20220126135243700](https://image.chance.fyi/image-20220126135243700.png)

截图显示的页面中文可能是一个个框框，这个是因为 Linux 没有中文字体的原因，因为不影响我要完成的功能，暂时不管了。

## 写代码

### 获取页面元素

通过`driver.find_element()`方法来获取页面元素，函数有两个参数。

+ by 根据什么来获取元素，可选值有以下几个

  ```
  By.ID = "id"
  By.XPATH = "xpath"
  By.LINK_TEXT = "link text"
  By.PARTIAL_LINK_TEXT = "partial link text"
  By.NAME = "name"
  By.TAG_NAME = "tag name"
  By.CLASS_NAME = "class name"
  By.CSS_SELECTOR = "css selector"
  ```

+ value 与第一个参数所选方式对应的值

### 写入内容

可以使用`send_keys()`对输入框写入一段字符串，例如：

```python
driver.find_element('css selector', 'input[type=text]').send_keys('18888888888')
```

![image-20220126140841063](https://image.chance.fyi/image-20220126140841063.png)

### 点击按钮

可以通过`click()`来实现点击事件，完成确认协议与登陆。

```python
driver.find_element('css selector', '.btn-login').click()
```

![image-20220126142519413](https://image.chance.fyi/image-20220126142519413.png)

### 获取滑动验证码图片

点击登陆之后需要 sleep 等待几秒，等运行完成可以看到滑动验证码已经弹出来了。

我们可以通过`get_attribute()`方法来获取元素的属性，读取 img 标签的 src 属性，例如：

```python
driver.find_element('css selector', '.verify-sub-block img').get_attribute('src')
```

src 获取到的是图片的 base64 数据，可以使用以下代码转换并报存。

```python
imgdata = base64.urlsafe_b64decode(bstr)
file = open(file_path, 'wb')
file.write(imgdata)
file.close()
```

![image-20220126152703974](https://image.chance.fyi/image-20220126152703974.png)![image-20220126152742456](https://image.chance.fyi/image-20220126152742456.png)

### 偏移量计算

查了一些资料，网上有很多破解滑动验证码的案例代码，但是好像都是获取一张无缺口的原图，和一张有缺口的图，然后一个个像素进行对比来计算偏移量。我们这里获取不到原图，只有一张滑块图和有缺口的大图。

#### 实现方法

首先滑块小图被一圈白边包裹，可以通过颜色获取边界坐标，然后使用边界坐标向右平移 X 轴，一个个像素对比，找到缺口的位置，从而计算出偏移量。

### 滑动滑块

得到偏移量之后就可以滑动滑块实验了，如果没有做机器滑动和人为滑动的验证，那么基本就成功了。

```python
# 滑动方法
dragger = driver.find_element('css selector', '.verify-move-block')
action = ActionChains(driver)
action.drag_and_drop_by_offset(dragger, offset, 0).perform()
```

很幸运并没有机器验证。

![image-20220126155404473](https://image.chance.fyi/image-20220126155404473.png)

### 获取Token

通过 Cookie 来获取 Token

```python
driver.get_cookie('_xzkj_')['value']
```

## 问题

### 页面崩溃

脚本运行几次之后，会获取不到元素报错，还有可能页面直接崩溃了。

```
selenium.common.exceptions.WebDriverException: Message: unknown error: session deleted because of page crash
from tab crashed
  (Session info: headless chrome=97.0.4692.99)
```

根据报错信息搜索大多数说是`shm`默认只有64M，太小的原因。

```bash
root@ec9ed055a1bb:/home# df -Th
Filesystem     Type           Size  Used Avail Use% Mounted on
overlay        overlay         63G  6.6G   53G  12% /
tmpfs          tmpfs           64M     0   64M   0% /dev
shm            tmpfs           64M   41M   24M  63% /dev/shm
grpcfuse       fuse.grpcfuse  357G   58G  300G  17% /home
/dev/sda1      ext4            63G  6.6G   53G  12% /etc/hosts
tmpfs          tmpfs          993M     0  993M   0% /proc/acpi
tmpfs          tmpfs          993M     0  993M   0% /sys/firmware
```

使用`df -Th`查看确实只有64M并且已经占用了63%，此时已经没有足够的内存给浏览器启动了，所以就页面崩溃了。可以只要设置大点就可以了吗，我发现容器刚启动时占用是0%的，随着脚本多次调用，占用会逐渐上升，并且只有在脚本异常退出时才会上升，由此我推断是脚本异常退出导致浏览器没有正确关闭，始终占用着内存，导致内存越来越大。所以只有我们正确关闭浏览器，就不会出现内存占用过高，导致下一次执行崩溃的问题了。可以加一个 `try except` 捕获异常然后退出浏览器就可以了。

### 异步问题

感觉这个很多操作都是异步执行的，比如点击登陆之后立即截图，是看不到验证码的，必须要 sleep 几秒运行之后截图，才能看得到。所以如果脚本没有得到想要的结果的时候，可以试着 sleep 一下看看。

### input 输入值丢失

可以看到上面验证码截图，账号内容丢失了。具体是点击登陆，滑动验证码弹出之后丢失了，具体原因不太清楚。解决办法是点击登陆之后再次输入一下账号就可以了，解决问题才是最重要的。

### 页面空白

偶发有打开的页面是一片空白，获取不到元素，直接报错，查了一些资料好像是 Chrome 与 ChromeDriver 版本不同的原因，Chrome 官网上没有找到历史版本下载的地方，ChromeDriver 最新版也不及 Chrome 的版本，暂时不管了，多试几次吧。
