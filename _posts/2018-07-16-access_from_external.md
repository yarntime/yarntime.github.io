---
layout: post
title: 从K8S集群外部访问容器IP
---

在k8s集群内部通常使用service ip访问目标服务，在测试过程中有时希望可以从集群外部直接访问容器ip，比如redis sentinel ha模式，或者solrcloud集群模式测试等。一般情况下从k8s集群外是不能直接访问容器ip的，可以使用添加路由的方式实现从k8s集群外部访问容器IP。

操作步骤如下：

1. 打开kubernetes节点的系统转发功能。

2. route add 10.254.0.0 mask 255.255.0.0 {NODE_IP}

同理可以配置路由实现从外部访问k8s service ip。
   
  