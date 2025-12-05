---
title: "使用ChatGPT辅助开发鼠标手势扩展"
date: 2024-06-04T14:45:20+08:00
draft: false
categories: ["chrome-extension"]
tags: ["chrome-extension", "ChatGPT"]
---

## 前言

我一直在使用的鼠标手势插件 SmartUp Gestures 最近被 Chrome 提示包含恶意软件，导致无法继续使用了。作为一个鼠标手势的重度使用者，我不仅在日常操作中频繁依赖它，有时甚至会在无聊时用鼠标手势画画。尝试了其他几个替代的手势插件后，我发现它们都不是很好用。因此，我决定自己开发一个新的鼠标手势插件。

## 学习插件开发

由于以前没有开发过 Chrome 插件，我决定先了解一下开发流程。在查找资料时，我发现了一篇名为[《chrome 插件开发指南（Manifest V3）》](https://juejin.cn/post/7173567493871501325)的文章，这篇文章简明扼要地介绍了开发流程。同时，我在其中找到了一个推荐的插件开发模板[chibat/chrome-extension-typescript-starter](https://github.com/chibat/chrome-extension-typescript-starter)。

下载了这个模板项目，然后运行起来。在 Chrome 的扩展程序管理页面加载 dist 目录后，便能看到插件的效果了。

![picture 0](/images/image-2024060417311406031.png)

![picture 1](/images/image-2024060417403154799.png)

在阅读上述文章和模板代码后，我了解到 `manifest.json` 是插件的配置文件。接下来，将文件内容发送给 GPT，让其为我解释其中的各项配置。

![picture 2](/images/image-2024060419501726244.png)

结合模板代码进行调试后，我大致了解了各个页面的作用。

## 插件开发

### 绘制鼠标滑动轨迹

不太会写这些逻辑，还是先问一下 GPT，我一般写的 prompt 都很简单，然后在遇到问题时再进行调整。

![picture 3](/images/image-2024060420424782582.png)

![picture 4](/images/image-2024060420420076315.png)

将代码复制进去进行测试，发现可以成功绘制鼠标轨迹。不过，代码中仍存在一些 Bug，需要进一步调试和修改。

![picture 5](/images/image-2024060420511084540.png)

#### 1. 第一个 Bug 是右键松开了还能绘制轨迹

猜测 Bug 原因是没有取消事件监听的原因。

![picture 6](/images/image-2024060422013918431.png)

问了 GPT 发现在乱说，经过进一步调试，发现不是这个原因。经过细致的调试，我发现 `handleMousedown` 中的代码在鼠标按下并放开之后才执行。

![picture 7](/images/image-202406042205338448.png)

GPT 推荐我使用 `mousedown` 事件。更换事件监听后，这个 Bug 得到了解决。之后，我又自己添加了在鼠标右键按下并移动时阻止右键菜单弹出的逻辑。

#### 2. 第二个 Bug 是鼠标左键也能绘制轨迹

![picture 8](/images/image-2024060422103263980.png)

加上是否是右键按下的判断也顺利解决。

#### 3. 优化 鼠标绘制的轨迹不顺滑

![picture 9](/images/image-2024060422135425042.png)

GPT 推荐使用贝塞尔曲线来平滑轨迹。添加这段逻辑后，效果达到了预期。

#### 4. 第三个 Bug 是页面滚动之后轨迹不在跟随鼠标

猜测问题出在元素定位上。由于我对 CSS 不太熟悉，写得也不多，所以又咨询了 GPT 🤣。

![picture 10](/images/image-2024060422193823498.png)

最后的代码实现：

```typescript
// 是否阻止右键菜单
let blockMenu: boolean = false;
document.addEventListener("mousedown", (e: MouseEvent) => {
  // 判断是否是右键按下
  if (e.button != 2) {
    return;
  }
  blockMenu = false;
  // 鼠标初始位置
  let clientX: number = e.clientX;
  let clientY: number = e.clientY;

  const canvas = document.createElement("canvas");
  canvas.style.position = "fixed";
  canvas.style.top = "0";
  canvas.style.left = "0";
  canvas.style.zIndex = "999999";
  canvas.style.pointerEvents = "none"; // 点击穿透
  canvas.width = window.innerWidth - 1; // -1 防止出现滚动条
  canvas.height = window.innerHeight - 1;
  document.body.appendChild(canvas);
  const ctx = canvas.getContext("2d");

  if (!ctx) return;
  ctx.strokeStyle = "#0072f3";
  ctx.lineWidth = 2;

  let lastX: number = e.clientX;
  let lastY: number = e.clientY;
  const handleMouseMove = (moveEvent: MouseEvent) => {
    const currentX: number = moveEvent.clientX;
    const currentY: number = moveEvent.clientY;

    ctx.beginPath();
    ctx.moveTo(lastX, lastY);
    // 使用贝塞尔曲线来平滑轨迹
    ctx.quadraticCurveTo(
      (lastX + currentX) / 2,
      (lastY + currentY) / 2,
      currentX,
      currentY
    );
    ctx.stroke();

    lastX = currentX;
    lastY = currentY;
  };

  const handleMouseUp = (e: MouseEvent) => {
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
    document.body.removeChild(canvas);
    if (clientX != e.clientX || clientY != e.clientY) {
      // 鼠标移动阻止右键菜单
      blockMenu = true;
    }
  };

  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", handleMouseUp);
});
document.addEventListener("contextmenu", (e: MouseEvent) => {
  if (blockMenu) {
    // 阻止右键菜单
    e.preventDefault();
  }
});
```

### 判断轨迹方向

轨迹方向检测算法是一个难点。我多次调整 prompt，最终得到了一个相对可用的代码。不过，这个算法仍有一些可以优化的地方。

![picture 11](/images/image-2024060612252476926.png)

至此，两个最主要的功能已经实现了。一边开发一边记录博客有些拖慢了进度，所以博客暂时记录到这里。后续，我会继续完善插件的完整功能。
