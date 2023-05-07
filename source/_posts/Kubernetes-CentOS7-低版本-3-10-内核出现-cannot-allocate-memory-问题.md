---
title: Kubernetes CentOS 7 低版本 3.10 内核出现 cannot allocate memory 问题
date: 2023-03-09 17:05:21
tags:
---

在标准的 CentOS 7 系统中，标配的是 3.10 版本的 Linux kernel，在这个内核下搭建 Kubernetes 一版来说还是比较稳定的，但是当集群跑着跑着，创建的 Pod 多于一定数量的时候，就会发现新 Pod 再也创建不起来了，报错

```plain
Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:279: applying cgroup configuration for process caused \"mkdir /sys/fs/cgroup/memory/kubepods/burstable/pod7d54e3ab-2639-11ea-b794-801844e495bc/cba62d4976e90bfdc2fb7cf20a7c37dfbf4b2273742b78f27b6d9b5d15b284ce: cannot allocate memory\"": unknown
```

主要的关键字是 `cannot allocate memory`，当然这个问题网上有诸多解答，搜索这个问题其实也挺费劲，但是最终发现一个帖子写得不错，比较有参考意义，在此贴上链接： https://cloud.tencent.com/developer/article/2141496 。这篇文章详细的说明了这个问题发生的原因，也给出了解决方案。在这里总结一下解决方案：

- 方案一：升级内核到 4.x 之后能解决
- 方案二：更改机器 grub 配置禁用 kmem 并重启机器
- 方案三：更改 kubelet 代码并重新编译

我们团队经过尝试，发现方案一和方案二均能解决问题，但是方案三不一定能解决，因此在这里我会着重介绍下方案二。

方案二在上面的文章里也说了一些，这里可以再做一些补充。

首先执行

```bash
vim /etc/default/grub
```

将 `GRUB_CMDLINE_LINUX` 的配置

```plain
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline intel_iommu=on iommu=pt rootflags=prjquota"
```

末尾加上 `cgroup.memory=nokmem`，例如

```plain
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline intel_iommu=on iommu=pt rootflags=prjquota cgroup.memory=nokmem"
```

然后下面需要注意，根据机器启动方式不一样，需要生成的 grub 配置可能不一样。

先试用 `ls /sys/firmware/efi` 查看 `/sys/firmware/efi` 是否存在，如果存在的话，那么该机器是通过 UEFI 启动，则执行

```bash
cp /boot/efi/EFI/centos/grub.cfg /boot/efi/EFI/centos/grub.cfg.bak
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
```

如果上述目录不存在的话，则执行

```bash
cp /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak
grub2-mkconfig -o /boot/grub2/grub.cfg
```

执行完成之后，可以用重启机器。

机器重启成功之后，可以使用 `cat /proc/cmdline` 检查配置是否生效，如果里面有 `cgroup.memory=nokmem` 相关的配置项，则说明修改成功。