---
title: docker网络冲突问题
date: 2020-05-17 16:59:16
tags:
- docker
categories:
- docker
---

### 问题描述
docker0网桥的默认网段为172.17.0.0/16, 某些虚拟机的内网地址为172.17.0.0/16,结果导致网段冲突，不同虚拟机无法互通
```
dawn@node-1:~$ ip route list
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
...
```

### 解决办法, 修改网桥ip地址段
1. 方法一(临时性): 去除docker0的路由 `sudo ip route del 172.17.0.0/16 dev docker0`
2. 方法二(永久): 修改网桥ip地址段: 在daemon.json中，添加"bip":"192.168.100.1/24"配置，重启docker


### 再发现问题
由于有的虚拟机会配置`"live-restore": true`, 所以在daemon.json中修改docker0的默认网段，重启docker发现还是没有生效。遇到这个问题只能重启虚拟机了。

```
Restart the Docker daemon. On Linux, you can avoid a restart (and avoid any downtime for your containers) by reloading the Docker daemon. If you use systemd, then use the command systemctl reload docker. Otherwise, send a SIGHUP signal to the dockerd process.
```
发送SIGHUP信号量是没法生效的，只能重启才可以生效

### 参考文档
- https://docs.docker.com/config/containers/live-restore/ //官方文档
- https://www.dazhuanlan.com/2019/12/24/5e01dd6bb1206 //如何优雅的重启dockerd