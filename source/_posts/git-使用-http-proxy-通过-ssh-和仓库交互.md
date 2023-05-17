---
title: git 使用 http proxy 通过 ssh 和仓库交互
date: 2023-05-17 20:21:38
tags: development
categories: Development
---

在公司内网开发的时候，有时候会无法连接外网，公司提供了一个 http proxy，例如 `http://my.proxy:1087` 来访问外网。因此我们需要通过这个代理来访问 github 等仓库。

访问 github 有两种方式，一种是 https 的，另一种是 ssh 的。当我们使用 https 来访问 github 的时候，直接在命令行使用 `export http_proxy=http://my.proxy:1087 https_proxy=http://my.proxy:1087` 就可以直接使用 `ggit clone https://github.com/weixiao-huang/weixiao-huang.github.io.git` 来克隆相应的仓库。

但是当我们要使用 ssh 来 `git clone` 的时候，会发现无法使用。我们需要更改一下 ssh 的配置，方式如下：

更改 `~/.ssh/config` 添加下面的代码

```plain
Host  github.com
  User git
  ProxyCommand socat - PROXY:my.proxy:%h:%p,proxyport=1087
```

通过上述方式即可以通过代理连接上 github，使用 `git clone git@github.com:weixiao-huang/weixiao-huang.github.io.git`