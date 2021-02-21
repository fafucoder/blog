---
title: Linux shell编程
date: 2020-04-03 09:29:31
tags:
- linux
categories:
- linux
---

### 常见特殊符号
- $# 是传给脚本的参数个数
- $@ 是传给脚本的所有参数的列表
- $* 是以一个单字符串显示所有向脚本传递的参数，与位置变量不同，参数可超过9个
- $$ 是脚本运行的当前进程ID号
- $? 是显示最后命令的退出状态，0表示没有错误，其他表示有错误

- $0 是脚本本身的名字
- $1 是传递给该shell脚本的第一个参数
- $2 是传递给该shell脚本的第二个参数

- $() 和 ` ` 是命令替换
- ${} 是参数替换
- $_ 在此之前执行的命令或者脚本的最后一个参数

### 命令执行返回值
在 Linux 下，不管你是启动一个桌面程序也好，还是在控制台下运行命令，所有的程序在结束时，都会返回一个数字值，这个值叫做返回值，或者称为错误号 ( Error Number )。

在控制台下，有一个特殊的环境变量 $?，保存着前一个程序的返回值

只要返回值是 0，就代表程序执行成功了～

![$?](https://tva1.sinaimg.cn/large/008eGmZEly1gnv90q36bgj30nc064wfp.jpg)

### & && | || 区别
`cmd1 操作符 cmd2 操作符 cmd3` 把这一整体称为一个命令

`&`：除了最后一个cmd，前面的cmd均已后台方式静默执行，执行结果显示在终端上，个别的cmd错误不影响整个命令的执行，全部的cmd同时执行
![&](https://tva1.sinaimg.cn/large/008eGmZEly1gnv8ul89vaj30vi09e0vc.jpg)

`&&`：从左到右顺序执行cmd，个别cmd错误不产生影响
![&&](https://tva1.sinaimg.cn/large/008eGmZEly1gnv8vf2h6yj30tu03ywfl.jpg)

`|`：各个cmd同时在前台被执行，但是除最后的cmd之外，其余的执行结果不会被显示在终端上
![|](https://tva1.sinaimg.cn/large/008eGmZEly1gnv8wgp2fcj30ka0420tr.jpg)

`||`：从左到右顺序执行cmd，只有左侧的cmd执行出错，右边的cmd才会被执行，同时一旦有cmd被成功执行，整个命令就会结束，返回终端
![||](https://tva1.sinaimg.cn/large/008eGmZEly1gnv8xabb6bj30qc04075c.jpg)

### 参考网址
- https://blog.csdn.net/w746805370/article/details/51044352  //linux shell编程中特殊字符
- https://www.cnblogs.com/unknown404/p/10355705.html //Linux shell中&，&&，|，||的用法
- https://www.cnblogs.com/x_wukong/p/5148237.html //linux命令执行返回值（附错误对照表）