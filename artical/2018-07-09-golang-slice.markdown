---
layout:     post
title:      "Golang slice 开始结束下标总结"
subtitle:   "Golang"
date:       2018-07-09
author:     "ZhangZhe"
tags:
    - Golang
    - slice
---

#Golang slice 示例
``` golang
s := []int{1,2,3,4,5,6,7,8,9}
b1 := s[0:3]
b2 := s[0:]
b3 := s[:6]
b4 := s[3:len(s)]
```
总结如下：
* 两个下标均可以省略，默认为0和slice长度
* slice[a:b]表示数组下标[a,b)，即包含开始下标，不包含结束下标。
* 所以，需要截取a下标开始，长度为len的slice，语法：slice[a:a+len]
* slice[a:b]截取的slice长度:b-a
* 第三个参数可选，为截取后slice的cap长度。