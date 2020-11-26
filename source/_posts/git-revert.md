---
layout: post
title: "Git 如何撤销分支的merge"
date: 2017-08-10
excerpt: "今天同事不小心把develop未测试的代码合并到了master，想取消这次合并，让分支回到位，肿么办？如果master上已经有了新的提交以及没有新的提交都该怎么办？"
categories: 
- Linux
tags: 
- Git
comments: true
---

### 首先回滚两种方式的区别

<b>revert</b>：是用一次新的commit回滚之前的commit,所以它的版本号不会变，这样就会导致在日后继续merge以前的老版本的时候，这些差异并不会显示，我们这次也有这个问题，master和develop对比差异的时候，并没有把回滚之后的差异显示出来。  

<b>reset</b>：是直接删除指定的commit，提交及之前的commit都会被保留，但是此次之后的修改都会被退回到暂存区。

举个例子：假设有三个commit， git log:  

```
commit3: add test3.c  
commit2: add test2.c  
commit1: add test1.c  
```
当执行git revert HEAD~1时， commit2被撤销了，git log可以看到：

```
revert "commit2":this reverts commit 5fe21s2...  
commit3: add test3.c  
commit2: add test2.c  
commit1: add test1.c
```
git status 没有任何变化，如果换做执行git reset --soft(默认) HEAD~1后，运行git log  

```
commit2: add test2.c  
commit1: add test1.c
```
运行git status， 则test3.c处于暂存区，准备提交。  
如果换做执行git reset --hard HEAD~1后，显示：HEAD is now at commit2，运行git log

```
commit2: add test2.c  
commit1: add test1.c
```
运行git log， 没有任何变化

### 怎么撤销megre呢？
<b>方法一</b> 如果没有新的提交可以用reset 到 merge 前的版本，然后再重做接下来的操作，要求每个合作者都晓得怎么将本地的 HEAD 都回滚回去：

```
$ git reset --hard 【merge前的版本号】
```
<b>方法二</b> 当 merge 以后还有别的操作和改动时，git 正好也有办法能撤销 merge，用 git revert：

```
$ git revert -m 【要撤销的那条merge线的编号，从1开始计算】 【merge前的版本号】
```
这样会创建新的 commit 来抵消对应的 merge 操作，而且以后 git merge 【那个编号所代表的分支】 会提示：

```
Already up-to-date.
```
因为使用方法二会让 git 误以为这个分支的东西都是咱们不想要的。

<b>方法三</b> 怎么撤销方法二

```
$ git revert 【方法二撤销merge时提交的commit的版本号，这里是88edd6d】
```

方法二里的megre线的编号怎么算，1：合并目标的分支 2：合并来源分支  
> 一般来说，如果你在master上merge zhc\_branch,那么parent 1就是master，parent 2就是zhc_branch.






