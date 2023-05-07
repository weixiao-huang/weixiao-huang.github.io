---
title: 用 docker 搭建 ceph rbd 服务
date: 2019-09-01 09:32:17
tags: container
categories: Container
---

# 0. 安装 docker 和 docker-compose

安装 docker 请参考 
{% post_link 在国内利用-kubeadm-在-Ubuntu-中国安装单-master-k8s-集群 安装 docker 和 k8s 相关命令行工具 %}

同时参考 [daocloud 上的 docker-compose 安装文档](http://get.daocloud.io/#install-compose)，可以使用下述语句安装 docker-compose：

```bash
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` > /tmp/docker-compose
chmod +x /tmp/docker-compose
sudo mv /tmp/docker-compose /usr/local/bin/docker-compose
```

# 1. 利用 docker-compose 部署单机单 osd ceph

主要参考[官方 docker 镜像文档](https://hub.docker.com/r/ceph/daemon/)，但最新的 `ceph/daemon` 镜像似乎有点奇怪的 bug，导致 osd 起不来，因此，我们使用一个 2019年年初的镜像，已经 tag 成 `huangwx/ceph-daemon` 供使用。ceph 的 docker-compose 文件如下：

```yaml
# docker-compose-ceph.yaml
version: '3.7'
services:
  ceph-mon:
    image: huangwx/ceph-daemon
    network_mode: host
    privileged: true
    volumes:
    - ./ceph:/etc/ceph
    - cephdata:/var/lib/ceph
    container_name: ceph-mon
    command:
    - mon
    environment:
    - MON_IP=${MON_IP}
    - CEPH_PUBLIC_NETWORK=${CEPH_PUBLIC_NETWORK}
  ceph-mgr:
    image: huangwx/ceph-daemon
    network_mode: host
    privileged: true
    volumes:
    - ./ceph:/etc/ceph
    - cephdata:/var/lib/ceph
    container_name: ceph-mgr
    command:
    - mgr
    depends_on:
    - ceph-mon
  ceph-osd:
    image: huangwx/ceph-daemon
    network_mode: host
    privileged: true
    volumes:
    - ./ceph:/etc/ceph
    - cephdata:/var/lib/ceph
    container_name: ceph-osd
    command:
    - osd
    depends_on:
    - ceph-mgr
    environment:
    - OSD_TYPE=directory
    pid: host
volumes:
  cephdata: {}
networks:
  default:
    external: true
    name: host
```

在 `docker-compose-ceph.yaml` 的相同目录下新建一个 `ceph` 目录

注意设置 `MON_IP` 和 `CEPH_PUBLIC_NETWORK` 两个环境变量，两个环境变量分别为主机 IP 地址，允许访问 ceph 的网段。下面仅作为演示，MON_IP 和 CEPH_PUBLIC_NETWORK 请设置成你自己的：

```shell
export MON_IP=192.168.2.2 CEPH_PUBLIC_NETWORK=192.168.0.0/16
```

同时请确认 `./ceph` 目录下没有任何 ceph 的配置文件，然后启动 ceph：
```shell
docker-compose -f docker-compose-ceph.yaml up -d
```

启动后会在 `./ceph` 目录下生成下述3个 ceph 配置文件：

```
ceph.client.admin.keyring
ceph.conf
ceph.mon.keyring
```

启动成功之后，稍等片刻，然后执行下述命令 check ceph 是否正常

```
$ docker exec -it ceph-mon ceph -s
  cluster:
    id:     e3cdecce-16f7-4c0f-9909-ab94a5e7c1b5
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum new
    mgr: new(active)
    osd: 1 osds: 1 up, 1 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   2.9 GiB used, 1020 GiB / 1023 GiB avail
    pgs:
```

如果成功看到类似上述的返回，即说明成功。

# 2. 初始化 rbd 配置

接下来初始化 rbd 配置。由于是单 osd 单 monitor 集群，因此我们需要创建并设置 rbd:

```bash
docker exec -it ceph-mon ceph osd pool create rbd 64 64 replicated
docker exec -it ceph-mon ceph osd pool application enable rbd rbd
docker exec -it ceph-mon rbd pool init rbd
docker exec -it ceph-mon ceph osd pool set rbd size 1
```

即可成功初始化并设置 rbd，可以利用 `rbd ls` 命令 check 是否能够不卡住返回：

```shell
docker exec -it ceph-mon rbd ls
```

如果能够成功不卡住返回空值，说明 rbd 集群正常，执行 `ceph -s` 可以看到初始化了 1 个 pool 和 64 个 pgs：

```
$ docker exec -it ceph-mon ceph -s
  cluster:
    id:     e3cdecce-16f7-4c0f-9909-ab94a5e7c1b5
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum new
    mgr: new(active)
    osd: 1 osds: 1 up, 1 in

  data:
    pools:   1 pools, 64 pgs
    objects: 0  objects, 0 B
    usage:   2.9 GiB used, 1020 GiB / 1023 GiB avail
    pgs:     64 active+clean
```