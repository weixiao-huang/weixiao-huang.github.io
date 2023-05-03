---
title: Dockerfile 的 VOLUME 关键字会从镜像传递到 k8s pod 中
date: 2023-01-27 13:04:38
tags:
---

在书写 `Dockerfile` 的时候，会存在一个 `VOLUME` 的关键字，例如

```dockerfile
FROM busybox
VOLUME /mnt/data
```

我们如果将这个 `Dockerfile` 打成一个镜像

```bash
docker build -t test-image .
```

并将这个镜像在 Kubernetes 中创建一个 Pod 的话，当我们 exec 进入该 Pod，执行

```bash
/ # df -Th /mnt/data/
Filesystem           Type            Size      Used Available Use% Mounted on
/dev/sda1            xfs           199.9G     53.8G    146.1G  27% /local-rootfs
```

会发现 /mnt/data 被挂载了一个宿主机的临时目录，这种情况其实不是 Kubernetes 能控制的，其实是底层的容器运行时控制的，但这样的话就会导致平台无法控制用户镜像中的 VOLUME，可能会导致用户往对应 VOLUME 里疯狂写入数据造成宿主机磁盘写满而存在安全风险。鉴于现在 Kubernetes 官方推荐使用 containerd，所以我们来说一下针对 containerd 的场景该问题该如何解决。

为了解决该问题，我们需要在 containerd 的 `/etc/containerd/config.toml` 中使用 `ignore_image_defined_volumes = true` 去避免该问题，参考 [文档](https://github.com/kinvolk/containerd-cri/blob/master/docs/config.md) ，里面有段注释说明了该选项的功能

```toml
    # ignore_image_defined_volumes ignores volumes defined by the image. Useful for better resource
    # isolation, security and early detection of issues in the mount configuration when using
    # ReadOnlyRootFilesystem since containers won't silently mount a temporary volume.
```

具体实现的代码可以在 [这里](https://github.com/containerd/containerd/blob/v1.5.10/pkg/cri/server/container_create.go#L142) 找到。
