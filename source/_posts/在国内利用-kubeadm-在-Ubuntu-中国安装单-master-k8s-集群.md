---
title: 在国内利用 kubeadm 在 Ubuntu 中安装单 master k8s 集群
date: 2019-08-31 15:26:29
tags: kubernetes
categories: Kubernetes
---

本文主要参考 Kubernetes 官方文档 [Creating a single control-plane cluster with kubeadm (v1.16)](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)，但由于国内实在是访问 Google 困难，之前的教程都是从国内各种镜像中拉取 k8s 镜像，再 tag 成官方的，实在是不太优雅。好在 K8S 新版支持了指定镜像源，可以方便的指定国内源。

# 0. 安装 docker 和 k8s 相关命令行工具

## 0.1 安装 docker

利用 [daocloud](http://get.daocloud.io/#install-docker) 上的 docker 安装方法在国内安装 docker，由于默认会从官方拉，我们可以考虑利用 Aliyun 的镜像

```
curl -sSL https://get.daocloud.io/docker | sh -s -- --mirror Aliyun
```

然后配置 docker 加速器，这里我们选择网易的 docker 源，如果有其他国内源，你也可以自行更改

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://hub-mirror.c.163.com
```

根据提示重启 docker

```
sudo systemctl restart docker.service
```

## 0.2 安装 k8s 相关命令行工具——`kubelet`, `kubeadm`, `kubectl`

在这里我们利用阿里云的 ubuntu k8s 源下载 k8s 相关包

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo tee -a /etc/apt/sources.list.d/kubernetes.list > /dev/null <<EOT
deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOT

sudo apt install -y kubelet=1.16.2-00 kubeadm=1.16.2-00 kubectl=1.16.2-00
```

我们这里安装的是目前 K8S 最新的版本 `v1.16.2`

# 1. 初始化 Kubernetes

## 1.1 初始化 k8s master

K8s 1.11 支持了 `kubeadm` 传入配置文件，我们可以新建下述配置文件 `kubeadm.conf`，注意我们采用的是 k8s 1.16.1 的相关配置，这个版本支持的 etcd 镜像 tag 为 `3.3.15-0`，如果有必要，`dnsDomain` 可以改成自己需要的，这里我们采用默认的 `cluster.local`

如果你的 k8s 版本不一致，可以采用 `kubeadm config images list` 查看你的 k8s 所需要的 etcd 镜像 tag。

```yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
etcd:
  # one of local or external
  local:
    imageRepository: "registry.aliyuncs.com/google_containers" # origin is "k8s.gcr.io"
    imageTag: "3.3.15-0"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"    # for flannel
  dnsDomain: "cluster.local"
kubernetesVersion: "v1.16.2"
imageRepository: "registry.aliyuncs.com/google_containers" # origin is "k8s.gcr.io"
```

这里我们用到了阿里云的 k8s 镜像源。上面的 `kubernetesVersion` 写明自己的 kubeadm 版本，不然会去 google 官方请求版本，可能会卡住。然后运行

```bash
kubeadm init --config kubeadm.conf
```

即可初始化集群。

完成之后，初始化成功，会输出类似于下面的信息

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.17.49.197:6443 --token zq59ch.wrr80vejg49091mw --discovery-token-ca-cert-hash sha256:197810ac44d29e4f311a639b22f1dac7dced81604c489be390a1fa5cf0b96f8d
```

执行

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

最后，为了让 master 也可以调度，可以加入下面的命令：

```
kubectl taint node $(kubectl get nodes -o name) node-role.kubernetes.io/master-
```

## 1.2 添加网络配置

上述信息做好了之后，在 master 上加入 `flannel` 插件，flannel 插件版本我们目前采用[文档中针对 v1.16 的版本](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)，由于国内网络环境缓慢，建议下载该文件更改镜像之后再 kubectl apply：

```bash
curl https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml | sed 's/quay.io/quay.azk8s.cn/g' > /tmp/kube-flannel.yml
kubectl apply -f /tmp/kube-flannel.yml
```

注意上述命令只需要在主节点执行。

## 1.3 节点加入

在 `kubeadm init`成功后，命令行会显示 Join 命令，命令格式如下：

```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

但如果之后忘记了命令，可以用下述命令获取 `<token>`：

```
kubeadm token list
```

然后可以用下述命令获得 `<hash>`

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

