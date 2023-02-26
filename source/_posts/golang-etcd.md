---
title: etcd使用手记
date: 2023-02-25 18:38:58
tags:
- golang
categories:
- golang
---

### 概述

[etcd](https://etcd.io/)是使用Go语言开发的一个开源的、高可用的分布式key-value存储系统，可以用于配置共享和服务的注册和发现。

类似项目有zookeeper和consul。

etcd具有以下特点：

- 完全复制：集群中的每个节点都可以使用完整的存档
- 高可用性：Etcd可用于避免硬件的单点故障或网络问题
- 一致性：每次读取都会返回跨多主机的最新写入
- 简单：包括一个定义良好、面向用户的API（gRPC）
- 安全：实现了带有可选的客户端证书身份验证的自动化TLS
- 快速：每秒10000次写入的基准速度
- 可靠：使用Raft算法实现了强一致、高可用的服务存储目录

### etcd安装使用方法

etcd支持单机模式，以及使用集群模式部署，支持部署在各种操作系统中。

#### mac安装etcd单机模式

mac安装etcd整体上来说非常简单，只需要一个命令即可（linux其实也是一个命令即可）

```bash
brew install etcd
brew searvices start etcd
brew services list

# etcdctl endpoint health
➜  ~ etcdctl endpoint health
127.0.0.1:2379 is healthy: successfully committed proposal: took = 3.219075ms
```

### etcdctl命令使用

```bash
NAME:
	etcdctl - A simple command line client for etcd3.

COMMANDS:
	alarm disarm		Disarms all alarms
	alarm list		Lists all alarms
	auth disable		Disables authentication
	auth enable		Enables authentication
	auth status		Returns authentication status
	check datascale		Check the memory usage of holding data for different workloads on a given server endpoint.
	check perf		Check the performance of the etcd cluster
	compaction		Compacts the event history in etcd
	defrag			Defragments the storage of the etcd members with given endpoints
	del			Removes the specified key or range of keys [key, range_end)
	elect			Observes and participates in leader election
	endpoint hashkv		Prints the KV history hash for each endpoint in --endpoints
	endpoint health		Checks the healthiness of endpoints specified in `--endpoints` flag
	endpoint status		Prints out the status of endpoints specified in `--endpoints` flag
	get			Gets the key or a range of keys
	help			Help about any command
	lease grant		Creates leases
	lease keep-alive	Keeps leases alive (renew)
	lease list		List all active leases
	lease revoke		Revokes leases
	lease timetolive	Get lease information
	lock			Acquires a named lock
	make-mirror		Makes a mirror at the destination etcd cluster
	member add		Adds a member into the cluster
	member list		Lists all members in the cluster
	member promote		Promotes a non-voting member in the cluster
	member remove		Removes a member from the cluster
	member update		Updates a member in the cluster
	move-leader		Transfers leadership to another etcd cluster member.
	put			Puts the given key into the store
	role add		Adds a new role
	role delete		Deletes a role
	role get		Gets detailed information of a role
	role grant-permission	Grants a key to a role
	role list		Lists all roles
	role revoke-permission	Revokes a key from a role
	snapshot restore	Restores an etcd member snapshot to an etcd directory
	snapshot save		Stores an etcd node backend snapshot to a given file
	snapshot status		[deprecated] Gets backend snapshot status of a given file
	txn			Txn processes all the requests in one transaction
	user add		Adds a new user
	user delete		Deletes a user
	user get		Gets detailed information of a user
	user grant-role		Grants a role to a user
	user list		Lists all users
	user passwd		Changes password of user
	user revoke-role	Revokes a role from a user
	version			Prints the version of etcdctl
	watch			Watches events stream on keys or prefixes

OPTIONS:
      --cacert=""				verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""					identify secure client using this TLS certificate file
      --command-timeout=5s			timeout for short running command (excluding dial timeout)
      --debug[=false]				enable client-side debug logging
      --dial-timeout=2s				dial timeout for client connections
  -d, --discovery-srv=""			domain name to query for SRV records describing cluster endpoints
      --discovery-srv-name=""			service name to query when using DNS discovery
      --endpoints=[127.0.0.1:2379]		gRPC endpoints
  -h, --help[=false]				help for etcdctl
      --hex[=false]				print byte strings as hex encoded strings
      --insecure-discovery[=true]		accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]	skip server certificate verification (CAUTION: this option should be enabled only for testing purposes)
      --insecure-transport[=true]		disable transport security for client connections
      --keepalive-time=2s			keepalive time for client connections
      --keepalive-timeout=6s			keepalive timeout for client connections
      --key=""					identify secure client using this TLS key file
      --password=""				password for authentication (if this option is used, --user option shouldn't include password)
      --user=""					username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"			set the output format (fields, json, protobuf, simple, table)
```

etcdctl 命令分为几大模块，分别是跟etcd集群现网的命令例如member, endpoint, 第二个是租约有关的lease（类似ttl）, 第三个是数据相关的，包括watch、put、get等，第四个是跟角色认证相关的信息，常见的命令如下：

```
# -w 参数用于输出执行的格式，只是fields, json, protobuf, sample, table 默认sample
# --endpoints 参数用于etcdctl跟ectd集群建联，默认是127.0.0.1:2379
# --cacert, --cert等参数用于指定ca证书信息
etcdctl -w table endpoint health
etcdctl -w table member list
etcdctl -w json endpoint status

# 增删改查
etcdctl put hello word
etcdctl get hello
etcdctl watch hello
etcdctl del hello

# 可以通过--prefix参数监听指定前缀的参数
etcdctl watch --prefix=true hello # 可以监听到hello1、hello2等参数数据
etcdctl get --prefix=true hello  # 可以获取到hello1、hello2等参数数据
```

### golang操作使用etcd

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		panic(err)
	}
	defer cli.Close()

	etcdBase(cli)
	etcdWatch(cli)
	etcdLease(cli)
	etcdKeepAlive(cli)
}

