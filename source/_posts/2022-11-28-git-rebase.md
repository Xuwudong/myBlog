---
title: git-rebase
date: 2022-11-28 22:28:36
tags:
- git
categories:
- git
---
### rebase 
> git rebase master 表示将当前分支rebase到master，即以master为基地

1. 例如 伙伴x 的a分支提交记录： 1 --> 2 --> 3, master分支提交记录：1 --> 2 --> 4 ,在a分支执行git rebase master后
a分支的提交记录将变成 1 --> 2 --> 4 -->3'(3'与3不是同一个提交，但是提交的内容相同，提交id不同)
2. 这时如果另一个伙伴 y 也在a分支上开发，并且本地提交记录为 1 --> 2 --> 5,而且将本地提交记录push到了远程的a分支。
3. 如果x 将rebase后的a分支push到远程a,因为远程a有x本地a没有的提交记录5，系统此时会提示x需要git pull --rebase(变基) 或者 --no-rebase（合并),
如果此时x直接git push --force 强推上去，远程a就会变成 1 --> 2 --> 4 --> 3'(5没有了)。
4. 伙伴y 接着git pull远程 a,因为远程a有y本地a分支没有的提交记录4、3'，系统此时也会提醒选择 --rebase 还是 --no-base
   * 假如选择git pull --rebase 本地分支提交记录将会变成 1 -- > 2 --> 4 --> 3' （**本地5提交记录也没了**），**因此会丢代码**
   * 假如选择git pull --no-rebase 本机分支提价记录将会变成 1 --> 2 --> 3'--> 4'-->5 -->merge commit( 3'--> 4'-->5 按照提交时间排序)，**不会丢代码**

因此实际开发中，如果有两个同事在同一个分支开发，**一定不能进行git push --force强推**
另外，为了避免别人强推导致可能丢失代码，建议拉代码时选择**git pull --no-rebase**(合并)

--force-with-lease 将解决这种安全问题
使用了 --force-with-lease 参数之后，上面那种安全问题就没有那么危险了。

使用此参数推送，如果远端有其他人推送了新的提交，那么推送将被拒绝