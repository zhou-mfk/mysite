---
title: "使用picGo+GitHub制作图床"
date: 2019-10-17T20:56:45+08:00
draft: false
tags: ["hugo", "github", "picGo"]
categories: ["Hugo"]
keywords: ["Hugo", "git", "github", "picGo", "Typora"]
isCJKLanguage: true
---

使用picGo+GitHub制作图床

## PicGo介绍

[官方站点]( https://molunerfinn.com/PicGo/)

下载及安装 参考[文档]( https://picgo.github.io/PicGo-Doc/zh/guide/ )

## 使用GitHub

- github上创建一个仓库

  ![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017215146.png)

- 设置Token

  个人主页 Settings

  ![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017215246.png)

  Developer settings --> Generate token

  ![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017215453.png)

  点击创建token就显示出当前的token 请记牢，因为就显示一次

  

## 配置PicGo + GitHub

打开picGo的GitHub图床的配置如下：

![github图床](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017214236.png)

存储路径即在仓库里创建的目录，

自定义域名，使用github 使用如下的模式:

```
https://raw.githubusercontent.com/github用户名/仓库名/分支名称
```

## PIcGo 结合Typora 

写文档特别是markdown的我是使用的Typora的，这两个工具结合起来，更为顺手

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017220349.png)

设置快捷为 Ctrl + Shift + C

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191017220455.png)

截图快捷键 + 上传快捷键，就可以把链接复制到Typora即可使用，这很快捷！

## 参考

[PicGo+GitHub图床，让Markdown飞]( https://juejin.im/entry/5c4ec5aaf265da614420689f )

> 本文为原创文章，转载注明出处 
>
> 如果帮到你了，还请多支持




