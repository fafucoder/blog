---
title: linux 文件挂载mount
date: 2021-08-15 21:24:00
tags:
- linux
categories:
- linux
---

### mount 简介

mount用来显示挂载信息或者进行文件系统挂载。mount并非只能挂载文件系统，也可以将目录挂载到另一个目录下，其实它实现的是目录【硬链接】，默认情况下，是无法对目录建立硬链接的，但是通过mount可以完成绑定，绑定后两个目录的inode号是完全相同的，但尽管建立的是目录的【硬链接】，但其实也仅是拿来当软链接用。

### mount 常用命令

##### mount

```shell
mount [-t vfstype ] [-o options] device directory
-a  将/etc/fstab文件里指定的挂载选项重新挂载一遍。
-t  支持ext2/ext3/ext4/vfat/fat/iso9660(光盘默认格式)。不用-t时默认会调用blkid来获取文件系统类型。
-n  不把挂载记录写在/etc/mtab文件中，一般挂载会在/proc/mounts中记录下挂载信息，然后同步到/etc/mtab，指定-n表示不同步该挂载信息。

-o  指定挂载特殊选项。下面是两个比较常用的：
  loop  挂载镜像文件，如iso文件
  ro    只读挂载
  rw    读写挂载
  auto  相当于mount -a
  dev   如果挂载的文件系统中有设备访问入口则启用它，使其可以作为设备访问入口
  default rw,suid,dev,exec,auto,nouser,async,and relatime
  async 异步挂载，只写到内存
  sync  同步挂载，通过挂载位置写入对方硬盘
  atime 修改访问时间，每次访问都修改atime会导致性能降低，所以默认是noatime
  noatime 不修改访问时间，高并发时使用这个选项可以减少磁盘IO
  nodiratime   不修改文件夹访问时间，高并发时使用这个选项可以减少磁盘IO
  exec/noexec  挂载后的文件系统里的可执行程序是否可执行，默认是可以执行exec，优先级高于权限的限定
  remount  重新挂载，此时可以不用指定挂载点。
  suid/nosuid 对挂载的文件系统启用或禁用suid，对于外来设备最好禁用suid
  _netdev 需要网络挂载时默认将停留在挂载界面直到加载网络了。使用_netdev可以忽略网络正常挂载。如NFS开机挂载。
  user  允许普通用户进行挂载该目录，但只允许挂载者进行卸载该目录
  users  允许所有用户挂载和卸载该目录
  nouser  禁止普通用户挂载和卸载该目录，这是默认的，默认情况下一个目录不指定user/users时，将只有root能挂载
```

##### umount

```
umount device/directory
```

### mount bind 命令

mount bind可为当前挂载点绑定一个新的挂载点。

执行如下命令，可创建foo目录的一个镜像目录bar，它们已经绑定在一起：

```shell
mkdir foo bar
mount --bind foo bar
```

mount bind绑定后的两个目录类似于硬链接，无论读写bar还是读写foo，都会反应在另一方，内核在底层所操作的都是同一个物理位置。将bar卸载后，bar目录回归原始空目录状态，期间所执行的修改都保留在foo目录下。

```shell
[root@centos3 mount]# echo "hello foo" > foo/hello
[root@centos3 mount]# ls bar
hello
[root@centos3 mount]# cat bar/hello
hello foo
```

mount bind除了可以绑定两个普通目录，还可以绑定挂载点。

```shell
mount --bind /mnt/foo /mnt/bar
```

##### shared subtrees

nux的每个挂载点都具有一个决定该挂载点是否共享子挂载点的属性，称为shared subtrees(注：这是挂载点的属性，就像是否只读一样的属性)。该属性用于决定某挂载点之下新增或移除子挂载点时，是否同步影响bind另一端的挂载点。

hared subtrees有四种属性值，它们的设置方式分别为：

```
# 挂载前直接设置shared subtrees属性
mount --make-private    --bind <olddir> <newdir>
mount --make-shared     --bind <olddir> <newdir>
mount --make-slave      --bind <olddir> <newdir>
mount --make-unbindable --bind <olddir> <newdir>

# 或者挂载后设置挂载点属性
mount --bind <olddir> <newdir>
mount --make-private    <newdir>
mount --make-shared     <newdir> 
mount --make-slave      <newdir>
mount --make-unbindable <newdir>
```

对于shared subtrees这几种属性值，以mount --bind foo bar为例：

private属性：表示在foo或bar下新增、移除子挂载点，不会体现在另一方，即foo <-x-> bar
shared属性：表示在foo或bar下新增、移除子挂载点，都会体现在另一方，即foo <--> bar
slave属性：类似shared，但只是单向的，foo下新增或移除子挂载点会体现在bar中，但bar中新增或移除子挂载点不会影响foo，即，即foo --> bar, bar -x-> foo
unbindable属性：表示挂载点bar目录将无法执行bind操作关联到其它挂载点

### /etc/fstab与/etc/mtab和/proc/mounts区别

##### /etc/fstab

/etc/fstab是用来存放文件系统的静态信息的文件。当系统启动的时候，系统会自动地从这个文件读取信息，并且会自动将此文件中指定的文件系统挂载到指定的目录。(/etc/fstab 是只读不写的)

##### /etc/mtab

/etc/mtab是当前的分区挂载情况，记录的是当前系统已挂载的分区。每次挂载/卸载分区时会更新/etc/mtab文件中的信息(执行mount命令会改变/etc/mtab的信息)。

##### /proc/mounts

/proc/mounts是mount namespace而引入的，功能跟/etc/mtab一样，都是记录当前分区的挂载情况。因为记录 mount 信息的 /etc/mtab 是全局的，也就是说，即使某个进程有自己的 namespace，但只要还和外面共享同一个 /etc/mtab，那么，里面进行umount/mount操作的信息也会被记录到/etc/mtab里，外面也会看到。

### 参考文档

- [/etc/mtab与/proc/mounts](https://blog.csdn.net/taiyang1987912/article/details/42492741)  //mtab mounts fstab这三个文件的区别
- [man Linux manual page](https://man7.org/linux/man-pages/man8/mount.8.html)    //man mount 命令
- [mount bind功能详解](https://www.junmajinlong.com/linux/mount_bind/#shared-subtrees)         // 骏马金龙老师的文章都很不错的