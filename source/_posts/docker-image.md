---
title: docker 镜像是怎样炼成的
date: 2021-07-28 20:10:35
tags:
- docker
categories:
- docker
---

## Docker 简介

Docker 是一个构建，发布和运行应用程序的开放平台。Docker 以容器为资源分隔和调度的基本单位，容器封装了整个项目运行时所需要的所有环境，通过 Docker 你可以将应用程序与基础架构分离，像管理应用程序一样管理基础架构，以便快速完成项目的部署与交付(docker 本质还是一个进程，只不过这个进程被关进了小黑屋)。

### docker 与传统的虚拟化技术区别

传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，进而在该系统上运行所需应用进程；而 Docker 容器内的应用进程则是直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟，因此要比传统虚拟机更为轻便。

> 关于什么是Hypervisor,  [维基百科](https://zh.wikipedia.org/wiki/Hypervisor)是这样说的：Hypervisor，又称虚拟机监控器（英语：virtual machine monitor，缩写为 VMM），是用来创建与运行[虚拟机](https://zh.wikipedia.org/wiki/虛擬機器)的软件、固件或硬件。

![docker vs 虚拟化](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt084pww6ij30zq0fg0u7.jpg)

### Docker 架构与核心概念

Docker 使用 client-server 架构， Docker 客户端将命令发送给 Docker 守护进程，后者负责构建，运行和分发 Docker 容器。 Docker 客户端和守护程序使用 REST API，通过 UNIX 套接字或网络接口进行通信。

![docker 架构](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt09h6g14oj31380ka40r.jpg)

### docker常见命令

Docker 提供了大量命令用于管理镜像、容器和服务，命令的统一使用格式为：`docker [OPTIONS] COMMAND`

![docker command](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsy8qkirb0j30zw0tiwh9.jpg)

#### 1. 基础命令

- **docker version**：用于查看 docker 的版本信息
- **docker info**：用于查看 docker 的配置信息
- **docker help**：用于查看帮助信息

#### 2. 镜像相关命令

- **docker search 镜像名**：从官方镜像仓库 Docker Hub 查找指定名称的镜像。常用参数为 `--no-trunc`，代表显示完整的镜像信息。
- **docker images**:   列出所有顶层镜像的相关信息。
- **docker pull  镜像名 [:TAG]**：从官方仓库下载镜像，`:TAG` 为镜像版本，不加则默认下载最新版本。
- **docker rmi 镜像名或ID [:TAG]**:  删除指定版本的镜像，不加 `:TAG` 则默认删除镜像的最新版本。
- **docker inspect 镜像名或ID [:TAG]**: 查看镜像的详细信息
- **docker push 镜像名或ID [:TAG]**: 推送镜像到镜像仓库
- **docker image save**: 保存镜像(为tar包)
- **docker image load**: 加载镜像(从tar包或者标准输入)


#### 3. 容器相关命令

- **docker run [OPTIONS] IMAGE [COMMAND] [ARG...]**: 新建并启动容器
- **docker ps [OPTIONS]**: 列出当前所有正在运行的容器。
- **docker start|restart|stop|kill 容器名或ID**: start 命令用于启动已有的容器，restart 用于重启正在运行的容器，stop 用于停止正在运行的容器，kill  用于强制停止容器
- **docker rm 容器名或ID**: 删除已停止的容器
- **docker insepct 容器名或ID**: 查看容器或者镜像的详细信息
- **docker exec -it 容器名或ID sh|/bin/bash**: 进入正在运行的容器，与容器主进程交互
- **docker logs 容器名或ID**: 查看容器日志
- **docker commit container imageName:tag**: 从容器创建一个新的镜像

下面这张图是docker 容器的状态流转图，方便助记:

![docker 命令流转图](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsy8vzdj26j31dw0pc40n.jpg)

## 镜像是怎样炼成的

### 从OCI规范说起

OCI 即Open Container Initiative, linux基金会与2015年6月成立的组织，旨在围绕容器格式和运行时制定一个开放的工业化标准。目前 OCI 主要有三个规范：运行时规范 [runtime-spec](https://github.com/opencontainers/runtime-spec) ，镜像规范 [image-spec](https://www.github.com/opencontainers/image-spec) 以及不常见的镜像仓库规范 [distribution-spec](https://github.com/opencontainers/distribution-spec) 。

![oci spec](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsx2adhgauj30y80783yq.jpg)

#### Image Spec

OCI 规范中的镜像规范(image-spec) 决定了我们的镜像按照什么标准来构建，以及构建完镜像之后如何存放，接着下文提到的 Dockerfile 则决定了镜像的 layer 内容以及镜像的一些元数据信息。一个镜像规范 image-spec 和一个 Dockerfile 就指导着我们构建一个镜像，那么接下来我们就简单了解一下这个镜像规范，看看镜像是长什么样子的，对镜像有个大体的主观认识。官方的image spec主要由以下文件组成:

```
├── annotations.md         # 注解规范
├── config.md              # image config 文件规范
├── considerations.md      # 注意事项
├── conversion.md          # 转换为 OCI 运行时
├── descriptor.md          # OCI Content Descriptors 内容描述
├── image-index.md         # manifest list 文件
├── image-layout.md        # 镜像的布局
├── implementations.md     # 使用 OCI 规范的项目
├── layer.md               # 镜像层 layer 规范
├── manifest.md            # manifest 规范
├── media-types.md         # 文件类型
├── README.md              # README 文档
├── spec.md                # OCI 镜像规范的概览
```

##### Layer

[文件系统](https://github.com/opencontainers/image-spec/blob/master/layer.md)：以 layer （镜像层）保存的文件系统，每个 layer 保存了和上层之间变化的部分，layer 应该保存哪些文件，怎么表示增加、修改和删除的文件等。

##### Image config

[image config](https://github.com/opencontainers/image-spec/blob/master/config.md) 文件: 保存了文件系统的层级信息（每个层级的 hash 值，以及历史信息），以及容器运行时需要的一些信息（比如环境变量、工作目录、命令参数、mount 列表），指定了镜像在某个特定平台和系统的配置(比较接近我们使用 `docker inspect <image_id>` 看到的内容,  通过docker save一个镜像后解压，里面会有config信息)。

```json
{
  "architecture": "amd64",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "DOCKER_VERSION=20.10.6",
      "DOCKER_TLS_CERTDIR=/certs"
    ],
    "Entrypoint": [
      "docker-entrypoint.sh"
    ],
    "Cmd": [
      "sh"
    ],
    "OnBuild": null
  },
  "created": "2021-07-22T07:20:09.896531Z",
  "history": [
    {
      "created": "2021-04-14T19:19:39.267885491Z",
      "created_by": "/bin/sh -c #(nop) ADD file:8ec69d882e7f29f0652d537557160e638168550f738d0d49f90a7ef96bf31787 in / "
    },
    {
      "created": "2021-04-14T19:19:39.643236135Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
      "empty_layer": true
    },
    {
      "created": "2021-04-14T20:16:58.64055643Z",
      "created_by": "/bin/sh -c apk add --no-cache \t\tca-certificates \t\topenssh-client"
    },
    {
      "created": "2021-04-14T20:16:59.651022704Z",
      "created_by": "/bin/sh -c [ ! -e /etc/nsswitch.conf ] && echo 'hosts: files dns' > /etc/nsswitch.conf"
    },
    {
      "created": "2021-04-14T20:16:59.839384238Z",
      "created_by": "/bin/sh -c #(nop)  ENV DOCKER_VERSION=20.10.6",
      "empty_layer": true
    },
    {
      "created": "2021-04-14T20:17:06.162866201Z",
      "created_by": "/bin/sh -c set -eux; \t\tapkArch=\"$(apk --print-arch)\"; \tcase \"$apkArch\" in \t\t'x86_64') \t\t\turl='https://download.docker.com/linux/static/stable/x86_64/docker-20.10.6.tgz'; \t\t\t;; \t\t'armhf') \t\t\turl='https://download.docker.com/linux/static/stable/armel/docker-20.10.6.tgz'; \t\t\t;; \t\t'armv7') \t\t\turl='https://download.docker.com/linux/static/stable/armhf/docker-20.10.6.tgz'; \t\t\t;; \t\t'aarch64') \t\t\turl='https://download.docker.com/linux/static/stable/aarch64/docker-20.10.6.tgz'; \t\t\t;; \t\t*) echo >&2 \"error: unsupported architecture ($apkArch)\"; exit 1 ;; \tesac; \t\twget -O docker.tgz \"$url\"; \t\ttar --extract \t\t--file docker.tgz \t\t--strip-components 1 \t\t--directory /usr/local/bin/ \t; \trm docker.tgz; \t\tdockerd --version; \tdocker --version"
    },
    {
      "created": "2021-04-14T20:17:06.643076836Z",
      "created_by": "/bin/sh -c #(nop) COPY file:abb137d24130e7fa2bdd38694af607361ecb688521e60965681e49460964a204 in /usr/local/bin/modprobe "
    },
    {
      "created": "2021-04-14T20:17:06.854234992Z",
      "created_by": "/bin/sh -c #(nop) COPY file:5b18768029dab8174c9d5957bb39560bde5ef6cba50fbbca222731a0059b449b in /usr/local/bin/ "
    },
    {
      "created": "2021-04-14T20:17:07.047323417Z",
      "created_by": "/bin/sh -c #(nop)  ENV DOCKER_TLS_CERTDIR=/certs",
      "empty_layer": true
    },
    {
      "created": "2021-04-14T20:17:08.025897909Z",
      "created_by": "/bin/sh -c mkdir /certs /certs/client && chmod 1777 /certs /certs/client"
    },
    {
      "created": "2021-04-14T20:17:08.214397968Z",
      "created_by": "/bin/sh -c #(nop)  ENTRYPOINT [\"docker-entrypoint.sh\"]",
      "empty_layer": true
    },
    {
      "created": "2021-04-14T20:17:08.435794313Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"sh\"]",
      "empty_layer": true
    },
    {
      "created": "2021-07-22T07:20:09.896531Z",
      "created_by": "RUN /bin/sh -c apk add curl # buildkit",
      "comment": "buildkit.dockerfile.v0"
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:b2d5eeeaba3a22b9b8aa97261957974a6bd65274ebd43e1d81d0a7b8b752b116",
      "sha256:91e3b96418079299100f88ca50988325e1f397f49d9c49c30b511e947a49bcd3",
      "sha256:563ea45970c143d16be5c97173d2720cc31f38e2892aa0bd9075e5bede9dc334",
      "sha256:9e9604462a57778a1d4b47a01039ee53fe29bd0e332f3f95a2cd20098fec4504",
      "sha256:241588b2373ed654409b0d0881a92b801f8749cb21e3c5ed10d1d82d0a4434de",
      "sha256:4bc13ba6e7d2ba7a0ff8df8f3ddab9893741d0a51f73659e2d0cf731510fb385",
      "sha256:b82f54db960be856743e9edecf13f04e8e650586b5bf6cc829b3efafdcc6aca5",
      "sha256:dd1b694543daae59c44b849b4ae37e82c722b8008a9b2a740ebc698702be7d0c"
    ]
  }
}
```

##### manifest

[manifest 文件](https://github.com/opencontainers/image-spec/blob/master/manifest.md) ：镜像的 config 文件索引，有哪些 layer，额外的 annotation 信息等。manifest 文件是存放在 registry 中，当我们拉取镜像的时候，会根据该文件拉取相应的 layer。

> manifest 是一个 JSON 文件，其定义包括两个部分，分别是Config 和 Layers。Config 是一个 JSON 对象，Layers 是一个由 JSON 对象组成的数组。Config 与 Layers 中的每一个对象的结构相同，都包括三个字段，分别是 digest、mediaType 和 size。其中 digest 可以理解为是这一对象的 ID。mediaType 表明了这一内容的类型。size 是这一内容的大小。
>
> 容器镜像的 Config 有着固定的 mediaType `application/vnd.oci.image.config.v1+json`。一个 Config 的示例配置如下，它记录了关于容器镜像的配置，可以理解为是镜像的元数据。通常它会被镜像仓库用来在 UI 中展示信息，以及区分不同操作系统的构建等。
>
> 而容器镜像的 Layers 是由多层 mediaType 为 `application/vnd.oci.image.layer.v1.*`（其中最常见的是 `application/vnd.oci.image.layer.v1.tar+gzip`) 的内容组成的。众所周知，容器镜像是分层构建的，每一层就对应着 Layers 中的一个对象。
>
> 容器镜像的 Config，和 Layers 中的每一层，都是以 Blob 的方式存储在镜像仓库中的，它们的 digest 作为 Key 存在。因此，在请求到镜像的 Manifest 后，Docker 会利用 digest 并行下载所有的 Blobs，其中就包括 Config 和所有的 Layers。

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 4203,
    "digest": "sha256:08bdaf2f88f90320cd3e92a469969efb1f066c6d318631f94e7864828abd7c75"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 2811969,
      "digest": "sha256:540db60ca9383eac9e418f78490994d0af424aab7bf6d0e47ac8ed4e2e9bcbba"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 2050156,
      "digest": "sha256:5a38b3726f4b24fa93b80450be63ad67fd3239c2f3b83695118d7b1a88447d84"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 154,
      "digest": "sha256:e5fa5deb334027202841b051d10e7c7137fa3b63e97734309cedf6b48804df5f"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 70247077,
      "digest": "sha256:c521bbbd9945cde2415eae95c3a2a604ea5ca18eb86ab5cadeb589d3b8a13185"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 545,
      "digest": "sha256:59707d521c45c42bbb7dbbef463d5d3800859f20122fb54894f0c79d9b435483"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 1015,
      "digest": "sha256:26abe223b186967eef77715b2c6efd147f3a203a0c7b19e61a8947dab4236397"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 150,
      "digest": "sha256:9d806bc203610fb061281b581a0b2522c9fcbd15e55c55ac0c496c9b1dbe63b0"
    }
  ]
}
```

##### 小结

看完 [image-spec](https://www.github.com/opencontainers/image-spec) 里面提到的各种 id 相信你又很多疑惑，在此总结一下这些 id 的作用(看完还是很懵逼)：

| image-id     | image config 的 sha256 哈希值，在本地镜像存储中由它唯一标识一个镜像 |
| :----------- | :----------------------------------------------------------- |
| image digest | 在 registry 中的 image manifest 的 sha256 哈希值，在 registry 中由它唯一标识一个镜像 |
| diff_ids     | 镜像每一层的 id ，是对本地镜像中 layer 的 tar 包的 sha256 哈希值 |
| layer digest | 镜像在 registry 存储中的 id ，是对远程 layer的 tar 包的 sha256 哈希值 |

![image id](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsx192u19nj30r80x440j.jpg)

镜像的 image config 中的 `rootfs` 字段记录了每一层 layer 的 id，而镜像的 layer id 则是 layer tar 包的 sha256 值，如果镜像的 layer 改变，则这个 layer id 会改变，而记录它的 image config 内容也会改变，image config 内容变了，image config 文件的 sha256 值也就会改变，这样就可以由 image id 和 image digest 唯一标识一个镜像，达到防治篡改的安全目的。

![diff_ids](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsx1es0afuj31620ocgs4.jpg)

#### Runtime Spec

容器运行时(Runtime Spec) 决定了根据镜像怎么运行一个容器，规范指定了容器的配置、执行环境和生命周期。其中容器的配置包含了创建和运行容器所需的元数据，包括要运行的进程、环境变量、资源限制和要使用的沙箱功能等。官方的Spec主要内容如下：

```
├── bundle.md              # 将容器编码为文件系统包的格式，一组以某种方式组织的文件
├── config.md              # 容器运行所需的元数据。包括要运行的进程、要注入的环境变量、要使用的沙盒功能等
├── runtime.md             # 容器的生命周期
├── README.md              # README 文档
├── spec.md                # OCI 运行时规范的概览
```

##### Runtime

runtime文件： 规定了state文件包含的内容，以及容器从创建到停止的事件。总结容器运行时主要做一下几方面工作：

- 容器镜像管理
- 容器生命周期管理
- 容器创建
- 容器资源管理

> 市面上常见的容器运行时有runc, rkt, lxc, containerd等。

![runtime](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNgy1gsx2sg6azij316c0iewf3.jpg)

### 关于Rootfs与Bootfs

一个典型的 Linux 系统要能运行的话，它至少需要两个文件系统, rootfs和bootfs.

#### Bootfs

Bootfs 包含BootLoader(引导加载程序)和kernel(内核)。回忆下操作系统的启动过程，当按下电脑的开机键之后，电脑首先进行加电自检(检查硬件是否有问题), 接着就是GRUB引导，然后是内核加载，内核加载完成后进行init初始化(内核启动第一个用户空间程序)，bootfs的作用就是把引导程序和把/boot文件系统加载进内核的过程，当内核都会被加载进内存后，此时 bootfs 会被卸载掉从而释放出所占用的内存。

![bootfs](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsy83q4f2aj30w006s0ss.jpg)

#### Rootfs

rootfs就是root文件系统，包含的就是典型的Linux系统中的/dev, /proc, /bin, /etc等标准目录和文件。

![rootfs](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsy86pccjbj31fk0260su.jpg)

Docker镜像是由文件系统叠加而成。最低端是bootfs，并使用宿主机的bootfs（docker中操作系统启动几秒钟，原因就是，通过docker镜像启动的操作系统，底层使用的是docker宿主机的bootfs不需要重新加载bootfs), 第二层是root文件系统rootfs 称为base image(基础镜像)。然后可以再往上叠加其它镜像文件，每一层就是一个layer(每一层都是只读的)。 当从一个镜像启动容器时，docker会使用联合文件系统把多个不同位置的目录(layer)联合挂载（union mount）到同一个目录下，然后会在最顶层加载一个读写文件系统作为容器。(联合文件系统下一个章节会说明)
![加载过程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt2peemzakj312w0u0jv1.jpg)

### Dockerfile

众所周知 docker 镜像需要一个 Dockerfile 来构建而成，当我们对 OCI 镜像规范和Rootfs,Bootfs有了个大致的了解之后，我们接下来就拿着 Dockerfile 这个 ”图纸“ 去一步步构建镜像。

下面是 [Nodejs 12.22.4](https://github.com/nodejs/docker-node/blob/main/12/alpine3.13/Dockerfile) 版本的Dockerfile:

```dockerfile
FROM alpine:3.13

ENV NODE_VERSION 12.22.4

RUN addgroup -g 1000 node \
    && adduser -u 1000 -G node -s /bin/sh -D node \
    && apk add --no-cache \
        libstdc++ \
    && apk add --no-cache --virtual .build-deps \
        curl \
    && ARCH= && alpineArch="$(apk --print-arch)" \
      && case "${alpineArch##*-}" in \
        x86_64) \
          ARCH='x64' \
          CHECKSUM="2fa5020bab6501cd3e6fb90dae0cd19a64df80bb8e8bf184af4f562df5a08b4f" \
          ;; \
        *) ;; \
      esac \
  && if [ -n "${CHECKSUM}" ]; then \
    set -eu; \
    curl -fsSLO --compressed "https://unofficial-builds.nodejs.org/download/release/v$NODE_VERSION/node-v$NODE_VERSION-linux-$ARCH-musl.tar.xz"; \
    echo "$CHECKSUM  node-v$NODE_VERSION-linux-$ARCH-musl.tar.xz" | sha256sum -c - \
      && tar -xJf "node-v$NODE_VERSION-linux-$ARCH-musl.tar.xz" -C /usr/local --strip-components=1 --no-same-owner \
      && ln -s /usr/local/bin/node /usr/local/bin/nodejs; \
  else \
    echo "Building from source" \
    # backup build
    && apk add --no-cache --virtual .build-deps-full \
        binutils-gold \
        g++ \
        gcc \
        gnupg \
        libgcc \
        linux-headers \
        make \
        python2 \
    # gpg keys listed at https://github.com/nodejs/node#release-keys
    && for key in \
      4ED778F539E3634C779C87C6D7062848A1AB005C \
      94AE36675C464D64BAFA68DD7434390BDBE9B9C5 \
      74F12602B6F1C4E913FAA37AD3A89613643B6201 \
      71DCFD284A79C3B38668286BC97EC7A07EDE3FC1 \
      8FCCA13FEF1D0C2E91008E09770F7A9A5AE15600 \
      C4F0DFFF4E8C1A8236409D08E73BC641CC11F4C8 \
      C82FA3AE1CBEDC6BE46B9360C43CEC45C17AB93C \
      DD8F2338BAE7501E3DD5AC78C273792F7D83545D \
      A48C2BEE680E841632CD4E44F07496B3EB3C1762 \
      108F52B48DB57BB0CC439B2997B01419BD92F80A \
      B9E2F5981AA6E0CD28160D9FF13993A75599653C \
    ; do \
      gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$key" || \
      gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" ; \
    done \
    && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION.tar.xz" \
    && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
    && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
    && grep " node-v$NODE_VERSION.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
    && tar -xf "node-v$NODE_VERSION.tar.xz" \
    && cd "node-v$NODE_VERSION" \
    && ./configure \
    && make -j$(getconf _NPROCESSORS_ONLN) V= \
    && make install \
    && apk del .build-deps-full \
    && cd .. \
    && rm -Rf "node-v$NODE_VERSION" \
    && rm "node-v$NODE_VERSION.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt; \
  fi \
  && rm -f "node-v$NODE_VERSION-linux-$ARCH-musl.tar.xz" \
  && apk del .build-deps \
  # smoke tests
  && node --version \
  && npm --version

