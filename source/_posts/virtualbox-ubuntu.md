---
title: virtualbox 安装ubuntu问题记录
date: 2020-01-10 11:39:01
tags:
- 环境搭建
categories:
- 环境搭建
---

### 安装ubuntu server源速度慢
安装过程中，不要使用默认的源，在安装时选择国内源可解决此问题
`http://cn.archive.ubuntu.com/ubuntu/` 替换为 `http://mirrors.163.com/ubuntu`或`http://mirrors.sohu.com/ubuntu`

### virtualbox 复制中IP冲突
/etc/machine-id，里面存放了机器ID，并尝试修改成不同值后，DHCP能得到不同IP地址了。

/etc/machine-id 是一个只读文件，权限是444（即,r--r--r--）。有两种修改方法：

- 先chmod修改其权限，让它可写；然后修改它；再将权限改回。
- sudo将它删除，然后创建一个并写入一些字符；再改回权限。

machine-id中的值是一串字母数字组合，应该是固定长度。

胡乱修改一下，使它缺几位，多几位，非法字符等，则系统重启后，该值会自动重新分配。

### 参考资料

- https://www.cnblogs.com/jyginger/p/12297548.html
- https://blog.csdn.net/u011850815/article/details/43668263