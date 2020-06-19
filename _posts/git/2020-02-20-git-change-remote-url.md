---
layout: post
title: git 仓库迁移更改源
categories: Git
description: git 仓库迁移，git remote 更改源
keywords: git, 仓库迁移, 更改源
---

ip 变了，git 远程仓库的地址也变了，这个时候就要自己手动修改项目连接到的远程仓库地址



## 首先查看你的remote的地址
```
git remote -vv
```

remote 信息如下 

```
(base) hongjunjiadeMacBook-Pro:rentcar hongjunjia$ git remote -vv
origin  http://gitlab.10101111.com:8888/zeus/XXX.git (fetch)
origin  http://gitlab.10101111.com:8888/zeus/XXX.git (push)
```

origin 是自己远程仓库的分支


## 方案 一 先删除后增加

删除远程仓库源地址
```
git remote rm origin  // 删除远程仓库源地址
```

添加远程git分支
```
git remote add origin [需要更换远程仓库的git地址] // 添加分支
```

## 方案 二 直接修改
```
git remote set-url origin xxxxx.git
```