ENV YARN_VERSION 1.22.5

RUN apk add --no-cache --virtual .build-deps-yarn curl gnupg tar \
  && for key in \
    6A010C5166006599AA17F08146C2130DFD2497F5 \
  ; do \
    gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$key" || \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" ; \
  done \
  && curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz" \
  && curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz.asc" \
  && gpg --batch --verify yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz \
  && mkdir -p /opt \
  && tar -xzf yarn-v$YARN_VERSION.tar.gz -C /opt/ \
  && ln -s /opt/yarn-v$YARN_VERSION/bin/yarn /usr/local/bin/yarn \
  && ln -s /opt/yarn-v$YARN_VERSION/bin/yarnpkg /usr/local/bin/yarnpkg \
  && rm yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz \
  && apk del .build-deps-yarn \
  # smoke test
  && yarn --version

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

CMD [ "node" ]
```

#### Docker build 原理

使用docker build 构建镜像的流程大概如下:

1. 执行 `docker build -t <imageName:Tag> .`可以使用 `-f`参数来指定 Dockerfile 文件；

2. docker 客户端会将构建命令后面指定的路径(`.`)下的所有文件(打包成一个 tar 包)，发送给 Docker 服务端;

3. docker 服务端收到客户端发送的 tar 包，然后解压，接下来根据 Dockerfile 里面的指令进行镜像的分层构建；

4. docker 下载 FROM 语句中指定的基础镜像，然后将基础镜像的 layer 联合挂载为一层，并在上面创建一个空目录；

5. 接着启动一个临时的容器并在 chroot 中启动一个 bash，运行 `RUN` 语句中的命令

![lsns](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gsz7qvscwmj31qq0ggtei.jpg)

(使用docker ps 查看临时的镜像):

![docker ps](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gszacahe66j31gs03cdgi.jpg)

6. 一条 `RUN` 命令结束后，会把上层目录压缩，形成新镜像中的新的一层；

7. 如果 Dockerfile 中包含其它命令，就以之前构建的层次为基础，从第二步开始重复创建新层，直到完成所有语句后退出；

8. 构建完成之后为该镜像打上 tag；

#### Base Image

当我们在写 Dockerfile 的时候都需要用 `FROM` 语句来指定一个基础镜像，这些基础镜像并不是无中生有，也需要一个 Dockerfile 来构建成镜像。例如上面的Nodejs Dockerfile指定了`apline:3.13`作为基础镜像。下面我们拿 [debian:buster](https://hub.docker.com/_/debian) 这个基础镜像的 [Dockerfile](https://github.com/debuerreotype/docker-debian-artifacts/blob/18cb4d0418be1c80fb19141b69ac2e0600b2d601/buster/Dockerfile) 来看一下基础镜像是如何炼成的(官方镜像例如ubuntu, centos是如何炼成的可以参考 [docker library official images](https://github.com/docker-library/official-images))。

```dockerfile
FROM scratch
ADD rootfs.tar.xz /
CMD ["bash"]
```

一个基础镜像的 Dockerfile 一般仅有三行。第一行 `FROM scratch` 中的`scratch` 这个镜像并不真实的存在。在 Dockerfile 中 `FROM scratch` 指令并不进行任何操作，也就是不会创建一个镜像层；接着第二行的 `ADD rootfs.tar.xz /` 会自动把 `rootfs.tar.xz` 解压到 `/` 目录下，由此产生的一层镜像就是最终构建的镜像真实的 layer 内容；第三行 `CMD ["bash"]` 指定这镜像在启动容器的时候执行的应用程序，一般基础镜像的 CMD 默认为 bash 或者 sh 。

`ADD rootfs.tar.xz /` 中，这个 `rootfs.tar.xz` 就是我们经过一系列骚操作（一般是发行版源码编译）搓出来的根文件系统，这个操作比较复杂(我太菜了，说不清楚哈~)，感兴趣可以去看一下构建 debian 基础镜像的 Jenkins 流水线任务 [debuerreotype](https://doi-janky.infosiftr.net/job/tianon/job/debuerreotype/)，上面有构建这个 `rootfs.tar.xz` 完整过程，或者参考 Debian 官方的 [docker-debian-artifacts](https://github.com/debuerreotype/docker-debian-artifacts) 这个 repo 里的 shell 脚本。

需要额外注意一点，在这里往镜像里添加 `rootfs.tar.xz` 时使用的是 `ADD` 而不是 `COPY` ，因为在 Dockerfile 中的 ADD 指令 src 文件如果是个 tar 包，在构建的时候 docker 会帮我们把 tar 包解开到指定目录，使用 copy 指令则不会解开 tar 包。另外一点区别就是 ADD 指令是添加一个文件，这个文件可以是构建上下文环境中的文件，也可以是个 URL，而 COPY 则只能添加构建上下文中的文件。

当我们把这个 `rootfs.tar.xz` 解开后里面就是一个 Linux 的根文件系统，不同于我们使用 ISO 安装系统的那个根文件系统，这个根文件系统是经过一系列的裁剪，去掉了一些在容器运行中不必要的文件，使之更加轻量适用于容器运行的场景。

```
@todo

