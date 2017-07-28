---
layout:     post
title:      "Jekyll 配置说明"
subtitle:   "Jekyll url相关配置"
date:       2017-03-14
author:     "ZhangZhe"
header-img: ""
tags:
    - Jekyll
---


# Jekyll 配置说明

## URL配置

默认情况，Jekyll会使用 page1/index.html的风格，为根目录下的所有页面自动创建相的目录以及静态文件。
在页面源文件的头信息中，使用permalink参数，能够灵活的配置相应的页面URL，permalink=/404.html，则会直接生成404.html并能直接访问。