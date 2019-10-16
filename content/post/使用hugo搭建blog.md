---
title: "使用hugo搭建blog"
date: 2019-10-11T21:44:22+08:00
draft: false
tags: ["hugo", "git"]
categories: ["学习"]
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
git submodule add https://github.com/halogenica/beautifulhugo.git beautifulhugo
# 使用主题
cp themes/beautifulhugo/exampleSite/config.toml mysite/

```

config.toml配置文件修改如下

```yaml
baseurl = "https://zhou-mfk.github.io"  # 你的站点地址
DefaultContentLanguage = "en"  # 默认的内容语言
title = "Zhou Li Shan"  # 站点名称
theme = "beautifulhugo"  # 站点主题名称
metaDataFormat = "yaml"   #元数据类型
pygmentsStyle = "dracula" # 主题的风格
pygmentsUseClasses = false 
pygmentsCodeFences = true
pygmentsCodefencesGuessSyntax = true
pygmentsUseClassic = false

paginate = 12
autoFigure = true
#pygmentOptions = "linenos=inline"
#disqusShortname = "XXX"
#googleAnalytics = "XXX"

[Params]
#  homeTitle = "Beautiful Hugo Theme" # Set a different text for the header on the home page
  subtitle = "无可奈何花落去"  # 副标题
  mainSections = ["post","posts"] # 主入口目录
  logo = "img/avatar-icon.png" # logog图片
  favicon = "img/favicon.ico" 
  dateFormat = "2019-10-10" # 日期格式
  commit = false
  rss = true
  comments = true
  readingTime = true
  wordCount = true
  useHLJS = true
  socialShare = true
  delayDisqus = true
  showRelatedPosts = true
#  hideAuthor = true
#  gcse = "012345678901234567890:abcdefghijk" # Get your code from google.com/cse. Make sure to go to "Look and Feel" and change Layout to "Full Width" and Theme to "Classic"
# 下面是首页的轮播图
# [[Params.bigimg]]
#   src = "img/triangle.jpg"
#   desc = "Triangle"
# [[Params.bigimg]]
#   src = "img/sphere.jpg"
#   desc = "Sphere"
#   # position: see values of CSS background-position.
#   position = "center top"
# [[Params.bigimg]]
#   src = "img/hexagon.jpg"
#   desc = "Hexagon"
# 
# 作者的信息
[Author]
  name = "Zhou Li Shan"
  website = "zhou-mfk.github.io"
  email = "zhou_mfk@163.com"
# facebook = "username"
  github = "zhou-mfk"
# gitlab = "username"
# bitbucket = "username"
# twitter = "username"
# reddit = "username"
# linkedin = "username"
# xing = "username"
# stackoverflow = "users/XXXXXXX/username"
# snapchat = "username"
# instagram = "username"
# youtube = "user/username" # or channel/channelname
# soundcloud = "username"
# spotify = "username"
# bandcamp = "username"
# itchio = "username"
# vk = "username"
# paypal = "username"
# telegram = "username"
# 500px = "username"
# codepen = "username"
# mastodon = "url"
# kaggle = "username"
# weibo = "username"
# slack = "username"
#
# 页面的菜单配置
[[menu.main]]
    name = "HOME"
    url = ""
    weight = 1

[[menu.main]]
    name = "关于"
    url = "page/about/"
    weight = 3

[[menu.main]]
    identifier = "categories"
    name = "文章分类"
    weight = 2

[[menu.main]]
    parent = "categories"
    name = "分类"
    url = "categories/"
    weight = 1

[[menu.main]]
    name = "Tags"
    url = "tags"
    weight = 3

```

