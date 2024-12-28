# realSinged.github.io

## 环境配置

* HUGO 安装


```
    macos:

    brew install hugo

    linux:

    snap install hugo
```

当前使用的版本是： hugo v0.87.0


* themes submodule

当前使用的主题是even，拉取submodule

```
cd themes/even
git submodule init && git submodule update  
```

## 创作

目录是`content/post`, 使用markdown进行创作。

每篇文章开始需要包含一个front matter，形如：

```
---
title: "Go语言调度（一）：系统调度器"
date: 2021-02-01T16:01:24+08:00
draft: false
---
```

以下是一些常用的front matter:

* date: 内容创作时间
* description: 内容的简短描述
* draft: 如果设置为true，则不会渲染这篇内容
* keywords: 描述内容的一些关键字
* title: 内容标题


所有的hugo front matter见[链接](https://gohugo.io/content-management/front-matter/)


标题以外的内容和所有markdown语法一致。

## 本地预览

终端执行 

```
hugo server
```

默认会在1313端口创建一个web server， 在浏览器中输入http://localhost:1313/, 就可以实时看到改动及效果。

执行hugo server 
## 发布

文章创作完成，检查无误后，终端执行

```
hugo --destination .
```
即可生成静态内容。

然后git提交，并push到远端。

访问realSinged.github.io，即可看到更新的创作内容。
