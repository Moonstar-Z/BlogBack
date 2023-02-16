---
title: Git 版本管理相关
tag: Git
categories: 技术
---

## Git常规操作



<!--more-->

1、我需要统计代码提交行数

```shell
git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
```

2、我在A分支上做了文件修改，此时我需要切换到B分支上，但是代码改没有写完，所以我不能commit那么我此时怎么在git上操作?

```shell
git status                     //查看目前的修改的文件

git stash                      // 保存现在修改的文件到一个新的临时本地分支 

git stash list                 //查看上面这种临时修改的分支

git stash apply
git stash apply stash@{2}      //恢复索引 stash@{2} 对应的暂存的修改，索引可以通过 git stash list 进行查看

git stash drop stash@{1}        //删除 stash@{1} 分支对应的缓存数据
git stash pop                   //将最近一次暂存数据恢复并从栈中删除

//恢复已经删除的文件

git checkout 8dafe1a7f634b163d45bd53c37dafda2c78dd6ec ~1 Samples/WorldShuttle/ui_worldshuttle_window.h
```

3、我需要修改历史的git log

```shell
git rebase -i HEAD~6 //查看历史的6条修改记录

将pick --> edit ： wq保存退出 ，此时分支信息会切换到....>R>:
    
git rebase --amend  //对历史的提价信息进行修改

git rebase --continue //从 ....>R> 分支合并到现在分支 b继续下一个修改的信息
```

4、我要查看分支上两个版本之间的差异

```shell
git diff head1 ~ head2

git branch -d
git push origin --delete
```

5、分支合并不想指针快速前进而是生成一个新的提交记录

```shell
git merge maset --no-ff
```

