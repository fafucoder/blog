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
- $() 和 \` ` 是命令替换
- ${} 是参数替换
- $_ 在此之前执行的命令或者脚本的最后一个参数

### 常见括号区别

- [ ]: 用于条件判断，判断对象包括文件类型和赋值比较 [ 4 -eq 3 ]
- [[ ]]: 跟[ ]基本一致，也是用于条件判断， 但是也有点区别，[[]] 支持正则表达式比较，且逻辑运算符不一致，"[[]]"为"&&"、"||"，"[]"为"-a"、"-o"。"[[]]"支持逻辑短路，而"[]"不支持， "[[]]"为一个keyword，同括号与表达式中间必须要有空格进行隔离，"[[]]"中使用比较符时不能转义，同时不会出现Word-Splitting

### 常见的判断类型

#### 文件判断

- `[ -a file ]`：如果 file 存在，则为`true`。
- `[ -b file ]`：如果 file 存在并且是一个块（设备）文件，则为`true`。
- `[ -c file ]`：如果 file 存在并且是一个字符（设备）文件，则为`true`。
- `[ -d file ]`：如果 file 存在并且是一个目录，则为`true`。
- `[ -e file ]`：如果 file 存在，则为`true`。
- `[ -f file ]`：如果 file 存在并且是一个普通文件，则为`true`。
- `[ -g file ]`：如果 file 存在并且设置了组 ID，则为`true`。
- `[ -G file ]`：如果 file 存在并且属于有效的组 ID，则为`true`。
- `[ -h file ]`：如果 file 存在并且是符号链接，则为`true`。
- `[ -k file ]`：如果 file 存在并且设置了它的“sticky bit”，则为`true`。
- `[ -L file ]`：如果 file 存在并且是一个符号链接，则为`true`。
- `[ -N file ]`：如果 file 存在并且自上次读取后已被修改，则为`true`。
- `[ -O file ]`：如果 file 存在并且属于有效的用户 ID，则为`true`。
- `[ -p file ]`：如果 file 存在并且是一个命名管道，则为`true`。
- `[ -r file ]`：如果 file 存在并且可读（当前用户有可读权限），则为`true`。
- `[ -s file ]`：如果 file 存在且其长度大于零，则为`true`。
- `[ -S file ]`：如果 file 存在且是一个网络 socket，则为`true`。
- `[ -t fd ]`：如果 fd 是一个文件描述符，并且重定向到终端，则为`true`。 这可以用来判断是否重定向了标准输入／输出／错误。
- `[ -u file ]`：如果 file 存在并且设置了 setuid 位，则为`true`。
- `[ -w file ]`：如果 file 存在并且可写（当前用户拥有可写权限），则为`true`。
- `[ -x file ]`：如果 file 存在并且可执行（有效用户有执行／搜索权限），则为`true`。
- `[ file1 -nt file2 ]`：如果 FILE1 比 FILE2 的更新时间最近，或者 FILE1 存在而 FILE2 不存在，则为`true`。
- `[ file1 -ot file2 ]`：如果 FILE1 比 FILE2 的更新时间更旧，或者 FILE2 存在而 FILE1 不存在，则为`true`。
- `[ FILE1 -ef FILE2 ]`：如果 FILE1 和 FILE2 引用相同的设备和 inode 编号，则为`true`。

#### 字符串判断

- `[ string ]`：如果`string`不为空（长度大于0），则判断为真。
- `[ -n string ]`：如果字符串`string`的长度大于零，则判断为真。
- `[ -z string ]`：如果字符串`string`的长度为零，则判断为真。
- `[ string1 = string2 ]`：如果`string1`和`string2`相同，则判断为真。
- `[ string1 == string2 ]` 等同于`[ string1 = string2 ]`。
- `[ string1 != string2 ]`：如果`string1`和`string2`不相同，则判断为真。
- `[ string1 '>' string2 ]`：如果按照字典顺序`string1`排列在`string2`之后，则判断为真。
- `[ string1 '<' string2 ]`：如果按照字典顺序`string1`排列在`string2`之前，则判断为真。

#### 整数判断

- `[ integer1 -eq integer2 ]`：如果`integer1`等于`integer2`，则为`true`。
- `[ integer1 -ne integer2 ]`：如果`integer1`不等于`integer2`，则为`true`。
- `[ integer1 -le integer2 ]`：如果`integer1`小于或等于`integer2`，则为`true`。
- `[ integer1 -lt integer2 ]`：如果`integer1`小于`integer2`，则为`true`。
- `[ integer1 -ge integer2 ]`：如果`integer1`大于或等于`integer2`，则为`true`。
- `[ integer1 -gt integer2 ]`：如果`integer1`大于`integer2`，则为`true`。

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