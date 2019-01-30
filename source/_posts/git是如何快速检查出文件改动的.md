---
title: git是如何快速检查出文件改动的
date: 2019-01-30 23:21:06
tags: git
---

### 问题的来源
`git status`可以说是最常使用的git命令，它不仅可以列出工作区的变动，如果你忘记怎么添加文件到index、想恢复工作区的改动也可以运行一下git status看看提示。
运行git status总是能很快返回变动的文件，它是如何做到这一点的？

### 监听操作系统通知？
我带着这个疑问请教了一个同事，他给出的答案是git会利用操作系统监听文件改动，这样git显然可以毫不费力获知文件，听起来很有道理。但接着我做了一个实验：先修改工作区但不提交，假设git通过操作系统获知有文件改动了，好没问题，接着把这个项目目录整个剪切到其他地方，这样的话所有文件都变动了，再运行git status发现还是能立即显示出改动的那个文件。而且git至少得先向操作系统注册回调才能收到通知吧，但实际只是在修改完成后运行了一次git status即可看到改动，所以基本可以排除回调这种机制，只剩下一种可能，git是在运行命令时，实时比较工作区与index的差别从而得到改动的文件。

### 修改时间?
git是通过文件的修改时间判断吗？毕竟时间相同应该不会改动，但是这还不够，因为系统时间是可改的，意味着只用这个时间不可靠。


### sha-1?
了解一点git原理的都知道，原文件被编码成二进制格式文件存入`.git/objects`，文件名是内容的*sha-1*值，那么在`git status`时，git是不是把所有文件hash一遍与index中记录进行比较呢？应该能行，但是效率太低了。


### 真相
经过一番搜索，找到了一些线索：[stackoverflow](https://stackoverflow.com/questions/4075528/what-algorithm-does-git-use-to-detect-changes-on-your-working-tree/4075667)，[git-update-index](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/git-update-index.html#_using_8220_assume_unchanged_8221_bit)。

大致意思是：git充分利用了系统调用lstat，index中记录了文件的lstat结果，文件大小、修改时间、inode等，在git status时，先通过文件的lstat结果与index中的比较，如果相同那么不用进一步处理就当做相同；对于那些不同的，再进一步通过内容sha-1比较是否相同来最终判别是否改动。