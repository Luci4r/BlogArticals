---
layout:     post
title:      "Storm 配置相关"
subtitle:   "Storm 配置相关"
date:       2017-05-31
author:     "ZhangZhe"
tags:
    - Java
    - Storm
---

# 日志相关
使用本地模式的时候,能够指定相应的配置文件来控制日志(log4j2)的输出情况.

但是使用集群模式,将jar文件上传到storm集群后,日志的配置文件将使用storm环境指定的日志配置文件,目前使用的是log4j2.存在于storm项目目录下的log4j2/worker.xml,如果想要控制日志输出,需要修改这个配置文件.

ps:如果配置文件出错,将导致日志无法输出,storm ui 页面无法统计单节点错误信息.需要在本地模式进行配置文件的调试.

# 内存占用
首先理解storm中topology,bolt/sprot,worker,executor,task的关系.

[Storm 性能优化](http://www.cnblogs.com/peak-c/p/6297794.html)

[Understanding the Parallelism of a Storm Topology](http://storm.apache.org/releases/1.1.0/Understanding-the-parallelism-of-a-Storm-topology.html)

[Concepts](http://storm.apache.org/releases/1.1.0/Concepts.html)

然后,默认情况下,storm集群给一个worker分配64(日志使用)+768M的内存空间,如果需要修改,则可以在配置文件(conf/storm.yaml)中,增加配置项:worker.heap.memory.mb: 1984(不包含日志的64M).但要注意的是,最好是集群中所有的storm节点都进行配置,因为这个配置项(或许可以扩展到全部配置项),默认会使用leader节点的信息,如果当前的storm节点不是leader(storm ui中查看),修改不会起作用.