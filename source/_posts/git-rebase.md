---
title: git-rebase使用
date: 2020-03-28 23:35:18
tags:
- git
- rebase
---

### git rebase注意事项
- 注意不要合并先前提交的东西，也就是已经提交远程分支的纪录。

### git rebase的常用命令
```
# Rebase a639348..8c898e1 onto a639348 (2 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending  //修改commit信息
# s, squash = use commit, but meld into previous commit  //丢弃commit信息, 合并到上个commit中
# f, fixup = like "squash", but discard this commit's log message   //丢弃commit信息，日志不记录
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out

```

### 调整commit的位置
```
1. git rebase -i HEAD~4
2. 任意调整位置(vim ddp 调整位置)
3. 如果有冲突,解决冲突
4. git rebase --continue
```
### 合并commit(合并十个commit为一个)
```
1. git rebase -i HEAD~9
2. 把需要合并的标记为s/squash(最顶上不能标记为s)
3. 保存， 重新填写commit信息
4. 
```

### 修改commit信息(修改前两个的commit信息)
```
1. git rebase -i HEAD~2
2. 标记为e/edit
3. git commit --amend
4. git rebase --continue
```

### 参考文档
- http://jartto.wang/2018/12/11/git-rebase/   //git rebase使用实例