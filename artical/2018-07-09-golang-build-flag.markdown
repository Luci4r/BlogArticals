---
layout:     post
title:      "Golang 条件编译"
subtitle:   "Golang"
date:       2018-07-09
author:     "ZhangZhe"
tags:
    - Golang
    - build flag
---

# golang条件编译
```golang
// +build !race
```
在源码前加特殊格式的注释，可以控制条件编译

# 函数前加注释，可以制定编译后链接的包名与函数名