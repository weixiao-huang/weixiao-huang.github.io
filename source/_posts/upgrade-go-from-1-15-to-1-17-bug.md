---
title: 尝试把 Golang 从 1.15 升级到 1.16 以上时遇到的一些坑
date: 2022-04-04 22:09:44
tags:
---

# 背景

之前项目中用的 go 版本是 1.15.8，期望能升级到 1.17.4。主要驱动是我们用了 [bazel](https://bazel.build)，然后想在 M1 Mac 上用 bazel 编译 go 项目，发现需要升级 [rules_go](https://github.com/bazelbuild/rules_go) 到最新版本，并且 Go 在 1.16 之前官方不提供编译好的 darwin-arm64 的二进制包。因此尝试把项目升级到了 1.17

# 问题

升级之后，发现一些奇怪的问题，后面我构造了一个详细的[最小复现例子](https://github.com/weixiao-huang/golang-setgroups-hang)。根据这个例子的 `README.md` 执行，我们发现 go 1.15 和 go 1.16 有了完全不同的两个行为。最终发现是 `syscall.Setgroups` 这个函数在 1.16、1.17 的行为和 1.15 的不一致，因此我给 go 提了一个 [issue](https://github.com/golang/go/issues/50113)。目前已被解决，经过测试，在 go 1.18 中已经不存在该问题。
