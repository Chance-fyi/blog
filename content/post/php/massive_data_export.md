---
title: "Massive_data_export"
date: 2022-05-19T19:43:58+08:00
draft: true
categories: []
tags: []
---

### 背景

公司业务部门经常会需要导出各种数据进行分析，因为需要的数据模板多变，产品一直没有将其规划成系统功能，每次需要的数据都是手写 SQL 语句进行导出。在写了上万行 SQL 语句以后，我对这种体力劳动感觉到了厌烦，于是和领导申请，自己利用空闲时间规划开发了一个灵活配置、适用性强的导表系统。

### 使用技术

+ API 框架 [Hyperf](https://www.hyperf.wiki/2.2/#/zh-cn/)
+ Web 框架 [Ant Design of React](https://ant.design/docs/react/introduce-cn)
+ Excel 扩展 [xlsWriter](https://xlswriter.viest.me/)

### 优化

为了尽快实现功能，导出 Excel 功能是做的同步导出，但在我的设计中应该是异步导出，因此在后续的优化中引入了 RabbitMQ 消息队列实现异步导出功能。

