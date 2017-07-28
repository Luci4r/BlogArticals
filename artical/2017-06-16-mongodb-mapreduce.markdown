---
layout:     post
title:      "MongoDB mapreduce 调试"
subtitle:   "大数据相关"
date:       2017-06-16
author:     "ZhangZhe"
tags:
    - MongoDB
---

#MongoDB
相关函数用与调试方法。
```
var c = db.UIDs.findOne()

var m = function(){
	 print(this)
	this.DS.forEach(function(z){
		print(z.D,z.C);
		emit(z.D,z.C);
	});
}

var emit = function(key, value) {
	print("emit");
	print("key: " + key + " value: " + tojson(value));
}

m.apply(c)//调试

var r = function(key,values){return Array.sum(values);}

db.UIDs.mapReduce(m,r,{limit:1000,out:"null"}).find()
```
引用自：

[跟踪调试](http://www.cnblogs.com/yuechaotian/archive/2013/02/26/2933455.html)

[ MongoDB mapReduce使用 ](http://blog.csdn.net/guoqianqian5812/article/details/52785557)

[ MongoDB mapReduce使用 ](http://blog.csdn.net/qqiabc521/article/details/6330783)