使用dockerfile 创建一个镜像~
```


## 镜像是怎样存放的

### 本地存储

当我们构建完一个镜像之后，镜像就存储在了我们 docker 本地存储目录，默认情况下为 `/var/lib/docker` ，下面就探寻一下镜像是以什么样的目录结构存放的。

```shell
[root@centos3 docker]# tree -L 1
.
├── buildkit
├── containers
├── image
├── network
├── overlay2
├── plugins
├── runtimes
├── swarm
├── tmp
├── trust
└── volumes
```

容器的元数据存放在 image 目录下，容器的 layer 数据则存放在 overlay2 目录下。

#### Image 目录

image目录下的overlay2 代表着本地 docker 存储使用的是 overlay2 存储驱动，目前最新版本的 docker 默认优先采用 **overlay2** 作为存储驱动，对于已支持该驱动的 Linux 发行版，不需要任何进行任何额外的配置，可使用 lsmod 命令查看当前系统内核是否支持 overlay2 。overlay2目录下的层级结构如下:

```bash
[root@centos3 overlay2]# tree -L 4 
.
├── distribution
│   ├── diffid-by-digest
│   │   └── sha256
│   │       ├── 12008541203a024dab3a30542c87421e5a27dbb601b9c6f029b3827af138ccb3
│   │       ├── 1f41b2f2bf94740d411c54b48be7f5e9dfbe14f29d1a5cf64f39150d75f39740
│   │       ├── 26abe223b186967eef77715b2c6efd147f3a203a0c7b19e61a8947dab4236397
│   │       ├── 28549a15ba3eb287d204a7c67fdb84e9d7992c7af1ca3809b6d8c9e37ebc9877
│   │       ├── 33847f680f63fb1b343a9fc782e267b5abdbdb50d65d4b9bd2a136291d67cf75
│   │       ├── 363ab70c2143bc121f037ba432cede225b7053200abba2a7a4ca24c2915a3998
│   │       ├── 42ec49ed44205e453626b369a55e9d55e714214b6382663499a030a9293079af
│   │       ├── 4afb39f216bd4e336f9b78584bae0f6bcb77150107471d8d67d3b8abfbdea46a
│   │       ├── 540db60ca9383eac9e418f78490994d0af424aab7bf6d0e47ac8ed4e2e9bcbba
│   └── v2metadata-by-diffid
│       └── sha256
│           ├── 04d694391e96c105b671692a9135fa5e61a5498a5c55b07df0dd4ae8186fe776
│           ├── 0ba2ba6b57851cc6f3c2508b39e4d3d86321598a87f496146d4e5f33aee91c2c
│           ├── 1a86ff1d449f9522c0af1515d11082885b2f198b5ae99673920382d45c929e28
│           ├── 2788d2f5dd8f8427ca7fa6124ca8fdf034ba358e0db75d533f448e0292966a5b
│           ├── 2f4625690283453c620cbf5e13855148d73941df5c2f774eb03d54328c36edc8
│           ├── 322e305537da267d4135a26993d7696e53201988a3059d33e2fabb83a4c81cf8
│           ├── 3764c3e89288ef5786f440a66a5981593ddceb8b273ec9465f45c1758d64ced3
│           ├── 3b6deb83ca3702881214614853644c4109ae6a217e3c0b0042d0a0e60725c69b
├── imagedb
│   ├── content
│   │   └── sha256
│   │       ├── 1fd8e1b0bb7ef6cc63e0094e2fb5a35259bdce7041ef5e7e4c99694ded1041d7
│   │       └── 69593048aa3acfee0f75f20b77acb549de2472063053f6730c4091b53f2dfb02
│   └── metadata
│       └── sha256
├── layerdb
│   ├── mounts
│   │   └── 72d706b5c93d78590875cce887c94351060eec659e281ae39113e0a887b003e2
│   │       ├── init-id
│   │       ├── mount-id
│   │       └── parent
│   ├── sha256
│   │   ├── 518ebede9dd4e553bcee3157e2f935affbf857abaa6ac3856d7fbfa8e0402d1d
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── parent
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   ├── 5b8c72934dfc08c7d2bd707e93197550f06c0751023dabb3a045b723c5e7b373
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   ├── 8f346dcd27d6735db3fd6792708d9fc93424e8c68be6e89f432766d932beea0a
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── parent
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   ├── 9a5d14f9f5503e55088666beef7e85a8d9625d4fa7418e2fe269e9c54bcb853c
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   ├── d3abebe503aa4a993321a26a0ac46b20c45540640bbd55cb1e007a8ec6ce046e
│   │   │   ├── cache-id
│   │   │   ├── diff
│   │   │   ├── parent
│   │   │   ├── size
│   │   │   └── tar-split.json.gz
│   │   └── ebc8eb4880594e292509a2a504a4ffb957a0935c7682d20849800e3024e8507c
│   │       ├── cache-id
│   │       ├── diff
│   │       ├── parent
│   │       ├── size
│   │       └── tar-split.json.gz
│   └── tmp
└── repositories.json
```

##### repositories.json 文件

repositories.json 就是存储镜像元数据信息，主要是 image name 和 image id 的对应，digest 和 image id 的对应。当 pull 完一个镜像的时候 docker 会更新这个文件。当我们 docker run 一个容器的时候也用到这个文件去索引本地是否存在该镜像，没有镜像的话就自动去 pull 这个镜像。

##### distribution文件夹

存放着 layer 的 diff_id 和 digest 的对应关系:

diffid-by-digest :存放 `digest` 到 `diffid` 的对应关系

v2metadata-by-diffid : 存放 `diffid` 到 `digest` 的对应关系

##### layerdb

layerdb 存放在layer的原始数据。 需要注意的是：tar-split.json.gz 文件是 layer tar 包的 split 文件，记录了 layer 解压后的文件在 tar 包中的位置（偏移量），通过这个文件可以还原 layer 的 tar 包，在 docker save 导出 image 的时候会用到，由根据它可以开倒车把解压的 layer 还原回 tar 包。详情可参考 [tar-split](https://github.com/vbatts/tar-split)

```
layerdb/
├── mounts
├── sha256
│   ├── 518ebede9dd4e553bcee3157e2f935affbf857abaa6ac3856d7fbfa8e0402d1d
│   │   ├── cache-id   # docker 下载镜像时随机生成的 id
│   │   ├── diff       # 存放 layer 的 diffid
│   │   ├── parent     # 放当前 layer 的父 layer 的 diffid，最底层的 layer 没有这个文件
│   │   ├── size       # 该 layer 的大小
│   │   └── tar-split.json.gz
```

#### Overlay2 目录

overley2目录下存放的是每个layer解压后的内容，使用tree命令查看可以知道，镜像 layer 的内容都存放在一个 `diff` 的文件夹下，diff 的上级目录就是以镜像 layer 的 digest 为名的目录。其中还有个 `l` 文件夹，下面有一坨坨的硬链接文件指向上级目录的 layer 目录。这个 l 其实就是 link 的缩写，l 下的文件都是一些比 digest 文件夹名短一些的。

```shell
overlay2
├── 259cf6934509a674b1158f0a6c90c60c133fd11189f98945c7c3a524784509ff
│   └── diff
│       ├── bin
│       ├── dev
│       ├── etc
│       ├── home
│       ├── lib
│       ├── media
│       ├── mnt
│       ├── opt
│       ├── proc
│       ├── root
│       ├── run
│       ├── sbin
│       ├── srv
│       ├── sys
│       ├── tmp
│       ├── usr
│       └── var
├── 27f9e9b74a88a269121b4e77330a665d6cca4719cb9a58bfc96a2b88a07af805
│   ├── diff
│   └── work
├── a0df3cc902cfbdee180e8bfa399d946f9022529d12dba3bc0b13fb7534120015
│   ├── diff
│   │   └── bin
│   └── work
├── b2fbebb39522cb6f1f5ecbc22b7bec5e9bc6ecc25ac942d9e26f8f94a028baec
│   ├── diff
│   │   ├── etc
│   │   ├── lib
│   │   ├── usr
│   │   └── var
│   └── work
├── be8c12f63bebacb3d7d78a09990dce2a5837d86643f674a8fd80e187d8877db9
│   ├── diff
│   │   └── etc
│   └── work
├── e8f6e78aa1afeb96039c56f652bb6cd4bbd3daad172324c2172bad9b6c0a968d
│   └── diff
│       ├── bin
│       ├── dev
│       ├── etc
│       ├── home
│       ├── lib
│       ├── media
│       ├── mnt
│       ├── proc
│       ├── root
│       ├── run
│       ├── sbin
│       ├── srv
│       ├── sys
│       ├── tmp
│       ├── usr
│       └── var
└── l
    ├── 526XCHXRJMZXRIHN4YWJH2QLPY -> ../b2fbebb39522cb6f1f5ecbc22b7bec5e9bc6ecc25ac942d9e26f8f94a028baec/diff
    ├── 5RZOXYR35NSGAWTI36CVUIRW7U -> ../be8c12f63bebacb3d7d78a09990dce2a5837d86643f674a8fd80e187d8877db9/diff
    ├── LBWRL4ZXGBWOTN5JDCDZVNOY7H -> ../a0df3cc902cfbdee180e8bfa399d946f9022529d12dba3bc0b13fb7534120015/diff
    ├── MYRYBGZRI4I76MJWQHN7VLZXLW -> ../27f9e9b74a88a269121b4e77330a665d6cca4719cb9a58bfc96a2b88a07af805/diff
    ├── PCIS4FYUJP4X2D4RWB7ETFL6K2 -> ../259cf6934509a674b1158f0a6c90c60c133fd11189f98945c7c3a524784509ff/diff
    └── XK5IA4BWQ2CIS667J3SXPXGQK5 -> ../e8f6e78aa1afeb96039c56f652bb6cd4bbd3daad172324c2172bad9b6c0a968d/diff
