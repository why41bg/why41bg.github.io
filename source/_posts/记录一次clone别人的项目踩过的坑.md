---
title: 记录一次clone别人的项目踩过的坑
date: 2023-09-21 00:59:14
tags: git
---

在使用 `github page` 搭建博客的时候，打算使用 `butterfly` 这个主题，于是就把它的源码 `git clone` 了下来。结果 `github action` 的时候，一直生成空白的页面。
经过仔细检查发现，生成的静态文件**全都为空**。结果查看 `git clone` 下来的 `butterfly` 源码发现，在 github 仓库中，butterfly 的文件夹图标变成了一个**向右的箭头**。
查阅了一番资料后了解到。这是因为直接使用`git clone`获得到源码中有`.git`隐藏文件，在 push 到自己的 github 仓库的时候，github 将他视为了一个子系统模块。
解决办法如下：

1. 删除文件夹里面的 `.git` 文件夹

2. 然后依次执行：
   ```
   git rm --cached [文件夹名]
   git add .
   git commit -m '[描述文字]'
   git push origin -u [branch_name]
   ```

