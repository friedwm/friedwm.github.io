---
title: 加速git clone
date: 2019-06-20 11:48:39
categories:
tags:
---

由于众所周知的原因，clone github上的项目非常慢，网上找到一些方案，利用ssr加速，亲测可用。

# Git支持的协议
git支持三种协议的项目地址
* http(s)协议，例如: https://github.com/....
* git协议，例如：git://github.com/...
* ssh协议，例如：git@github.com/...
三种协议配置不同，如下：

## http协议
> git config --global http.proxy "proto://user:password@proxy.server:port"

注意proto替换成实际的代理协议，支持：http(s), socks5, socks5h

## git协议
1. 安装connect-proxy
> sudo apt-get install connect-proxy
2. 建立文件:  /usr/bin/socks5proxywrapper
```bash
#!/bin/sh
connect -S 127.0.0.1:1081 "$@"
```

#### 注意
a. 文件要赋予可执行权限:
> chmod +x /usr/bin/socks5proxywrapper

b. 命令中的`-S`表示socks协议，-H表示http协议

3. 命令：
> git config --global core.gitproxy /usr/bin/socks5proxywrapper


## ssh协议
1. git协议中的1,2两步
2. 再建立一个文件  /usr/bin/socks5proxyssh

```
#!/bin/sh
ssh -o ProxyCommand="/usr/bin/socks5proxywrapper %h %p" "$@"
```
3. 导出环境变量：export GIT_SSH="/usr/bin/socks5proxyssh"

### 更简单的SSH协议方法
直接利用系统命令nc，在.ssh/config里添加配置

```
Host github.com
ProxyCommand nc -X 5 -x 127.0.0.1:1081 %h %p
```
注意config文件的权限, chmod 600 ~/.ssh/config，不用再装 connect-proxy，直接开始用把。
附上前后对比：
{% asset_img before.png 加速前 %}
{% asset_img after.png 加速后 %}



参考文档:
[1](https://gist.github.com/laispace/666dd7b27e9116faece6)