```

### Registry 存储

当我们执行`docker push`命令之后，就把镜像存储到远程Registry仓库中了，镜像在远程仓库中是如何存储的呢，这就回到了 OCI 规范中的镜像仓库规范 [distribution-spec](https://github.com/opencontainers/distribution-spec)，该规范就定义着容器镜像如何存储在远端（即 registry）上。我们可以把 registry 看作镜像的仓库，使用该规范可以帮助我们把这些镜像按照约定俗成的格式来存放。

想要分析一下镜像是如何存放在 registry 上的，我们在本地使用 docker run 来起 registry 的容器即可：

```linux
docker run -d --name registry -p 5000:5000 -v /var/lib/registry:/var/lib/registry registry
```

当我们在本地启动一个 registry 容器之后，容器内默认的存储位置为 `/var/lib/registry` ，所以我们在启动的时候加了参数 `-v /var/lib/registry:/var/lib/registry` 将本机的路径挂载到容器内。

接着我们往registry推送busybox镜像

```
docker pull busybox:latest
docker tag busybox:latest localhost:5000/busybox:latest
docker push localhost:5000/busybox:latest
```

然后我们进入/var/lib/registry目录，使用 tree 命令查看一下这个目录的存储结构。

```linux
[root@centos3 v2]# pwd
/var/lib/registry/docker/registry/v2
[root@centos3 v2]# tree
.
├── blobs
│   └── sha256
│       ├── 08
│       │   └── 08bdaf2f88f90320cd3e92a469969efb1f066c6d318631f94e7864828abd7c75
│       │       └── data
│       ├── 26
│       │   └── 26abe223b186967eef77715b2c6efd147f3a203a0c7b19e61a8947dab4236397
│       │       └── data
│       ├── 54
│       │   └── 540db60ca9383eac9e418f78490994d0af424aab7bf6d0e47ac8ed4e2e9bcbba
│       │       └── data
│       ├── 59
│       │   └── 59707d521c45c42bbb7dbbef463d5d3800859f20122fb54894f0c79d9b435483
│       │       └── data
│       ├── 5a
│       │   └── 5a38b3726f4b24fa93b80450be63ad67fd3239c2f3b83695118d7b1a88447d84
│       │       └── data
│       ├── 63
│       │   └── 63cb2df6dfe1fb041b952ddb9f75c69569331fa39577bc41a3e56cf8f79f7e2e
│       │       └── data
│       ├── 69
│       │   └── 69593048aa3acfee0f75f20b77acb549de2472063053f6730c4091b53f2dfb02
│       │       └── data
│       ├── 9d
│       │   └── 9d806bc203610fb061281b581a0b2522c9fcbd15e55c55ac0c496c9b1dbe63b0
│       │       └── data
│       ├── b7
│       │   └── b71f96345d44b237decc0c2d6c2f9ad0d17fde83dad7579608f1f0764d9686f2
│       │       └── data
│       ├── c5
│       │   └── c521bbbd9945cde2415eae95c3a2a604ea5ca18eb86ab5cadeb589d3b8a13185
│       │       └── data
│       ├── dc
│       │   └── dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b
│       │       └── data
│       └── e5
│           └── e5fa5deb334027202841b051d10e7c7137fa3b63e97734309cedf6b48804df5f
│               └── data
└── repositories
    ├── busybox
    │   ├── _layers
    │   │   └── sha256
    │   │       ├── 69593048aa3acfee0f75f20b77acb549de2472063053f6730c4091b53f2dfb02
    │   │       │   └── link
    │   │       └── b71f96345d44b237decc0c2d6c2f9ad0d17fde83dad7579608f1f0764d9686f2
    │   │           └── link
    │   ├── _manifests
    │   │   ├── revisions
    │   │   │   └── sha256
    │   │   │       └── dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b
    │   │   │           └── link
    │   │   └── tags
    │   │       └── latest
    │   │           ├── current
    │   │           │   └── link
    │   │           └── index
    │   │               └── sha256
    │   │                   └── dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b
    │   │                       └── link
    │   └── _uploads
    └── docker
        ├── _layers
        │   └── sha256
        │       ├── 08bdaf2f88f90320cd3e92a469969efb1f066c6d318631f94e7864828abd7c75
        │       │   └── link
        │       ├── 26abe223b186967eef77715b2c6efd147f3a203a0c7b19e61a8947dab4236397
        │       │   └── link
        │       ├── 540db60ca9383eac9e418f78490994d0af424aab7bf6d0e47ac8ed4e2e9bcbba
        │       │   └── link
        │       ├── 59707d521c45c42bbb7dbbef463d5d3800859f20122fb54894f0c79d9b435483
        │       │   └── link
        │       ├── 5a38b3726f4b24fa93b80450be63ad67fd3239c2f3b83695118d7b1a88447d84
        │       │   └── link
        │       ├── 9d806bc203610fb061281b581a0b2522c9fcbd15e55c55ac0c496c9b1dbe63b0
        │       │   └── link
        │       ├── c521bbbd9945cde2415eae95c3a2a604ea5ca18eb86ab5cadeb589d3b8a13185
        │       │   └── link
        │       └── e5fa5deb334027202841b051d10e7c7137fa3b63e97734309cedf6b48804df5f
        │           └── link
        ├── _manifests
        │   ├── revisions
        │   │   └── sha256
        │   │       └── 63cb2df6dfe1fb041b952ddb9f75c69569331fa39577bc41a3e56cf8f79f7e2e
        │   │           └── link
        │   └── tags
        │       └── latest
        │           ├── current
        │           │   └── link
        │           └── index
        │               └── sha256
        │                   └── 63cb2df6dfe1fb041b952ddb9f75c69569331fa39577bc41a3e56cf8f79f7e2e
        │                       └── link
        └── _uploads
