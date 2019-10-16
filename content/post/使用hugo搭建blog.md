---
title: "使用hugo搭建blog"
date: 2019-10-11T21:44:22+08:00
draft: false
tags: ["hugo", "git"]
categories: ["Hugo"]
keywords: ["Hugo", "git", "github"]
isCJKLanguage: true
---

hugo 的使用到发布到github

<!--more-->

## 安装hugo

参考[官方文档](https://gohugo.io/getting-started/quick-start/)

安装完成后创建站点

```shell
hugo new site you_site_name # 会创建you_sit_name这个点目录
# 目录结构如下
cd you_site_name
ls
archetypes/  config.toml  content/  LICENSE  README.md  resources/  themes/
# 其中themes目录是存放主题的目录，没有主题，hugo启动后是无法访问的，所以要去安装主题
```

## hugo 主题配置

可以到官方站点去下载你喜欢的主题 [主题](https://themes.gohugo.io/)

本博客使用的是: beautifulhugo

主题下载到themes中即可

```shell
git submodule add https://github.com/halogenica/beautifulhugo.git beautifulhugo
```

## 发布到github 并使用github的GitPage

下面开始初始化hugo并使用github来做为站点的全部步骤

```shell
# 先创建一个目录存放hugo的blog 
mkdir hugo-blog
# 创建站点
cd hugo-blog
hugo new site mysite
# 下载主题
cd mysite/themes
git submodule add https://github.com/olOwOlo/hugo-theme-even.git even
# 使用even主题, 此主题比较友好，并有中文说明。
cp themes/beautifulhugo/exampleSite/config.toml mysite/
```

hugo 常用命令

```shell
hogo # 生成静态html文件 此时站点目录会有一个public html主题目录
hogo server # 会在本地启动一个服务，默认为1313端口，用来在本地进行测试或查看
hogo server -d # 会把草稿类的.md生成静态html 并在本地进行查看
# 其他命令需要自己查看了
hogo --help 
```

发布到github

```shell
cd mysite
git init
# 添加

```



