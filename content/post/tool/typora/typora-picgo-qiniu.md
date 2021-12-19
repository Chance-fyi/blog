---
title: "Typora使用Picgo图片上传七牛云的配置"
date: 2021-11-25T16:47:08+08:00
draft: false
categories: ["Typora"]
tags: ["Typora"]
---

```json
{
  "picBed": {
    "uploader": "qiniu",
    "qiniu": {
        "accessKey": "****************************************",
        "secretKey": "****************************************",
        "bucket": "chance-img",// 存储空间名
        "url": "http://image.chance.fyi",// 自定义域名
        "area": "z2",// 存储区域编号 z0华东 z1华北 z2华南 na0北美 as0东南亚
        "options": "",// 网址后缀，比如?imgslim
        "path": "" // 自定义存储路径，比如img/  经测试只是在文件名上添加了前缀 并不是文件夹路径
    }
  },
  "picgoPlugins": {}
}
```

