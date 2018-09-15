---
title: hexo更新文章
date: 2018-09-15 10:51:21
tags:
---

## 新建文章

在MyBlog目录下执行：hexo new “文章标题”，会在source->_posts文件夹内生成一个.md文件。
编辑该文件（遵循Markdown规则）
修改起始字段
- title 文章的标题
- date 创建日期 （文件的创建日期 ）
- updated 修改日期 （ 文件的修改日期）
- comments 是否开启评论 true
- tags 标签
- categories 分类
- permalink url中的名字（文件名）
- 编写正文内容（MakeDown）

## 更新文章

```
hexo clean 删除本地静态文件（Public目录），可不执行。
hexo g 生成本地静态文件（Public目录）
hexo deploy 将本地静态文件推送至github（hexo d）

```
