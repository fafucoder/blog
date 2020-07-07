---
title: linux nsenter命令
date: 2020-05-09 22:24:35
tags:
- linux
categories:
- linux
---

### 问题概述
要显示某个pod获取容器的IP地址(或者执行ping操作)， docker进入到容器里面，使用的命令包括docker exec或者docker attach。kubernetes进入到pod里面，使用的命令包括 kubectl exec。 进入到容器里面在ifconfig 或者ip a, 但是有的pod却没有这些命令，显示如下, 一般的做法就是起一个带来这些网络命令的pod(比如busybox)。
```bash
root@dawn:~# kubectl exec -it nginx-6db489d4b7-srfp7 /bin/sh
# ifconfig
/bin/sh: 1: ifconfig: not found
# ping
/bin/sh: 2: ping: not found
```

进入到容器网络的namespace等同于进入到容器中，而且还能使用宿主机的网络工具，例如ip或者ifconfig命令，所以我们只需要进入到容器网络就可以使用这些命令了。nsenter就是一个进入到namespace的命令，通过使用nsenter命令就不需要专门跑一个带有网络命令的pod了。

### nsenter用途
nsenter命令是一个可以在指定进程的命令空间下运行指定程序的命令。它位于util-linux包中。nsenter不仅可以进入到network命名空间，也可以进入mnt, uts, ipc, pid, user命令空间，以及指定根目录和工作目录。

### 使用方法
nsenter的命令如下

```bash
Usage:
 nsenter [options] [<program> [<argument>...]]

Run a program with namespaces of other processes.

Options:
 -a, --all              enter all namespaces
 -t, --target <pid>     target process to get namespaces from
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -p, --pid[=<file>]     enter pid namespace
 -C, --cgroup[=<file>]  enter cgroup namespace
 -U, --user[=<file>]    enter user namespace
 -S, --setuid <uid>     set uid in entered namespace
 -G, --setgid <gid>     set gid in entered namespace
     --preserve-credentials do not touch uids or gids
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -F, --no-fork          do not fork before exec'ing <program>
 -Z, --follow-context   set SELinux context according to --target PID

 -h, --help             display this help
 -V, --version          display version

For more details see nsenter(1).
```

### 如何使用
比如有个pod叫做 nginx-6db489d4b7-srfp7, 那我们要怎么进入呢？

1. 获取pod的ContainerID
```bash
dawn@dawn:~$ kubectl get pod nginx-6db489d4b7-srfp7 -oyaml | grep containerID
  - containerID: docker://e60c6eb3f0caedb1f48cd80c2ab0450f54cc344fad2182f26e1f2426c8b597c4

```

2. 获取容器的pid
```bash
dawn@dawn:~$ docker inspect -f {{.State.Pid}} e60c6eb3f0c
24784

或者

dawn@dawn:~$ docker inspect e60c6eb3f0c | grep Pid
            "Pid": 24784,
            "PidMode": "",
            "PidsLimit": null,

```

3. 进入容器网络空间, 进入后可以使用宿主机的网络命令(ping, ifconfig等)
```bash
dawn@dawn:~$ sudo nsenter -n -t 24784
root@dawn:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1400
        inet 10.16.0.5  netmask 255.255.0.0  broadcast 10.16.255.255
        ether 00:00:00:f8:aa:c8  txqueuelen 0  (Ethernet)
        RX packets 30  bytes 1728 (1.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2  bytes 100 (100.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

### 参考资料
- https://staight.github.io/2019/09/23/nsenter%E5%91%BD%E4%BB%A4%E7%AE%80%E4%BB%8B/