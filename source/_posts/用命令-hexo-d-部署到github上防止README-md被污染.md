---
title: 用命令 hexo d 部署到github上防止README.md被污染
date: 2016-03-19 19:27:04
tags: Hexo
categories: Hexo
---

#### 问题描述

如果在github上新建一个github pages时忘了添加README.md文件，这时直接在github上面新建一个README.md文件是不行的，因为每次用命令hexo d将本地提交到github时，README.md文件都会消失。

#### 解决方法

1.在hexo博客根站点下的source文件夹建立README.md文件。
2.在站点配置文件_config.yml中找到`skip_render`参数，在其值上添加 README.md，如下：
```yml```
skip_render: README.md
```
保存退出，重新部署即可。
