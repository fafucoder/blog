---
title: linux 三剑客 awk, grep, sed
date: 2020-09-20 21:40:22
tags:
- linux
categories:
- linux
---

### 概述
awk、grep、sed 是 linux 操作文本的三大利器， 三者的功能都是处理文本，但侧重点各不相同。

grep适合单纯的查找或匹配文本，sed适合编辑匹配到的文本，awk适合格式化文本，对文本进行较复杂格式处理。

简单概述为：
- grep：数据查找定位
- awk：数据切片
- sed：数据修改

### grep
grep(全局正则表达式打印)--命令用于查找文件里符合条件的字符串。 从文件的第一行开始，grep 将一行复制到 buffer 中，将其与搜索字符串进行比较，如果比较通过，则将该行打印到屏幕上。grep将重复这个过程，直到文件搜索所有行。

常见参数：
1. -n   --line-number         #在显示符合样式的那一行之前，标示出该行的列数编号。 `grep -n license package.json`
2. -v   --revert-match        #显示不包含匹配文本的所有行。 
3. -c   --count               #计算符合样式的列数。
4. -l   --file-with-matches   #列出文件内容符合指定的样式的文件名称。 `grep -l license *`   
5. -i   --ignore-case         #忽略字符大小写的差别。
6. -x   --line-regexp         #只显示全列符合的列。 `grep -x "boo" sampler.log`
7. -s   --no-messages         #不显示错误信息。 `grep -ls license *`
8. -H   --with-filename       #在显示符合样式的那一行之前，表示该行所属的文件名称。  

### awk
awk 程序对输入文件的每一行进行操作。它可以有一个可选的 BEGIN{ } 部分在处理文件的任何内容之前执行的命令，然后主{ }部分运行在文件的每一行中，最后还有一个可选的END{ }部分操作将在后面执行文件读取完成:
```
BEGIN { …. initialization awk commands …}
{ …. awk commands for each line of the file…}
END { …. finalization awk commands …}
```

对于输入文件的每一行，它会查看是否有任何模式匹配指令，在这种情况下它仅在与该模式匹配的行上运行，否则它在所有行上运行。 这些 'pattern-matching' 命令可以包含与 grep 一样的正则表达式。 

类似数据库列的概念，但它是按照序号来指定的，比如我要第一个列是1， 第二列是2，依此类推。$0就是输出整个文本的内容。默认用空格作为分隔符，可以自己通过-F设置适合自己情况的分隔符。 例如：`awk '{print $1}' demo.txt`

### sed
sed执行时，会从文件或者标准输入中读取一行，将其复制到缓冲区，对文本编辑完成之后，读取下一行直到所有的文本行都编辑完毕。

所以sed命令处理时只会改变缓冲区中文本的副本，如果想要直接编辑原文件，可以使用-i选项或者将结果重定向到新的文件中。

常见命令:
1. -e ：直接在命令列模式上进行 sed 的动作编辑。
2. -i ：直接修改读取的文件内容，而不是输出到终端。
3. -f ：直接将 sed 的动作写在一个文件内， -f filename 则可以运行 filename 内的 sed 动作。
4. -n ：使用安静(silent)模式。在一般 sed 的用法中，所有来自 STDIN 的数据一般都会被列出到终端上。但如果加上 -n 参数后，则只有经过sed 特殊处理的那一行(或者动作)才会被列出来。

其他命令:

一般形式是
```
sed -e '/pattern/ command' sampler.log
```

其中 'pattern' 是正则表达式，'command' 可以是:

- a(append) ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
- c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
- d(delete) ：删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
- i(insert) ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
- p(print) ：列印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
- s(search＆replace) ：取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法！例如 1,20s/old/new/g

示例：
1. 删除:
```
# 删除指定行(第三行)
sed -e '3d' demo.txt

➜  indigo git:(master) ✗ sed '2d' demo.txt  
Twinkle, twinkle, little star
Up above the world so high
Like a diamond in the sky
When the blazing sun is gone

# 删除范围行(2-4行)
sed -e '2,4d' demo.txt

➜  indigo git:(master) ✗ sed '2,4d' demo.txt
Twinkle, twinkle, little star
When the blazing sun is gone

# 到结尾行
sed -e '2,$d' demo.txt

➜  indigo git:(master) ✗ sed '3,$d' demo.txt
Twinkle, twinkle, little star
How I wonder what you are
```

### 参考文档
- https://cloud.tencent.com/developer/article/1465475
- https://juejin.im/post/6845166891493785613
