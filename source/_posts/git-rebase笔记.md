---
title: git rebase笔记
date: 2019-02-07 19:03:05
tags:
---

### 一句话
重放一些提交到指定的分支，同时当前分支将重置到覆盖后的最新提交，导致原有的提交没有指针引用，最终被回收掉。

### 命令格式
`git rebase [--onto <newbase>] [<upstream> [<branch>]]`
参数含义：
* newbase：提交重放的分支，当没指定`--onto`时，用`<upstream>`代替，不一定是分支名，可以是任意有效提交。
* upstream: 与branch进行比较求差异的分支，默认使用当前分支的远程分支，不一定是分支名，可以是任意有效提交。
* branch: 被移动的分支，没指定则时当前分支HEAD，可以是任意有效提交

> 注意一下upstream和branch的关系，它们都是可选的，但branch参数的前提是有upstream


### 举例子：
1. `git rebase`：把当前本地分支rebase到同一分支的远程分支上。
* `newbase`: upstream
* `upstream`: 当前分支的远程分支
* `branch`: 当前分支HEAD
  
2. `git rebase master`：把当前分支与master不同的部分rebase到master上。
* newbase: upstream = master
* upstream: master
* branch: 当前分支HEAD

3. `git rebase master feature`：把feature分支与master不同的部分rebase到master上。
* newbase: upstream = master
* upstream: master
* branch: feature

4. `git rebase --onto master topicA topicB`：把topicA与topicB不同的部分rebase到master上。
* newbase: master
* upstream: topicA
* branch: topicB

### 工作流：
1. 如果指定了`<branch>`则git checkout到目标分支，否则不动作
2. 如果`<upstream>`没指定，则使用branch.<name>.remote的值。所有当前分支<branch>有，而<upstream>没有的提交将暂存起来。也就是<upstream>…HEAD
3. 当前分支`<branch>`做git reset --hard <newbase/upstream>（效果相同）
4. 先前保存的改动逐个应用到当前分支(已经移动到newbase或upstream了)
5. 如果有冲突，解决冲突git add，然后git rebase --continue
	
### 常规使用：
合并两个分支步骤：
举例：feature, master 合并feature到master上
git checkout feature
git rebase --interactive master

出现编辑界面，可以决定顺序，操作等，保存退出编辑界面
git将依次把提交合并到master的最新结点后，如果在解决冲突时不是完全用的feature的代码，那么后续的提交都需要解决冲突。
解决完有冲突的文件后
git add xxx
git rebase --continue

完成后，feature分支的新提交将合并到master后，master可以fast forward合并到feature上。
