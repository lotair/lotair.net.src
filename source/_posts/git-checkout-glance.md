---
title: Git checkout浅析
date: 2017-04-19 16:14:17
author: sel-fish
tags:
  - git
  - strace
  - hlink
categories:
  - fun
---

我使用[`hexo`](https://hexo.io/)来搭建[博客](https://github.com/sel-fish/sel-fish.net.src)，并且使用[`next`](https://github.com/iissnan/hexo-theme-next)主题。不想把`next`主题相关的源码加入到博客的`git仓库`里面，只想保存`next`的配置文件`themes/next/_config.yml`。于是建立了一个`硬链接`到`git仓库`的根目录下，命令如下：

```
ln themes/next/_config.yml next.yml
```

这样，每次修改`themes/next/_config.yml`，都可以把对应的修改保存到`git仓库`中。然而对`next.yml`执行`git checkout`之后，`魔法`就“失效”了……

<!-- more -->

## 溯源

没有执行`git checkout`之前，可以看到建立`硬链接`之后，两个文件的`inode号`是一样的：

```
ls -i next.yml themes/next/_config.yml

1792890 next.yml  1792890 themes/next/_config.yml
```

对`next.yml`执行了`git checkout`之后，可以看到`next.yml`的`inode号`产生了变化，也就是说创建了一个新的文件：

```
git checkout next.yml; ls -i next.yml themes/next/_config.yml

130739 next.yml  1792890 themes/next/_config.yml
```

使用`strace`观察`git checkout next.yml`执行了哪些系统调用：

```
strace git checkout next.yml 2>&1 |grep next

execve("/usr/bin/git", ["git", "checkout", "next.yml"], [/* 27 vars */]) = 0
lstat(".git/next.yml", 0x7ffe7def1a30)  = -1 ENOENT (No such file or directory)
lstat(".git/refs/next.yml", 0x7ffe7def1a30) = -1 ENOENT (No such file or directory)
lstat(".git/refs/tags/next.yml", 0x7ffe7def1a30) = -1 ENOENT (No such file or directory)
lstat(".git/refs/heads/next.yml", 0x7ffe7def1a30) = -1 ENOENT (No such file or directory)
lstat(".git/refs/remotes/next.yml", 0x7ffe7def1a30) = -1 ENOENT (No such file or directory)
lstat(".git/refs/remotes/next.yml/HEAD", 0x7ffe7def1a30) = -1 ENOENT (No such file or directory)
lstat("next.yml", {st_mode=S_IFREG|0775, st_size=12951, ...}) = 0
lstat("next.yml", {st_mode=S_IFREG|0775, st_size=12951, ...}) = 0
unlink("next.yml")                      = 0
open("next.yml", O_WRONLY|O_CREAT|O_EXCL, 0777) =
```

可以看到确实调用了`unlink`删除，并调用了`open`新建了文件。

## 其它

在`macOS Sierra`上，对应到`strace`的工具是[`dtruss`](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dtruss.1m.html)。如果使用时遇到如下报错：

```
sudo dtruss df -lh

dtrace: system integrity protection is on, some features will not be available
dtrace: failed to initialize dtrace: DTrace requires additional privileges
```

需要先关闭`system integrity protection`，在`Recovery OS`状态下调用：

```
csrutil enable --without dtrace
```

## 再其它

这段故事到这里就结束了。但是，关于`git`的代码级别的解释还没有，`.git`下面的各个目录和文件的含义还是不清楚，还需继续了解。

## 参考

- https://apple.stackexchange.com/questions/208478/how-do-i-disable-system-integrity-protection-sip-aka-rootless-on-os-x-10-11
- https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dtruss.1m.html






