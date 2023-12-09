---
title: commit合并
date: 2023-12-09 13:35:57
tags: git
categories: git
---

##  如何将多个commit合并为一个commit



> 场景说明

有时候，我们在本地提交代码时执行了多个无效的commit，比如因为多次扫描代码而产生的commit，这些commit不需要提交到origin。而最终提交到origin的commit只能有一个，因此，这时需要将多个commit合并为一个origin。
> 详细操作

## 1 . git log 查看commitId(hash)
![查看commitId](images/git/git_merge_1.png)

## 2 . 取origin的commitId作为原点，执行 `git rebase -i [commitId]`
![picture2](images/git/git_merge_2.png)

## 3.进入rebase vi界面，只保留第一行的pick，将其余的pick改为s，wq保存退出
![picture3](images/git/git_merge_3.png)


## 4.退出后自动进入commit message edit界面，这时将其他提交的message注释,wq后退出
## 5.在idea中查看git log 
![picture4](images/git/git_merge_4.png)

## 6.执行git push 操作
![picture5](images/git/git_merge_5.png)
