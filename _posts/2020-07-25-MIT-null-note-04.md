---
title: "学习笔记：The Missing Semester of Your CS Education（4）"
tags: shell MIT
---

# 学习笔记：The Missing Semester of Your CS Education（4）

### 数据处理

从一种数据格式到另一种数据格式，使用linux的过程中，我们经常需要不断对数据进行处理，直到变成想要的结果。

首先是两条基础命令

```shell
journalctl
ssh myserver journalctl
```

`jornalctl`可以显示本次启动的所有日志，可能直观上看起来不像数据，但是通过数据处理，我们能按照意愿提取日志中的信息，例如查看最近的登录用户。

`ssh`则可以将远程主机的文件传输到本地环境。

#### 管道符

最常用也相对简单的数据处理手段，顾名思义，使用管道对数据进行传输，将其教给下一个程序处理。

```shell
ssh myserver journalctl | grep sshd
ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' | less
```

grep是最常用的处理程序，用来查找文件里符合条件的字符串。

[具体用法]: https://www.runoob.com/linux/linux-comm-grep.html	"grep"

前面的命令过滤结果中仍然包含很多无用信息，有很多手段可以帮助我们滤除信息，首先看sed。

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed 's/.*Disconnected from //'
```

sed是个基于文本编辑器构建的“流编辑器”。通过`sed`，我们可以使用一些简短的命令来修改文件，或者直接操作文件内容。相关命令行很多，但是最常用的是`s`。

`sed 's/.*Disconnected from //'`这条命令中使用了一个简单的正则表达式匹配。s命令的语法如下：`s/REGEX/SUBSTITUTION/`其中`REGEX`部分是需要使用的正则表达式，而`SUBSTITUTION`是用于替换匹配结果的文本。

PS1：日志是很大的文件，把这么大的文件直接传输是对流量的浪费，因此采用另一种方式，先在远端机器过滤文本，然后把过滤后的内容传输回主机：`ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' > ssh.log`。

PS2：使用`less`创建一个文件分页器，让我们可以通过翻页的方式浏览长文本，而不是一行行往下拉：`ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' | less`。

### 正则表达式

非常常见，非常有用，但是想写好很难，这大概是对正则表达式比较准确的描述了。正则表达式中大部分ASCII字符都表示他们本身的含义，但有一些字符具有特殊意义：

- `.` 除空格之外的”任意单个字符”
- `*` 匹配前面字符零次或多次
- `+` 匹配前面字符一次或多次
- `[abc]` 匹配 `a`, `b` 和 `c` 中的任意一个
- `(RX1|RX2)` 任何能够匹配`RX1` 或 `RX2`的结果
- `^` 行首
- `$` 行尾

回到范例`/.*Disconnected from /`这个表达式可以匹配任何以若干任意字符开头，接着包含“Disconnected from”的字符串，但是`+`和`*`在默认情况下是贪婪匹配，会匹配尽可能多的字符，如果有人把"Disconnected from"作为自己的用户名，运行结果如下：

```shell
匹配前
Jan 17 03:13:00 thesquareplanet.com sshd[2631]: Disconnected from invalid user Disconnected from 46.97.239.16 port 55920 [preauth]
匹配后
46.97.239.16 port 55920 [preauth]
```

可以在`*`和`+`后面添加一个`?`来使其变为非贪婪模式，不过sed不支持这种语法。使用sed的做法：

```shell
| sed -E 's/.*Disconnected from (invalid |authenticating )?user .* [^ ]+ port [0-9]+( \[preauth\])?$//'
```

开始的部分和以前是一样的，随后匹配两种类型的“user”（在日志中基于两种前缀区分）。再然后匹配属于用户名的所有字符。接着，再匹配任意一个单词（`[^ ]+` 会匹配任意非空切不包含空格的序列）。紧接着后面匹配单“port”和它后面的一串数字，以及可能存在的后缀`[preauth]`，最后再匹配行尾。

日志的内容全部被替换成了空字符串，整个日志的内容因此都被删除了。如果想保留用户名，可以使用“capture groups”来完成，被圆括号内的正则表达式匹配到的文本，都会被存入一系列用编号区分的捕获组中，被捕获的内容可以在替换字符串时使用，例如`\1`，`\2`，`\3`等等。

```shell
| sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
```

为了完成某种匹配，对应的正则表达式可能会非常复杂。这里有一篇关于如何匹配电子邮箱地址的文章[e-mail address](https://www.regular-expressions.info/email.html)，匹配电子邮箱可一点[也不简单](https://emailregex.com/)。网络上还有很多关于如何匹配电子邮箱地址的[讨论](https://stackoverflow.com/questions/201323/how-to-validate-an-email-address-using-a-regular-expression/1917982)。人们还为其编写了[测试用例](https://fightingforalostcause.net/content/misc/2006/compare-email-regex.php)及 [测试矩阵](https://mathiasbynens.be/demo/url-regex)。您甚至可以编写一个用于判断一个数[是否为质数](https://www.noulakaz.net/2007/03/18/a-regular-expression-to-check-for-prime-numbers/)的正则表达式。

#### 回到数据处理

现在我们的表达式如下：

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
```

该命令可以获得一个用户名列表，表内的用户都曾经尝试过登录。

过滤出最常见的：

```shell
| sort | uniq -c
```

`sort` 会对其输入数据进行排序。`uniq -c` 会把连续出现的行折叠为一行并使用出现次数作为前缀。我们希望按照出现次数排序，过滤出最常登陆的用户：

```
| sort -nk1,1 | tail -n10
```

