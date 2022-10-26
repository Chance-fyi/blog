---
title: "Casbin的Model详解与使用"
date: 2022-10-25T16:06:11+08:00
draft: false
categories: ["Go"]
tags: ["Go", "Casbin", "RBAC", "权限管理"]
---

## 简介

文档：[https://casbin.org/docs/zh-CN/overview](https://casbin.org/docs/zh-CN/overview)

Casbin 是干什么的文档说的很详细，就不在重复说了。相信很多人在看完文档之后，仍然是不知道 `model.conf` 和 `policy.csv` 是什么，看不懂里面的内容，有一种无从下手的感觉。

知识是有一个知识壁垒的，在你打破这个壁垒之前，你觉得很难，查资料看文章也是云里雾里的。但是在打破壁垒之后，掌握了这个知识在回头看，感觉真的好简单。

我查看了好几篇讲 Casbin 的文章，讲的都不是很通俗易懂，对于掌握了 Casbin 的人来看没什么，但是对于还不会的人看了还是有些云里雾里。所以我放弃了去搜索查看别人的经验，转而去读官方文档，在反反复复读了几遍文档之后终于掌握了 Casbin。不得不说在去学习一个新东西的时候，最好的办法就是去查看官方文档。接下来我会以不会 Casbin 的人的角度来尽可能通俗易懂的讲一讲 Casbin。

## Model 与 Policy

接下来我们以 Linux 的文件系统权限为例来讲解 `model.conf` 和 `policy.csv` 。

### 语法

文档：[https://casbin.org/docs/zh-CN/syntax-for-models](https://casbin.org/docs/zh-CN/syntax-for-models)

#### Request 定义

不管哪个语言的 Casbin 都会提供一个 `enforce` 函数方法来校验是否有权限，`[request_definition]` 部分就定义了 `enforce` 函数的参数。

```conf
[request_definition]
r = sub, obj, act
```

`sub, obj, act` 经典三元组: 访问实体 (Subject)，访问资源 (Object) 和访问方法 (Action)，是可以自定义的。表示 `enforce` 函数需要按此顺序传入三个参数。

在 Linux 文件系统中：

- sub：代表用户
- obj：代表访问的文件或文件夹
- act：代表读、写或执行权限

在 RESTful 风格的 API 中：

- sub：代表请求用户
- obj：代表请求路由 例如：`/api/user`
- act：代表请求方法 GET、POST、PUT、DELETE

#### Policy 定义

`[policy_definition]` 部分是对 policy 的定义，对策略权限的规则定义，可以自定义多个规则。

```conf
[policy_definition]
p = sub, obj, act
p2 = sub, act
```

上面这个 model 配置表示定义了两条策略权限的规则 `p` 和 `p2` ，等号前面的 `p` 和 `p2` 表示规则的名称，等号后面则是规则的内容。

`policy.csv` 文件或数据库中的一行行则代表了一条策略权限的规则或下面讲到的角色规则。第一列代表规则名，后面几列依次代表规则内容。

`user` 用户拥有 `/home/file` 的读（r）权限就可以这样表示：

```csv
p, user, /home/file, r
```

#### Policy effect 定义

`[policy_effect]` 是策略效果的定义，目前只支持内置的 5 种规则，它是按整个字符串匹配的，所以一个字符都不能错。一般这个默认的就可以满足业务要求，不需要改动。具体 5 种规则的效果可以等全部掌握 Casbin 之后自行尝试一下。

如果需要改动这个使用其他的策略效果，则 `[policy_definition]` 的定义中还需要增加一个 `eft`，在 `policy.csv` 文件对应的值只能是 `allow` 和 `deny`。

- some(where (p.eft == allow))
- !some(where (p.eft == deny))
- some(where (p.eft == allow)) && !some(where (p.eft == deny))
- priority(p.eft) || deny
- subjectPriority(p.eft)

#### 匹配器

`[matchers]` 定义了匹配的规则，说明了我们通过 `enforce` 函数传入的参数与定义的策略权限规则怎么才算是匹配成功。

```conf
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

上述匹配器说明我们通过 `enforce` 函数传入的参数与定义的策略权限规则应一一对应才算是匹配成功。

接下来我们看一下官网给的一些 Models 的例子。

### ACL

ACL：`Access Control List` ，它的核心主要在于用户和权限直接挂钩。

**model.conf**

```conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

**policy.csv**

```csv
p, user1, /home/file, r
p, user1, /home/file, w
p, user2, /home/file, x
```

**效果**

```go
# 首先解读一下 policy.csv
#
# 用户 user1 对文件 /home/file 有读 r 权限
# p, user1, /home/file, r
#
# 用户 user1 对文件 /home/file 有写 w 权限
# p, user1, /home/file, w
#
# 用户 user2 对文件 /home/file 有执行 x 权限
# p, user2, /home/file, x

# 使用 enforce 函数来检查权限
enforce("user1", "/home/file", "r") // => true
enforce("user1", "/home/file", "w") // => true
enforce("user1", "/home/file", "x") // => false
enforce("user2", "/home/file", "x") // => true
```

### RBAC

RBAC：Role Based Access Control，基于角色的访问控制系统。权限是与角色进行相关联，用户通过成为适当的角色从而得到这些角色的权限。

**model.conf**

```conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

model 使用 RBAC, 还需要添加 `[role_definition]` 部分。

**policy.csv**

```csv
p, user_group1, /home/file, r
p, user_group1, /home/file, w
p, user_group2, /home/file, x

g, user1, user_group1
g, user2, user_group1
g, user3, user_group2
```

**效果**

```go
# 首先解读一下 policy.csv
#
# 用户组 user_group1 对文件 /home/file 有读 r 权限
# p, user_group1, /home/file, r
#
# 用户组 user_group1 对文件 /home/file 有写 w 权限
# p, user_group1, /home/file, w
#
# 用户组 user_group2 对文件 /home/file 有执行 x 权限
# p, user_group2, /home/file, x

# 给用户 user1 分配 用户组 user_group1
# g, user1, user_group1
#
# 给用户 user2 分配 用户组 user_group1
# g, user2, user_group1
#
# 给用户 user3 分配 用户组 user_group2
# g, user3, user_group2

# 使用 enforce 函数来检查权限
enforce("user1", "/home/file", "r") // => true
enforce("user1", "/home/file", "w") // => true
enforce("user1", "/home/file", "x") // => false
enforce("user2", "/home/file", "r") // => true
enforce("user2", "/home/file", "w") // => true
enforce("user2", "/home/file", "x") // => false
enforce("user3", "/home/file", "x") // => true
```
