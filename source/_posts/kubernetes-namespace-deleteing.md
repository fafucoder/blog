---
title: kubernetes中namespace一直处于deleteing状态分析
date: 2021-11-08 23:13:04
tags:
- kubernetes
categories
- kubernetes
---

### 概述

删除namespace,但是一直处于terminating状态，无法被删除

![pod terminating](https://tva1.sinaimg.cn/large/008i3skNly1gw869bt9jgj30tw096q4d.jpg)

### 原因分析

删除namespace是，首先会把属于namespace的扩展资源(如service, extension-api)删除，但是因为某种原因但是扩展资源无法被删除(如extension-api无法响应请求等)，将会导致namespace无法被删除

### 解决方案

#### 解决扩展资源无法被删除原因

1. 可以通过命令获取无法删除的扩展API：

```shell
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --show-all --ignore-not-found -n <terminating-namespace>
```

2. 如果上一个命令返回以下错误消息：`unable to retrieve the complete list of server APIs: <api-resource>/<version>: the server is currently unable to handle the request`，请继续使用您收到的信息运行以下命令

```shell
kubectl get APIService <version>.<api-resource>
```

	3. 查看扩展资源为何无法响应请求

```shell
kubectl describe APIService <version>.<api-resource>
```

	4. 根据描述信息解决相关问题
 	5. 解决完成之后查看namespace是否被删除

```shell
kubectl get ns
```

#### 强制删除namespace

如果上面的方案无法解决(如无法解决扩展API无法响应请求的问题)，可以通过以下方式解决

1. 查看卡在*Terminating*状态的命名空间：

   ```shell
    kubectl get namespaces
   ```

2. 选择一个终止命名空间并查看命名空间的内容以找出`finalizer`：

   ```shell
    kubectl get namespace <terminating-namespace> -o yaml
   ```

   您的 YAML 内容可能类似于以下输出：

   ```shell
    apiVersion: v1
    kind: Namespace
    metadata:
      creationTimestamp: 2018-11-19T18:48:30Z
      deletionTimestamp: 2018-11-19T18:59:36Z
      name: <terminating-namespace>
      resourceVersion: "1385077"
      selfLink: /api/v1/namespaces/<terminating-namespace>
      uid: b50c9ea4-ec2b-11e8-a0be-fa163eeb47a5
    spec:
      finalizers:
      - kubernetes
    status:
      phase: Terminating
   ```

3. 创建临时 JSON 文件：

   ```shell
    kubectl get namespace <terminating-namespace> -o json >tmp.json
   ```

4. 编辑您的`tmp.json`文件。`kubernetes`从`finalizers`字段中删除值并保存文件。`tmp.json`文件可能类似于以下输出：

   ```json
     {
         "apiVersion": "v1",
         "kind": "Namespace",
         "metadata": {
             "creationTimestamp": "2018-11-19T18:48:30Z",
             "deletionTimestamp": "2018-11-19T18:59:36Z",
             "name": "<terminating-namespace>",
             "resourceVersion": "1385077",
             "selfLink": "/api/v1/namespaces/<terminating-namespace>",
             "uid": "b50c9ea4-ec2b-11e8-a0be-fa163eeb47a5"
         },
         "spec": {
            "finalizers": 
         },
         "status": {
             "phase": "Terminating"
         }
     }
   ```

5. 要设置临时代理 IP 和端口，请运行以下命令。在删除卡住的命名空间之前，请务必保持终端窗口打开：

   ```shell
    kubectl proxy
   ```

   代理 IP 和端口可能类似于以下输出：

   ```plaintext
    Starting to serve on 127.0.0.1:8001
   ```

6. 在新的终端窗口中，使用您的临时代理 IP 和端口进行 API 调用：

   ```plaintext
   curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/<terminating-namespace>/finalize
   ```

   相应的输出可能类似于以下内容：

   ```plaintext
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "<terminating-namespace>",
        "selfLink": "/api/v1/namespaces/<terminating-namespace>/finalize",
        "uid": "b50c9ea4-ec2b-11e8-a0be-fa163eeb47a5",
        "resourceVersion": "1602981",
        "creationTimestamp": "2018-11-19T18:48:30Z",
        "deletionTimestamp": "2018-11-19T18:59:36Z"
      },
      "spec": {
   
      },
      "status": {
        "phase": "Terminating"
      }
   }
   ```

   **注意：**该`finalizer`参数已删除。

7. 验证命名空间已被删除：

   ```shell
    kubectl get namespaces
   ```

### 参考文档

- https://www.cnblogs.com/douh/p/12487577.html
- https://www.ibm.com/docs/en/cloud-private/3.2.0?topic=console-namespace-is-stuck-in-terminating-state

