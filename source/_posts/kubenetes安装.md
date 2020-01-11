---
title: kubeadm搭建kubernetes环境(ubuntu)
date: 2020-01-10 11:39:01
tags:
- kubebernetes
- 环境搭建
---

### 安装docker
方法一：
https://docs.docker.com/v17.12/install/

方法二(使用国内镜像):
1. curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add 
2. sudo add-apt-repository "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
3. sudo apt-get update
4. sudo apt install docker-ce

#### docker 国内镜像列表
- 中国科技大学: https://docker.mirrors.ustc.edu.cn
- ustc: https://docker.mirrors.ustc.edu.cn
- 网易: http://hub-mirror.c.163.com
- docker中国：https://registry.docker-cn.com

#### 默认使用国内镜像
sudo vim /etc/docker/daemon.json
`
{
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
`
sudo service docker reload

#### 解决docker没权限问题(把当前用户组加到docker里面)
1. sudo groupadd docker            #添加docker用户组
2. sudo gpasswd -a $USER docker    #将登陆用户加入到docker用户组中
3. newgrp docker                   #更新用户组
4. docker ps                       #测试docker命令是否可以使用sudo正常使用

### 禁用交换分区
1. sudo sed -i '/swap/ s/^/#/' /etc/fstab
2. sudo swapoff -a

### 安装k8s黄精
#### 使用阿里云的源
1. sudo apt update && sudo apt install -y apt-transport-https curl
2. curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
3. echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

#### 安装kubeadm, kubelet, kubectl(默认安装的是最新版)
1. sudo apt update
2. sudo apt install -y kubelet kubeadm kubectl
3. sudo apt-mark hold kubelet kubeadm kubectl

#### 安装kubeadm, kubelet, kubectl(指定版本)
1. sudo apt update
2. sudo apt install -y kubelet=1.15.0-00 kubeadm=1.15.0-00 kubectl=1.15.0-00
3. sudo apt-mark hold kubelet=1.15.0-00 kubeadm=1.15.0-00 kubectl=1.15.0-00

#### 验证是否安装成功
1. kubeadm version
2. kubectl version

### 初始化集群
#### master节点
1. sudo kubeadm init --pod-network-cidr=192.168.0.0/16（还有其他参数可以加。。。）
2. 已经init完，可以通过`kubeadm token create --print-join-command`获取token

#### node节点
sudo kubeadm join 172.20.245.50:6443 --token hsf26a.7la0qux76fnm0wko     --discovery-token-ca-cert-hash sha256:8aae49541d380f16fae8a3fe4b9c436fbb5e7d2d4641f30790e561479f48cf13 --ignore-preflight-errors=all -v 10

#### 解决无法init问题
原因：　无法拉取包
解决办法： 
1. fq(百度，github)
2. 本地先拉去下来，然后替换（https://blog.csdn.net/jinguangliu/article/details/82792617)

#### node not ready排查
master节点上：　`kubectl describe nodes node-name` 查看node的详细信息
node节点上： `journalctl -xefu kubelet`, 一般问题可能就是包没法下载
docker排查：　`docker logs c36c56e4cfa3`