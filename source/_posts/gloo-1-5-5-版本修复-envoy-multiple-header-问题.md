---
title: gloo 1.5.5 版本修复 envoy multiple header 问题
date: 2021-07-21 00:02:58
tags: gateway
categories: Gateway
---

# 问题背景

[gloo](https://github.com/solo-io/gloo) 是一个开源的基于 envoy 的云原生 api 网关，其用 K8S 的 CRD 对于网关路由表等进行了一些封装，使得能通过云原生的方式配置 api 网关，提供灵活的 gateway 网络管控。

我们在项目中使用 gloo 的过程中，将其作为最上层的网关，对接了一个 kubernetes apiserver 的 upstream，在对接过程中，遇到了 [这个问题](https://github.com/solo-io/gloo/issues/2983)，主要是我们发现 envoy 的 ext_authz 模块不支持把 append response header，只支持 set response header，然而 kube-apiserver 有一个 [--requestheader-group-headers](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy) 的概念，通过这个 Header 能读到用户的 Groups，但是如果采用 envoy 的 ext_authz 模块的话，就会导致最终只读到一个 Group，而不能拿到用户所有的 Groups。

# 解决方案

## 1. PR

我给 envoy 官方提了一个 [PR](https://github.com/envoyproxy/envoy/pull/11158)，目前已经合并，这个 PR 合并之后，我又给 gloo 项目提了一个 [PR](https://github.com/solo-io/gloo/pull/5726)，这个 PR 已经被合并了，gloo 1.10 版本终于支持了相关的功能。

## 2. 低版本打 patch

不幸的是，我们的线上环境还在用 gloo 1.5.5 版本，还没有相关的功能，因此需要手动给 1.5.5 版本打一个小 patch。

为了更方便的打 patch 和使用，我们就没有采取 PR 中的方式来做，而是直接没有考虑兼容性，直接给 Set header 的部分加了一个 append 功能，牺牲了兼容性，但是改起来更加方便了，具体的改动步骤如下

### 2.1 循线追踪

问题发生在 gateway-proxy 相关的 pod 中，因此我们需要改的是 gateway-proxy 的 image。即 `solo-io/gloo-envoy-wrapper` 这个镜像。而 `solo-io/gloo-envoy-wrapper` 的 base 镜像是 `solo-io/envoy-gloo`，这个镜像的主要源头在 [envoy-gloo](https://github.com/solo-io/envoy-gloo) 这个项目中，下面简单说一下如何查询并做代码改动：

1. 查看 https://github.com/solo-io/gloo/blob/v1.5.5/Makefile#L32 可以看出，`envoy-gloo` 镜像版本为 `quay.io/solo-io/envoy-gloo:1.16.0-rc4`
2. 查看 https://github.com/solo-io/envoy-gloo/blob/v1.16.0-rc4/bazel/repository_locations.bzl#L4 可以看出 envoy 版本用了 https://github.com/yuval-k/envoy 的 8f4d759cb53493159bcba921d6109eace48cb6da commit
3. 查看 [https://github.com/](https://github.com/envoyproxy/envoy/blob/8f4d759cb53493159bcba921d6109eace48cb6da/source/extensions/filters/http/ext_authz/ext_authz.cc#L164)[yuval-k](https://github.com/yuval-k/envoy)[/envoy/blob/8f4d759cb53493159bcba921d6109eace48cb6da/source/extensions/filters/http/ext_authz/ext_authz.cc#L164](https://github.com/envoyproxy/envoy/blob/8f4d759cb53493159bcba921d6109eace48cb6da/source/extensions/filters/http/ext_authz/ext_authz.cc#L164) 更改 setCopy 为 addCopy

### 2.2 更改 envoy 代码

通过更改 envoy 代码可以形成下面的 patch

```
diff --git a/source/extensions/filters/http/ext_authz/ext_authz.cc b/source/extensions/filters/http/ext_authz/ext_authz.cc
index d6eede1..bb10544 100644
--- a/source/extensions/filters/http/ext_authz/ext_authz.cc
+++ b/source/extensions/filters/http/ext_authz/ext_authz.cc
@@ -179,7 +179,7 @@ void Filter::onComplete(Filters::Common::ExtAuthz::ResponsePtr&& response) {
     ENVOY_STREAM_LOG(trace, "ext_authz filter added header(s) to the request:", *callbacks_);
     for (const auto& header : response->headers_to_set) {
       ENVOY_STREAM_LOG(trace, "'{}':'{}'", *callbacks_, header.first.get(), header.second);
-      request_headers_->setCopy(header.first, header.second);
+      request_headers_->addCopy(header.first, header.second);
     }
     for (const auto& header : response->headers_to_add) {
       ENVOY_STREAM_LOG(trace, "'{}':'{}'", *callbacks_, header.first.get(), header.second)
```

### 2.3 改动并编译 envoy-gloo 代码

#### 2.3.1 改代码

**把上面的 patch 放到 `bazel/external/envoy.patch`**，然后需要更改的代码如下：

```
diff --git a/bazel/repositories.bzl b/bazel/repositories.bzl
index 4128f75..129d716 100644
--- a/bazel/repositories.bzl
+++ b/bazel/repositories.bzl
@@ -28,6 +28,10 @@ def _repository_impl(name, **kwargs):
             (location["tag"], name),
         )
 
+    if "patches" in location:
+        kwargs["patches"] = location["patches"]
+        kwargs["patch_args"] = ["-p1"]
+
     if "commit" in location:
         # Git repository at given commit ID. Add a BUILD file if requested.
         if "build_file" in kwargs:
diff --git a/bazel/repository_locations.bzl b/bazel/repository_locations.bzl
index 6a6509b..99a7801 100644
--- a/bazel/repository_locations.bzl
+++ b/bazel/repository_locations.bzl
@@ -3,6 +3,7 @@ REPOSITORY_LOCATIONS = dict(
     envoy = dict(
         commit = "51faa5a74f8e19d914078a02acd7b19565b583e8",
         remote = "https://github.com/yuval-k/envoy",
+        patches = ["//bazel/external:envoy.patch"],
     ),
     inja = dict(
         commit = "4c0ee3a46c0bbb279b0849e5a659e52684a37a98"
```

#### 2.3.2 编译

建议在镜像中做编译。查看 https://github.com/solo-io/envoy-gloo/blob/v1.16.0-rc4/cloudbuild.yaml#L2 可以看到使用的镜像为 `envoyproxy/envoy-build-ubuntu:e7ea4e81bbd5028abb9d3a2f2c0afe063d9b62c0-amd64`，然后执行

```shell
git clone https://github.com/solo-io/envoy-gloo && cd envoy-gloo
git checkout v1.16.0-rc4
docker run --rm -it -v $PWD:/src -w /src \
  envoyproxy/envoy-build-ubuntu:e7ea4e81bbd5028abb9d3a2f2c0afe063d9b62c0-amd64 ./ci/do_ci.sh bazel.release
```

即可以开始编译。注意最好在 Linux 下编译，建议内存为 16Gi。

编译完毕之后，参考 https://github.com/solo-io/envoy-gloo/blob/v1.16.0-rc4/cloudbuild.yaml#L12 的命令得知 linux/amd64/build_release_stripped/envoy 是实际使用的二进制包，利用下述命令打镜像

```shell
cp -f linux/amd64/build_release_stripped/envoy ci/envoy.stripped
TAGGED_VERSION=v1.16.0-rc4-patch make docker-release
```

如果 docker build 太慢，可以考虑把 ci/Dockerfile 改为下述的形式，其中 1.16.0-rc4 根据版本来选择

```dockerfile
FROM quay.io/solo-io/envoy-gloo:1.16.0-rc4
ADD envoy.stripped /usr/local/bin/envoy
```

可以看到打了 `quay.io/solo-io/envoy-gloo:1.16.0-rc4-patch` 镜像，注意这里会自动执行 push 操作，其实是 push 不上去的，可以忽略 push 的报错。

### 2.4 打 gloo-envoy-wrapper 镜像

```shell
git clone https://github.com/solo-io/gloo && cd gloo
git checkout v1.5.5
ENVOY_GLOO_IMAGE=quay.io/solo-io/envoy-gloo:1.16.0-rc4-patch make gloo-envoy-wrapper-docker
docker tag quay.io/solo-io/gloo-envoy-wrapper:1.5.5 quay.io/solo-io/gloo-envoy-wrapper:1.5.5-patch
```

就可以得到 `quay.io/solo-io/gloo-envoy-wrapper:1.5.5-patch` 镜像了，这个镜像就可以用于启动 gateway-proxy 了。

# 总结

总的来说，本文主要目的没有讲述 gloo 的 PR 相关的工作，着重点讲述了 gloo gateway-proxy 镜像如何制作的，希望通过本文，让大家能了解到 gloo 和 envoy 直接的一些联系，以及 gloo 是如何在 envoy 上做了一些插件和编译操作。
