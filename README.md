# realSinged.github.io

## 环境配置

* HUGO 安装


```
    macos:

    brew install hugo

    linux:

    snao install hugo
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

每篇文章开始需要包含一个标题，形如：

```
---
title: "Go语言调度（一）：系统调度器"
date: 2021-02-01T16:01:24+08:00
draft: false
---
```

标题以外的内容和所有markdown语法一致。

## 本地检查

执行hugo server 
## 发布
