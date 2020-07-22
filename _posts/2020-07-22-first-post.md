---
layout: post
title: "git:FATAL: Unable to connect to relay host, errno=10061 解决方案"
date: 2020-07-22
---

#### git:FATAL: Unable to connect to relay host, errno=10061 解决方案

之前在执行git命令拉库的时候，报错如下：

```sh
Cloning into 'danxian-baiheng.github.io'...
FATAL: Unable to connect to relay host, errno=10061
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
```

搜了很久相关信息，终于找到了问题的根因：

使用`ssh git@github.com -vT`排查问题：

```
debug1: Reading configuration data /c/Users/.../.ssh/config
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: Executing proxy command: exec connect -S 127.0.0.1:2333 github.com 22
```

这下明眼人就看出来了，127.0.0.1：2333是v2ray的默认代理端口，之前在给ssh配代理的时候添加了这条规则，使用ssh方式进行git clone时，这条规则也起效了，然而我并没有启动v2ray，导致没有访问到正确的地址，按照上面的输出，去对应的config配置文件中找，果然看到了代理配置段。

删掉该段配置，就正常了。

```ssh
debug1: Connecting to github.com [13.229.188.59] port 22.
debug1: Connection established.
```

如果不删的话启动v2ray可能也能正常访问，但是我自己搭的v2ray代理服务器老被墙，还是买的现成的代理香啊。