---
title: K8S 替换 CA 证书中遇到的一些坑
date: 2022-04-04 22:11:09
tags:
---

# 起因

我们在给客户搭建高可用 K8S 的时候，发现我们使用的 loadbalance 的 IP 填错了，导致在给高可用 K8S 集群加入 worker node 的时候报错无法连接上 loadbalance 的 IP，因此我们需要更改一下这个 IP，使用一个 worker node 能访问的 IP。

我们尝试改了 K8S 中和 IP 相关的配置之后，再次加入 worker node 却发现报错证书错误，因此涉及到需要用新的 IP 给 K8S 重新签署一整套证书的工作。

# 操作流程

需要注意的是，其实我们可以不重新签署 CA 证书，但是我们担心会在其他地方出现什么问题，因此利用脚本尝试把这个 K8S 用的整套证书都重新签署了一遍。

## 1. 签署所有证书

## 2. 重新生成 /etc/kubernetes 下的所有 conf 文件

先备份一下相关文件

```shell
mv /etc/kubernetes/admin.conf /etc/kubernetes/admin.conf.old
mv /etc/kubernetes/controller-manager.conf /etc/kubernetes/controller-manager.conf.old
mv /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.old
mv /etc/kubernetes/scheduler.conf /etc/kubernetes/scheduler.conf.old
```

然后执行下面命令生成新配置

```shell
sudo kubeadm init phase kubeconfig all --kubernetes-version=1.19.8
```

## 3. 重启所有 K8S 组件

## 3.1 重启管理 pod

在所有节点将 `/etc/kubernetes/manifests` 下的所有 `kube-*` 和 `etcd.yaml` 挪出来，然后再挪进去，达到重启所有 K8S pod 的目的

## 3.2 重启 kubelet 

此操作需要保证 `kube-apiserver` 重启成功之后再操作。由于 CA 证书改了，但 kubelet 用的还是之前老 CA 签署的证书，因此需要删除 kubelet 之前的证书，让 kubelet 重新生成一份。

先备份一下所有节点的 kubelet 之前的证书

```shell
mv /var/lib/kubelet/pki /var/lib/kubelet/pki.old
```

然后重启所有节点的 kubelet

```shell
systemctl restart kubelet
```

等待 kubelet 重新生成 pki 即可

## 4. 更改 configmap 中的证书配置

### 4.1 更改 cluster-info

```shell
kubectl -n kube-public edit cm cluster-info
```

将其中的 `certificate-authority-data` 换成新 CA 证书的 base64 值，将 `server` 换成新 IP 的地址

### 4.2 更改 kube-proxy 配置

```shell
kubectl -n kube-system edit cm kube-proxy
```

将 `kubeconfig.conf` 中 `clusters` 里的 `server` 改为新 IP

### 4.3 更改 kubeadm 配置

```shell
kubectl -n kube-system edit cm kubeadm-config
```

按需更改 `advertiseAddress` 地址为新 IP

# 后处理

由于 K8S CA 变了，导致所有 namespace 下的 serviceaccount token 均无效，因此需要把所有 namespace 下的 serviceaccount token 删除重建一下，同时需要删除所有用到 serviceaccount token 的 pod。

## 1. 删除 serviceaccount token

### 1.1 删除除了 kube-system namespace 之外的 sa token

利用 

```shell
kubectl get secret --field-selector type=kubernetes.io/service-account-token -A | grep -v kube-system
```

可以查到所有除了 kube-system 下 namespace 的 sa token，然后一个个删除，接着删除或者重启对应的 pod

### 1.2 删除 kube-system 下相关 sa token

执行

```shell
kubectl -n kube-system get secret --field-selector type=kubernetes.io/service-account-token
```

可以看到有一堆 token ，然后需要把这些 token 都删掉，删掉之后会重新生成他们
