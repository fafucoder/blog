---
title: kubernetes client-go原理解析
date: 2020-01-31 14:29:47
tags:
- kubernetes
categories:
- kubernetes
---

### 参考文档
- https://blog.csdn.net/weixin_42663840/article/details/81699303   //深入浅出client-go
- https://www.cnblogs.com/charlieroro/p/10330390.html  //关于client-go的原理和关键代码
- https://www.huweihuang.com/article/source-analysis/client-go-source-analysis   //client-go源码详细解析
- https://blog.ihypo.net/15763910382218.html  //原理图
- https://mp.weixin.qq.com/s/HSMza_UAkVRMtMLU9LkqFw

### kubebuilder
- https://blog.ihypo.net/15763910382218.html
- https://book.kubebuilder.io/quick-start.html
- https://github.com/kubernetes-sigs/controller-runtime