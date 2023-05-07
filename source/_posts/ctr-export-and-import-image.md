---
title: "利用老版本 containerd 自带的 ctr 命令进行镜像 tag 操作"
date: 2022-03-05T10:36:12+08:00
tags: container
categories: Container
---

# 一、起因

在帮助客户做线上操作的时候，需要安装 [寒武纪卡官方的 K8S device-plugin](https://github.com/Cambricon/cambricon-k8s-device-plugin/tree/master/device-plugin) ，
按照 [daemonset yaml](https://github.com/Cambricon/cambricon-k8s-device-plugin/blob/v1.1.3/device-plugin/examples/cambricon-device-plugin-daemonset.yaml) 部署 device-plugin 之后，
发现 yaml 里写的 [cambricon-k8s-device-plugin:v1.1.3](https://github.com/Cambricon/cambricon-k8s-device-plugin/blob/v1.1.3/device-plugin/examples/cambricon-device-plugin-daemonset.yaml#L47) 根本 pull 不下来


```bash
$ docker pull cambricon-k8s-device-plugin:v1.1.3
Error response from daemon: pull access denied for cambricon-k8s-device-plugin, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```

后面发现这个镜像[需要自己打](https://github.com/Cambricon/cambricon-k8s-device-plugin/blob/v1.1.3/device-plugin/README.md#download-and-build)。但是其实之前有离职的同学曾经打过一个镜像，放在了一台寒武纪机器的本地 containerd image 中，因此我们没必要再打一次镜像了，最好就直接复用这个镜像，把这个镜像 push 到一个内网 registry 上供所有机器使用。

操作了一番之后发现 containerd 这个 ctr 操作和 docker 还是非常不一样的，一开始想尝试要不直接安装一个 [nerdctl](https://github.com/containerd/nerdctl) 得了，但是感觉这么小的一个需求没必要搞那么麻烦，因此就开始探究 ctr export 镜像并 push 的操作。

# 二、操作

[ctr](https://github.com/containerd/containerd/tree/main/cmd/ctr) 是 containerd 内置的一个查看镜像的命令，一般安装了 containerd 的话一定会安装 `ctr`。用 `ctr` 做镜像操作的话，需要用 `ctr image` 命令，或者方便一些，直接用 `ctr i`。如果查看镜像列表，就使用 `ctr i ls`。但其实我们执行了一下发现

```bash
$ ctr i ls
REF TYPE DIGEST SIZE PLATFORMS LABELS
```

并没有得到什么结果。这里主要是因为 `ctr` 里面有一个 `namespace` 的概念，用 `ctr --help` 可以看到这个参数

```bash
$ ctr --help
...
--namespace value, -n value  namespace to use with commands (default: "default") [$CONTAINERD_NAMESPACE]
...
```

我们的镜像是给 k8s 使用的，k8s 会使用 `k8s.io` 这个 `namespace` 下的镜像，因此我们需要加一个 `-n k8s.io`

```bash
$ ctr -n k8s.io i ls -q | grep cambricon-k8s-device-plugin:v1.1.3
172.1.1.34:5000/library/cambricon-k8s-device-plugin:v1.1.3
```

这里使用 `-q` 参数只打印了镜像名。然后我们找到了存在在这台寒武纪机器上的这个 `172.1.1.34:5000/library/cambricon-k8s-device-plugin:v1.1.3` 镜像。现在我们的目的是把这个镜像 tag 成 `docker-registry.i.demo/library/cambricon-k8s-device-plugin:v1.1.3`，其中 `docker-registry.i.demo` 姑且当做是内网 docker-registry 的域名地址。

经过查阅资料发现，`ctr` 在[后续版本合并了 `tag` 命令](https://github.com/containerd/containerd/pull/3388)，然而一个不幸的消息是我们的 `ctr` 版本是 [v1.2.6](https://github.com/containerd/containerd/tree/v1.2.6)，似乎并没有集成 `tag` 命令，于是只能另辟蹊径。

后来发现，我们可以尝试先把镜像 `export` 出来成为 tar 包，然后再 `import` 回去，`ctr` 在 import 镜像的时候支持重新写个名字。这样看来这么绕一下看起来是可行的：

```bash
$ ctr -n k8s.io i export cambricon-k8s-device-plugin-v1.1.3.tar 172.1.1.34:5000/library/cambricon-k8s-device-plugin:v1.1.3
$ ctr -n k8s.io i import cambricon-k8s-device-plugin-v1.1.3.tar --base-name docker-registry.i.demo/library/cambricon-k8s-device-plugin
unpacking docker-registry.i.demo/library/cambricon-k8s-device-plugin:v1.1.3 (sha256:4b67b0f9c5c7fee7ed80409b2df4f68d4fa5dd170af88e7281fab7c88cbbd62a)...done
$ ctr -n k8s.io i ls -q | grep docker-registry.i.demo
docker-registry.i.demo/library/cambricon-k8s-device-plugin:v1.1.3
```

可以看到非常成功，我们成功做了一个 tag 操作。虽然看起来比较蠢，似乎下载一个新版的 ctr 就可以实现。但是如果客户环境不通互联网的话，这类 hack 操作还是比较好的救命稻草。

之后我们就可以使用 `ctr i push` 把镜像 push 上去了

```bash
$ ctr -n k8s.io i push docker-registry.i.demo/library/cambricon-k8s-device-plugin:v1.1.3
```

大功告成