```

(有点看不懂，且看下面的层级结构图):

![目录结构](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt1jxsria5j31lv0u0jwe.jpg)

#### blobs 目录

`blobs` 目录用来存放镜像的三种文件： layer 的真实数据，镜像的 manifest 文件，镜像的 image config 文件。这些文件都是以 `data` 为名的文件存放在于该文件 `sha256` 相对应的目录下.对于 layer 来讲，这是个 `data` 文件的格式是 `vnd.docker.image.rootfs.diff.tar.gzip` ，我们可以使用 `tar -xvf` 命令将这个 layer 解开。当我们使用 docker pull 命令拉取镜像的时候，也是去下载这个 `data`文件，下载完成之后会有一个 `docker-untar`的进程将这个 `data`文件解开存放在`/var/lib/docker/overlay2/${digest}/diff` 目录下。

#### repositories 目录

repositories 目录存放着各个镜像的manifest和layer信息，每个镜像版本都有_layers, _manifests,  _uploads这三个文件夹。

```
[root@centos3 busybox]# pwd
/var/lib/registry/docker/registry/v2/repositories/busybox
[root@centos3 busybox]# ls
_layers  _manifests  _uploads
```

##### _upload文件夹

 _uploads 文件夹是个临时的文件夹，主要用来存放 push 镜像过程中的文件数据，当镜像 `layer` 上传完成之后会清空该文件夹。其中的 `data` 文件上传完毕后会移动到 `blobs` 目录下。

##### _manifest 文件夹

`_manifests` 文件夹是镜像上传完成之后由 registry 来生成的，并且该目录下的文件都是一个名为 `link`的文本文件，它的值指向 blobs 目录下与之对应的目录。在_manifests` 文件夹下包含着镜像的 `tags` 和 `revisions` 信息，每一个镜像的每一个 tag 对应着于 tag 名相同的目录，

