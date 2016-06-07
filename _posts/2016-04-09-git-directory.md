---
layout: post
title: "git directory"
description: "Git 目录结构"
category: git
---
{% include JB/setup %}

**.git目录下的重要文件**

* config

  主要包含：远程主机信息、分支信息；用于Git配置，描述远程主机地址、分支以及子模块信息

* index

  对应于Git的暂存区的二进制文件

* HEAD

  文件内容一般为 `ref: refs/heads/master`，对应当前本地分支（或概念模型中的版本库）

**.git目录下的重要目录**

* refs/

  该目录有三个子目录：

  1. **heads** : 本地分支指针
  2. **remotes** : 远程分支指针
  3. **tags** ：分支相关

* objects/

  实际的 Git 对象库

* logs/

  日志文件
