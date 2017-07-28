---
layout:     post
title:      "Java log4j2 配置相关"
subtitle:   "Java log4j2 配置相关"
date:       2017-05-05
author:     "ZhangZhe"
tags:
    - Java
---

# java新日志系统log4j2配置相关
根据代码运行情况,目前得到的信息是如果使用java api提供的接口:
```LoggerFactory.getLogger()
```
返回的log对象,是从全局的一个LoggerFactory中实例出来的对象,相应的配置信息会从全局的LoggerContext中获取(搭载使用log4j2日志系统).

如果需要修改这些配置,则需要按照如下方式,先获取到全局的LoggerContext,然后使用相应的接口进行配置的rewrite.

How do I reconfigure log4j2 in code with a specific configuration file?

See the below example. Be aware that this LoggerContext class is not part of the public API so your code may break with any minor release.
[来源,官网FAQ](http://logging.apache.org/log4j/2.x/faq.html#reconfig_from_code)
```
// import org.apache.logging.log4j.core.LoggerContext;
 
LoggerContext context = (org.apache.logging.log4j.core.LoggerContext) LogManager.getContext(false);
File file = new File("path/to/a/different/log4j2.xml");
// this will force a reconfiguration
context.setConfigLocation(file.toURI());
```