每个 `tag`目录下面有 `current` 目录和 `index` 目录， `current` 目录下的 link 文件保存了该 tag 目前的 manifest 文件的 sha256 编码，对应在 `blobs` 中的 `sha256` 目录下的 `data` 文件，而 `index` 目录则列出了该 `tag` 历史上传的所有版本的 `sha256` 编码信息。`_revisions` 目录里存放了该 `repository` 历史上上传版本的所有 sha256 编码信息。

当我们 `pull` 镜像的时候如果不指定镜像的 `tag`名，默认就是 latest，registry 会从 HTTP 请求中解析到这个 tag 名，然后根据 tag 名目录下的 current link 文件找到该镜像的  manifest 的位置返回给客户端，客户端接着去请求这个 manifest 文件，最后客户端根据这个 manifest 文件来 pull 相应的镜像 layer 。

```linux
[root@centos3 busybox]# find ./_manifests/ -type f
./_manifests/revisions/sha256/dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b/link
./_manifests/tags/latest/index/sha256/dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b/link
./_manifests/tags/latest/current/link

[root@centos3 busybox]# cat ./_manifests/tags/latest/index/sha256/dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b/link
sha256:dca71257cd2e72840a21f0323234bb2e33fea6d949fa0f21c5102146f583486b
```

>registry 存储目录里并不会存储与该 registry 相关的信息，比如我们 push 镜像的时候需要给镜像加上 `localhost:5000` 这个前缀，这个前缀并不会存储在 registry 存储中。假如要迁移一个很大的 registry 镜像仓库，镜像的数量在 5k 以上。最便捷的办法就是打包这个 registry 存储目录，将这个 tar 包 rsync 到另一台机器即可。需要强调一点，打包 registry 存储目录的时候不需要进行压缩，直接 `tar -cvf` 即可。因为 registry 存储的镜像 layer 已经是个 `tar.gzip` 格式的文件，再进行压缩的话效果甚微而且还浪费 CPU 时间得不偿失。

## 镜像是怎么搬运的

当我们知道镜像是如何存储了之后，我们就该了解镜像是如何传输的了。

### docker pull

