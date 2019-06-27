---
title: 在win10 WSL中安装oh-my-zsh遇到的坑
date: 2019-06-26 12:39:44
categories:
tags:
---

按照官网指南，在wsl的bash中执行命令：
> sh -c "$(wget -O- https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

出现大量如下错误：
> command not found: ^M

原因是wsl中的git配置了 autocrlf = true，这样clone下来的代码换行符就被自动转换成了 CRLF，但shell并不识别这个windows特有换行符，解决办法是：
> git config --global core.autocrlf input

然后删除已下载的文件重新安装