---
title: kubernetes生成swagger ui
date: 2020-11-29 17:02:37
tags:
- kubernetes
categories:
- kubernetes
---

### 如何获取kubernetes的Openapi

百度上搜索kubernetes获取Openapi基本上都是老版本的获取方法，非常繁琐。

kubernetes会自己生成openapi，不需要任何配置只需要：

```bash
kubectl proxy --port=8081
```

配置kubectl proxy即可，通过localhost:8081就可以查看api的定义，要获取到openapi.json只需要访问`http://localhost:8081/openapi/v2` 保存为json就行， 获取通过curl `curl localhost:8081/openapi/v2 > swagger.json`

![openapi](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gl65smm4izj312208kdh3.jpg)

### 运行Swagger UI

通过直接跑docker 可以运行swgger ui

```bash
docker run \
    --rm \
    -p 80:8080 \
    -e SWAGGER_JSON=/swagger.json \
    -v $(pwd)/swagger.json:/swagger.json \
    swaggerapi/swagger-ui
```

### 生成自定义Swagger UI

k8s 的swagger ui 包含了所有crd的api, 有时候只想生成自己定义的CRD的swgger ui，这时候就需要编写代码生成， 分为go生成方式跟java生成方式，推荐使用go的生成方式。

生成的方法可以参考文档2, 整体的思想就是自己定义一个apiserver, 然后把自定义的api 通过scheme注册到apiserver中。

### 参考文档

- [kubernetes: 如何查看Swagger UI](https://jonnylangefeld.com/blog/kubernetes-how-to-view-swagger-ui)
- [OpenAPI学习笔记](https://blog.gmem.cc/openapi)