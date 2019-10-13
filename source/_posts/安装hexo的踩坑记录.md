---
title: 安装hexo的踩坑记录
date: 2019-01-26T04:41:46.411Z
tags: hexo
comment: on
---

hexo是一个静态页面渲染器，非常适合建博。于是我参照官网安装指引在win10 WSL中尝试安装了一下，遇到一个小坑记录如下：

	1. 首先安装nvm：
在wsl里运行：
{% blockquote %}
$ wget -qO- https://raw.github.com/creationix/nvm/v0.33.11/install.sh | sh
{% endblockquote %}

这一步是为了安装npm版本管理程序，方便安装npm，隔离多版本。

	2. 接着安装npm，运行：
> $ nvm install stable

	3. 通过npm安装hexo：
> $ npm install -g hexo-cli

三条命令运行都很顺利，但初始化博客目录却报错了：
> $ hexo init blog

报错找不到hexo命令，根据经验判断这是没有把命令加到PATH，以为需要重启WSL使PATH改动生效，结果还是不行，并且连npm命令也找不到了。于是转而寻找nvm的资料，看看npm安装的位置，发现安装到了：~/.nvm/versions/node/v11.8.0，hexo也在其中，说明安装正常，那为什么用不了呢？

突然灵机一动，会不会是shell非bash导致的呢？我使用的是zsh，查看~/.bashrc, ~/.profile，发现安装nvm时修改了~/.profile，但 .zshrc 没有导入之，于是增加一行：
. ~/.profile
保存重开WSL，果然 npm, hexo都可用了。

总结：Linux环境配置文件众多，/etc/profile，~/.profile，~/.bashrc等等，根据选用的shell不同，加载的配置也不用，多种配置文件有层级优先级的特点，当出现命令找不到时要想到这一点。

===================
update: 又装了一次，家里电信下载node慢到抓狂，在安装程序前在shell里运行  `export https_proxy=http://127.0.0.1:1080` 让https流量走代理，终于快起来了。。。

===================
update: 关于主题，参考教程使用git submodule，fork了一份到自己仓库，然后在博客根目录添加主题仓库地址为submodule。重建博客目录时在根目录运行:

```
git submodule init
git submodule update
```

要跟踪上游的主题更新，只需要在github中 new pull request，比较自己仓库分支和上游仓库master分支，然后直接新建并同意即可合并更新。

[参考资料1](https://lrscy.github.io/2018/01/26/Hexo-Github-Backup/)
[参考资料2](https://juejin.im/post/5c2e22fcf265da615d72c596)