理解 docker pull 一个镜像的流程最好的办法是查看 OCI registry 规范中的这段文档 [pulling-an-image](https://github.com/opencontainers/distribution-spec/blob/master/spec.md#pulling-an-image), 下面流程图是镜像传输的流程:

![docker pull](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt1mm3241qj30wa0sqdj2.jpg)

docker pull 就和我们使用 git clone 一样效果，将远程的镜像仓库拉取到本地来给容器运行时使用，结合上图大致的流程如下：

- 第一步应该是使用`~/.docker/config.json` 中的 auth 认证信息在 registry 那里进行鉴权授权，拿到一个 token，后面的所有的 HTTP 请求中都要包含着该 token 才能有权限进行操作
- dockerd 守护进程解析 docker 客户端参数，接着向 registry 请求 manifest 文件
- dockerd 得到 `manifest` 后，读取里面 image config 文件的 `digest`，这个 sha256 值就是 image 的 `ID`
- 根据 `ID` 在本地的 `repositories.json`中查找找有没有存在同样 `ID` 的 image，有的话就不用下载了
- 如果没有，那么会给 registry 服务器发请求拿到 image config 文件
- 根据 image config 文件中的 `diff_ids`在本地找对应的 layer 是否存在
- 如果 layer 不存在，则根据 `manifest` 里面 layer 的 `sha256` 和 `media type` 去服务器拿相应的 layer（相当去拿压缩格式的包）
- dockerd 守护进程并行下载各 layer 
- 拿到后进行解压，并检查解压(gzip -d)后 tar 包的 sha256 是否和 image config 中的 `diff_id` 相同，不相同就翻车了
- 等所有的 layer 都下载完成后，整个 image 的 layer 就下载完成，接着开始进行解压(tar -xf) layer 的 tar 包。
- dockerd 起一个单独的进程 `docker-untar` 来 gzip 解压缩已经下载完成的 layer 文件
- 验证 image config 中的 RootFS.diffids 是否与解压后hash 相同

### docker push

docker push 的流程恰好和 docker pull 拉取镜像到本地的流程相反。docker pull 一个镜像的时候往往需要先获取镜像 的 manifest 文件，然后根据这个文件中的 layer 信息取相应的 layer。doker push 一个镜像，需要先将镜像的各个 layer 推送到 registry ，当所有的镜像 layer 上传完毕之后最后再 push Image manifest 到 registry。大体的流程如下:

- 第一步和 pull 一个镜像一样也是进行鉴权授权，拿到一个 token
- 向 registry 发送 `POST /v2/<name>/blobs/uploads/`请求，registry 返回一个上传镜像 layer 时要应到的 URL
- 客户端通过 `HEAD /v2/<name>/blobs/<digest>` 请求检查 registry 中是否已经存在镜像的 layer
- 客户端通过URL 使用 POST 方法来实时上传 layer 数据，上传镜像 layer 分为 `Monolithic Upload` （整体上传）和`Chunked Upload`（分块上传）两种方式
- 镜像的 layer 上传完成之后，客户端向 registry 发送一个 PUT HTTP 请求告知该 layer 已经上传完毕。
- 当所有的 layer 上传完之后，再将 manifest 推送上去

> 此处应该安利下skopeo, skopeo对跨registry传输的镜像场景下非常实用， 免去了像 docker pull –> docker tag –> docker push 这样的流程，感兴趣的同学可以去了解下~

## 镜像是怎样食用的

当我们拿到一个镜像之后, 容器运行时通过一个叫 [OCI runtime filesytem bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md) 的标准格式将 OCI 镜像通过工具转换为 bundle ，然后 OCI 容器引擎能够识别这个 bundle 来运行容器。

>filesystem bundle 是个目录，用于给 runtime 提供启动容器必备的配置文件和文件系统。标准的容器 bundle 包含以下内容：
>
>- config.json: 该文件包含了容器运行的配置信息，该文件必须存在 bundle 的根目录，且名字必须为 config.json
>- 容器的根目录，可以由 config.json 中的 root.path 指定

![oci bundle](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt0ant3ao0j61750u00ts02.jpg)

当我们启动一个容器之后使用 tree 命令来分析一下 overlay2目录 就会发现，较之前的目录，容器启动之后 overlay2 目录下多了一个 `merged` 的文件夹，该文件夹就是容器内看到的。docker 通过 overlayfs (联合文件系统)联合挂载的技术将镜像的多层 layer 挂载为一层，这层的内容就是容器里所看到的，也就是 merged 文件夹。

### 什么是联合文件系统

联合挂载是一种文件系统，它可以在不修改其原始（物理）源的情况下创建多个目录，并把内容合并为一个文件的错觉。联合挂载或联合文件系统是文件系统；但不是文件系统类型，而是一个包含许多实现的概念。docker 支持多种文件系统，现在默认使用overlay2作为默认的存储驱动程序(version > 18.06)，如果是老版本则默认使用aufs作为存储驱动程序(version <=18.06), 下面就介绍几种常见的联合文件系统：

#### UnionFS

UnionFS 其实是一种为 Linux 操作系统设计的用于把多个文件系统『联合』到同一个挂载点的文件系统服务。 现在UnionFS 似乎不再积极开发，其最新提交是从 2014 年 8 月开始的。您可以在其网站https://unionfs.filesystems.org/上阅读更多有关它的信息。

#### AUFS

AUFS 即 Advanced UnionFS 其实就是 UnionFS 的升级版, 它能够将将单个 Linux 主机上的多个目录分层并将它们呈现为单个目录。这些目录在 AUFS 术语中称为*分支*，在 Docker 术语中称为*层*，关于AUFS如何工作的在 [docker 官方文档 ](https://docs.docker.com/storage/storagedriver/aufs-driver/) 里有说明

![aufs](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt0b8ajmpzj617k0lsdiq02.jpg)

#### OverlayFS

OverlayFS也是一个现代*联合文件系统*，类似于 AUFS，但速度更快，实现更简单。Docker 为 OverlayFS 提供了两个存储驱动程序：原始的`overlay`和更新更稳定的`overlay2`。

OverlayFS 由低层和高层的目录组成，下层目录称之为`lowerdir`, 上层称之为`upperdir`, 文件系统中低层的目录是只读的，而高层的文件系统则是可读可写的。

下图显示了 Docker 镜像和 Docker 容器是如何分层的。图像层是`lowerdir`，容器层是`upperdir`。`merged` 为镜像和容器中所有图层的合并视图。关于AUFS如何工作的在 [docker 官方文档 ](https://docs.docker.com/storage/storagedriver/overlayfs-driver/) 里有说明

![overlay](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gt0bmp76u7j315i0asmz3.jpg)

当我们使用`docker inspect `命令的时候可以看到镜像(容器)的的层信息

```linux
[root@centos3 nginx]# docker inspect nginx | jq .[0].GraphDriver.Data
{
  "LowerDir": "/var/lib/docker/overlay2/26b544d41358c2c6818f5ec30855b997a85b7d7d5291bf3115def847626971a3/diff:/var/lib/docker/overlay2/452f96de6d70b17221a06b9b81af3cb5b5b855231c018826f030dbc9393e1867/diff:/var/lib/docker/overlay2/6a10e6330e511ef9f6ea94301366d3b303da6b751a9703fc223a73869aa98912/diff:/var/lib/docker/overlay2/7c28defbbc32d0bd94dc675639a8e26ffd9e16903d1c69cce6a109d495bb230c/diff:/var/lib/docker/overlay2/32680ffbf919dbc1d2a4a626ce4bba1203dc0c3ec649dfb44f9c15657514e0e8/diff",
  "MergedDir": "/var/lib/docker/overlay2/be030cc6ed9cfa5c9e7ba78d6d62a16842e09940f615617c0ed284bc80c8f3ce/merged",
  "UpperDir": "/var/lib/docker/overlay2/be030cc6ed9cfa5c9e7ba78d6d62a16842e09940f615617c0ed284bc80c8f3ce/diff",
  "WorkDir": "/var/lib/docker/overlay2/be030cc6ed9cfa5c9e7ba78d6d62a16842e09940f615617c0ed284bc80c8f3ce/work"
}
```

>- LowerDir: 是只读镜像层的目录，以冒号分隔
>- MergedDir：镜像和容器中所有图层的合并视图
>- UpperDir：写入更改的读写层
>- WorkDir：Linux OverlayFS 用于准备合并视图的工作目录

#### ZFS

 ZFS 是由 Sun Microsystems（现在是 Oracle）创建的联合文件系统。它有一些有趣的功能，如分层校验和、快照和备份/复制的本机处理或本机数据压缩和重复数据删除。但是，由 Oracle 维护，它具有非 OSS 友好许可 (CDDL)，因此不能作为 Linux 内核的一部分提供。但是，您可以在 Linux (ZoL)项目上使用 ZFS，Docker 文档中将其描述为健康和成熟的......，但尚未准备好用于生产。关于ZFS如何工作的在 [docker 官方文档 ](https://docs.docker.com/storage/storagedriver/zfs-driver/) 里有说明

#### Btrfs

 Btrfs是多家公司（包括 SUSE、WD 或 Facebook ）的联合项目，在 GPL 许可下发布，是 Linux 内核的一部分。Btrfs 是 Fedora 33 的默认文件系统。它还具有一些有用的功能，例如块级操作、碎片整理、可写快照等等。Btrfs 作为下一代写时复制文件系统，非常适合 Docker。关于Btrfs如何工作的在 [docker 官方文档 ](https://docs.docker.com/storage/storagedriver/btrfs-driver/) 里有说明

### 动手实验部分

##### docker overlay2创建联合挂载

```bash
#!/bin/bash

# 删除所有镜像
# docker image prune -af

# 拉取nginx镜像(使用degist模式以保证layer都一致)
docker pull nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90

# 查看GraphDriver
docker inspect nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90 | jq .[0].GraphDriver.Data

# 创建nginx文件夹
mkdir -p /root/nginx

# 联合挂载, lowerdir, uppperdir, workdir 使用 GraphDriver.Data中的数据
mount -t overlay -o \
lowerdir=/var/lib/docker/overlay2/26b544d41358c2c6818f5ec30855b997a85b7d7d5291bf3115def847626971a3/diff:/var/lib/docker/overlay2/452f96de6d70b17221a06b9b81af3cb5b5b855231c018826f030dbc9393e1867/diff:/var/lib/docker/overlay2/6a10e6330e511ef9f6ea94301366d3b303da6b751a9703fc223a73869aa98912/diff:/var/lib/docker/overlay2/7c28defbbc32d0bd94dc675639a8e26ffd9e16903d1c69cce6a109d495bb230c/diff:/var/lib/docker/overlay2/32680ffbf919dbc1d2a4a626ce4bba1203dc0c3ec649dfb44f9c15657514e0e8/diff,\
upperdir=/var/lib/docker/overlay2/be030cc6ed9cfa5c9e7ba78d6d62a16842e09940f615617c0ed284bc80c8f3ce/diff,\
workdir=/var/lib/docker/overlay2/be030cc6ed9cfa5c9e7ba78d6d62a16842e09940f615617c0ed284bc80c8f3ce/work \
overlay /root/nginx

# 取消挂载并删除目录
# umount overlay
# rm -rf /root/nginx
```

##### 手动创建overlay2联合挂载

```bash
#!/bin/bash

# 创建文件夹
cd /root && rm -rf overlay2 && mkdir overlay2 && cd $_

# 创建layer1
mkdir -p layer1/bin

echo "layer1" > layer1/bin/bash
echo "layer1" > layer1/layer1


# 创建layer2
mkdir -p layer2/bin

echo "layer2" > layer2/bin/bash
echo "layer2" > layer2/layer2


# 创建layer3
mkdir -p layer3/etc

echo "layer3" > layer3/etc/cat
echo "layer3" > layer3/layer3


mkdir -p layer4 && \
mkdir -p workdir && \
mkdir -p merged

mount -t overlay -o \
lowerdir=/root/overlay2/layer1:/root/overlay2/layer2:/root/overlay2/layer3,\
upperdir=/root/overlay2/layer4,\
workdir=/root/overlay2/workdir \
overlay1 /root/overlay2/merged
```

```bash
[root@centos3 overlay2]# ls
layer1  layer2  layer3  layer4  merged  workdir

# 切换到merged层并且创建layer文件
[root@centos3 overlay2]# cd merged/
[root@centos3 merged]# echo "hello world" > layer
[root@centos3 merged]# cat layer
hello world

# layer会复制一份到layer4
[root@centos3 merged]# cd ../layer4
[root@centos3 layer4]# ls
layer
[root@centos3 layer4]# cat layer 
hello world

# 删除layer文件，layer4下的文件也会删除
[root@centos3 layer4]# cd - 
[root@centos3 merged]# rm layer
rm: remove regular file ‘layer’? y
[root@centos3 overlay2]# cd layer4
[root@centos3 layer4]# ls
[root@centos3 layer4]# 
```

##### 使用runc 运行联合挂载的目录

```shell
#!/bin/bash

# 拉取busybox镜像(使用degist模式以保证layer都一致)
docker pull busybox@sha256:0f354ec1728d9ff32edcd7d1b8bbdfc798277ad36120dc3dc683be44524c8b60

# 查看GraphDriver
docker inspect nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90 | jq .[0].GraphDriver.Data

# 创建busybox文件夹, 一定要在rootfs文件夹下，因为runc的config.json默认配置是rootfs
mkdir -p /root/busybox/rootfs && mkdir -p /root/busybox/workdir && /root/busybox/upperdir

# 联合挂载 lowerdir 使用 GraphDriver.Data中upperdir中的数据
mount -t overlay -o \
lowerdir=/var/lib/docker/overlay2/f3640f23bac0bbbac072f3609bba70cd50a3160a8820828d9047173686fe170d/diff,\
upperdir=/root/busybox/upperdir,\
workdir=/root/busybox/workdir \
busybox /root/busybox/rootfs

# 切换到busybox目录下
cd /root/busybox

# 创建config.json
runc spec

# 通过runc运行busybox
runc run mybusybox
```

#### 通过runc 运行容器

```bash
# 创建目录
mkdir -p /root/mybusybox/rootfs && cd /root/mybusybox

# 生成rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -

# 生成config.json文件
runc spec

# 运行容器
runc run mybusybox1
```



### 参考文档

[深入浅出容器镜像的一生🤔](https://blog.k8s.li/Exploring-container-image.html )  // 木子李的博客，上面的内容基本拷贝的木子李的~

[关于存储驱动程序 ](https://docs.docker.com/storage/storagedriver/ )  //docker 官方文档,推荐阅读

[虚拟化技术与Hypervisor回顾](https://www.jianshu.com/p/dd463368a3c9) //理解一下虚拟化也不错

[镜像是怎样炼成的](https://blog.fleeto.us/post/how-are-docker-images-built)   // 伪架构师的博客

[深入研究Docker联合文件系统](https://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247495888&idx=1&sn=39ed455b12cc2e3ad18f5f72c8e6cc21&chksm=ea77c468dd004d7e42aa4a9c095822e41fd3fa01a08156181615fb66f724bc0d03ff228a39bd&mpshare=1&scene=23&srcid=0725WOb425c0NjJIJh8vjO0F&sharer_sharetime=1627183441694&sharer_shareid=2c5557b062e3783c43cbfcd88042eb27%2523rd) //linux云计算网络

[OCI 和 runc：容器标准化和 docker](https://cizixs.com/2017/11/05/oci-and-runc/) //关于OCI的解读，以及如何通过runc 起一个容器，值得一看，推荐阅读

[开放容器计划 (OCI) 规范](https://www.alibabacloud.com/blog/open-container-initiative-oci-specifications_594397) // 阿里国际站上的博客，关于OCI的解读，图不错(英文的)

[Docker 原理篇（三）Docker 基础操作和基础概念](https://www.shangyang.me/2016/12/20/docker-basics/) // 命令的状态流转图还不错，值得一看

[Docker 原理与核心概念](https://juejin.cn/post/6844904039180681229) //docker 的基本命令，命令的状态流转图也不错

https://windsock.io/explaining-docker-image-ids/  // docker 镜像ID，英文的

https://segmentfault.com/a/1190000011294361  //关于container -> docker-shim的