// etcdBase 基础的创建跟删除
func etcdBase(client *clientv3.Client) {
	var key = "/hb/etcd/base"
	if _, err := client.Put(context.Background(), key, "123456"); err != nil {
		fmt.Printf("error is: %#v \n", err)
	}

	resp, err := client.Get(context.Background(), key)
	if err != nil {
		fmt.Printf("error is: %#v \n", err)
	}

	for _, kvs := range resp.Kvs {
		fmt.Printf("Get key=%v, value=%v \n", string(kvs.Key), string(kvs.Value))
	}

	if _, err := client.Delete(context.Background(), key); err != nil {
		fmt.Printf("error is: %#v \n", err)
	}
}

// etcd watch机器
func etcdWatch(client *clientv3.Client) {
	var watchKey = "/hb/etcd/watch"

	go func() {
		for i := 0; i < 10; i++ {
			_, _ = client.Put(context.Background(), fmt.Sprintf("%s/key_%d", watchKey, i), "world")
		}

		for i := 0; i < 10; i++ {
			_, _ = client.Delete(context.Background(), fmt.Sprintf("%s/key_%d", watchKey, i))
		}
	}()

	go func() {
		watchCh := client.Watch(context.Background(), watchKey, clientv3.WithPrefix())
		for resp := range watchCh {
			for _, ev := range resp.Events {
				fmt.Printf("Watch Type: %s; Key: %s; Value: %s\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
			}
		}
	}()
}

// etcdLease etcd租约
func etcdLease(client *clientv3.Client) {
	var key = "/hb/etcd/lease"
	// 创建一个5秒的租约
	grantResp, err := client.Grant(context.Background(), 5)
	if err != nil {
		fmt.Printf("hello")
		log.Fatal(err)
	}

	// 5秒钟之后, key就会被移除
	_, err = client.Put(context.Background(), key, "lease", clientv3.WithLease(grantResp.ID))
	if err != nil {
		log.Fatal(err)
	}

	resp, err := client.Get(context.Background(), key)
	if err != nil {
		log.Fatal(err)
	}

	for _, kvs := range resp.Kvs {
		fmt.Printf("Lease key=%v, value=%v \n", string(kvs.Key), string(kvs.Value))
	}

	time.Sleep(6 * time.Second)

	resp, err = client.Get(context.Background(), key)
	if err != nil {
		log.Fatal(err)
	}

	for _, kvs := range resp.Kvs {
		fmt.Printf("Lease Expired key=%v, value=%v \n", string(kvs.Key), string(kvs.Value))
	}
}

// etcdKeepAlive etcd keepAlive
func etcdKeepAlive(client *clientv3.Client) {
	var key = "/hb/etcd/keepalive"
	resp, err := client.Grant(context.TODO(), 5)
	if err != nil {
		log.Fatal(err)
	}

	_, err = client.Put(context.TODO(), key, "keepalive", clientv3.WithLease(resp.ID))
	if err != nil {
		log.Fatal(err)
	}

	// the key 'foo' will be kept forever
	ch, err := client.KeepAlive(context.TODO(), resp.ID)
	if err != nil {
		log.Fatal(err)
	}
	for {
		ka := <-ch
		fmt.Println("KeepAlive TTL: ", ka.TTL)
	}
}
```

### etcd工作原理

etcd 是一个基于 Raft 共识算法实现的分布式键值存储服务，在项目结构上采用了模块化设计，其中最主要的三个部分是实现分布式共识的 Raft 模块、实现数据持久化的 WAL 模块和实现状态机存储的 MVCC 模块。

![etcd组件](https://wingsxdu.com/posts/database/etcd/etcd-Architecture@2x.png)

#### Leader 选举

Raft 是一种用来管理日志复制过程的算法，Raft 通过『领导选举机制』选举出一个 Leader，由它全权管理日志复制来实现一致性。一个 Raft 集群包含若干个服务器节点，每一个节点都有一个唯一标识 ID。Raft 算法论文规定了三种节点身份：Leader、Follower 和 Candidate，etcd 的实现中又添加了 PreCandidate 和 Learner 这两种身份。

![etcd原理](https://wingsxdu.com/posts/database/etcd/Node-State-Change@2x.png)

##### leader选择过程

1. 集群启动时所有节点初始状态均为 Follower，同时启动选举定时器（时间随机，降低冲突概率），随后会有一个节点选举成功成为 Leader，在绝大多数时间里集群内的节点会处于这两种身份之一。
2. 当某个节点当选成为Leader后，需要定期的向Follower发送心跳包，Follower收到心跳包后会重置选举定时器。
3. 当一个 Follower 节点的选举计时器超时后（节点在指定的时间之内没有收到leader或者candidate的有效消息时会发起选举），会先进入`preVote`状态（切换为 PreCandidate 身份），在它准备发起一次选举之前，需要尝试连接集群中的其他节点，并询问它们是否愿意参与选举
4. 如果集群中的其它节点能够正常收到 Leader 的心跳消息，那么会拒绝参与选举。
5. 如果有超过法定人数的节点响应并表示参与新一轮选举，该节点会从 PreCandidate 身份切换到 Candidate，发起新一轮的选举。

### 参考文档

- [go操作etcd](https://www.liwenzhou.com/posts/Go/etcd/)
- [分布式键值存储 etcd 原理与实现 · Analyze](https://wingsxdu.com/posts/database/etcd/#top)
- [高可用分布式存储 etcd 的实现原理](https://draveness.me/etcd-introduction/)
- [Etcd Raft库的日志存储](https://www.codedump.info/post/20210628-etcd-wal/)
- https://pkg.go.dev/go.etcd.io/etcd/client/v3
