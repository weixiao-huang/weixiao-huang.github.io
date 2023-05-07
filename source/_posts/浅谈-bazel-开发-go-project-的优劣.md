---
title: 浅谈 bazel 开发大型 go project 的优劣
date: 2020-04-25 22:22:11
tags: development
categories: Development
---

[Bazel](https://bazel.build/) 是 Google 开发的一款跨平台编译工具，是内部 blaze 的开源版本。本文不探讨 bazel 的使用，如果想了解如何用 Bazel 构建 go project，可以参考 [这篇文章](https://www.qtmuniao.com/2019/12/07/bazel-build-golang/)。本文主要探讨 Bazel 开发 go project 和纯使用 go modules 开发会有哪些优劣比较，我们如何去同时利用二者的优劣。

# 1. Bazel 开发的优势

## 1.1. 多语言支持

利用 Bazel 开发的 go project 能够支持非常好的可扩展性，对多语言的支持也比较友好，如果是一个公司内部的大项目，希望统一风格，统一 CI，统一构建，那么会存在这种状况：以 go 语言为主，其他语言为辅助。举一个例子，如果你的项目中包含了前端，那么 Bazel 的多语言处理就可以非常方便的让你同时编译 go project 和 frontend project。同理，如果你的项目中有 C++ 等其他语言也可以类似的用 Bazel 统一处理。这样在 monorepo 的管理上会方便很多。

## 1.2. 便于复用缓存和其他 bazel 的功能

bazel 自身是支持 remote cache 和 remote executor，个人认为 remote cache 功能是比较易用的一个功能。假如我们在 CI 中编译 go 项目，其实很难做到 go remote cache，基本上每次都得重新编译，但是 bazel 可以通过搭建 remote cache 服务重用缓存，极大的加大编译的速度。

另外，如果机器比较多，也可以使用 bazel 的 remote executor 服务搭建代码执行集群，让远端来跑 bazel 代码，这样也可以加速 go 的构建。不过实践下来一般 remote executor 可能不会比 remote executor 加速太多，除非专门对 build cluster 做过优化。

## 1.3. 镜像打包便捷而可复现

如果是一个 go modules 项目，那么镜像打包我可能就得写这么一个 `Dockerfile`

```dockerfile
FROM golang:1.14.2

ENV GOPATH=/go
ENV GOPROXY=https://goproxy.cn,direct
COPY . /go/src/my.project

WORKDIR /go/src/my.project
RUN go build -o main ./

FROM busybox

COPY --from=0 /go/src/my.project/main /usr/local/bin

CMD ["main"]
```

可以看到，这个 `Dockerfile` 的问题在于，在 ci 上每次编译都会导致重新编译，不会有任何的缓存，这就回到了上一个问题——无法复用缓存。

但是如果使用 Bazel，我们就可以使用 Bazel rules_docker 的 `container_image` 和 `container_push` 方法，实现镜像的打包和上传。而且 rules_docker 本身是不需要有 docker 环境的，**因此我们的 ci 甚至不用装 docker，只需要一个 bazel 就行了**。

## 1.4. 第三方包引用和更改比较方便

我自己并不是一个 go modules 深度用户，所以假如我们引用了一个第三方包，要对第三方包打 patch，就会比较麻烦，当然可以说，我们可以使用 vendor，但这并不符合 go modules 的哲学，那这样其实就会比较麻烦。

bazel 利用 gazelle 把所有第三方包显式的声明在 WORKSPACE 中，如果需要改动，只需给这个包加上 `.patch` 文件的 patch 即可实现更改，这样其实比起 vendor 来说更改也更容易被记录。

# 2. Bazel 开发相较于 go modules 的劣势

## 2.1 IDE 提示没有 go modules 友好

众所周知，go modules 很多 IDE 有非常好的支持，例如 vscode 和 goland，对于跳转等待支持得也非常好。但是 Bazel 就难办了，目前我尝试下来，只有 goland（或者 Jetbrains 系列） 的 bazel 插件是比较完善的，跳转基本能支持。

另一个方面，由于 Bazel 的尿性，每次本地执行 bazel build 会删除某些不必要的执行结果，因此可能会导致本地自己执行 bazel build 等指令之后，需要在 goland 中重新 sync 一下，这样会造成一些时间成本。

## 2.2 Bazel 依赖管理错乱

go modules 有自己一套依赖管理，但是 Bazel 的依赖，很多时候可能都是无脑 gazelle update 添加依赖，不会考虑依赖之间的版本关系，因此会导致 Bazel 项目中的版本依赖很错乱。

## 2.3 一些 go modules 相关的工具无法支持

一个例子就是 [golangci-lint](https://github.com/golangci/golangci-lint) 这个包。这个东西是不支持 Bazel 的，因此只有 go modules 项目可以支持。

## 2.4 有一定的学习成本

对于团队中的同学来说，如何使用 Bazel，如何添加依赖，是有一定的学习成本的。

# 3. 同时支持 Bazel 和 go modules

可以看到二者有他的优劣，我们能否合并二者，取长补短，发挥二者的优势呢？答案是肯定的。

我们可以以 bazel 为主，go modules 为辅。以下针对上面说的 Bazel 3 个劣势来说明如何解决。

对于第一个劣势，虽然我个人觉得 goland 的 Bazel 插件其实还行，但是如果实在不能忍受，可以直接使用我们提供的 go.mod 来基于 go modules 开发。

对于第二个劣势，鉴于 bazel-gazelle 的 `update` 命令有 `--from-file=go.mod` flag，使得我们可以从 go.mod 中生成 Bazel 版本依赖，这样就可以保证 Bazel 的依赖管理和 go modules 的依赖管理是一致的，不会使得依赖管理错乱，解决了上面的第二个问题。这个技巧在 k8s [repo-infra 的 update-deps.sh](https://github.com/kubernetes/repo-infra/blob/v0.0.3/hack/update-deps.sh) 中有使用。

对于第三个劣势，我们可以书写一些 Bazel 脚本，在 Bazel 的沙箱中使用 go modules 模式来使用 golangci-lint，这个方式在 k8s [repo-infra 的 verify-golangci-lint.sh](https://github.com/kubernetes/repo-infra/blob/v0.0.3/hack/verify-golangci-lint.sh) 中有使用。

对于第四个劣势，就要求项目维护者提供详尽的文档说明，如何 update 依赖，如何处理 Bazel 的一些问题，如何优化项目结构。当然我认为这并不是太大的问题，因为对于一个大项目来说，规范是必须的。

另外，k8s 有使用 Bazel 来构建 go projects，他们也采用的是 go modules 和 Bazel 同时支持的方案来实现。这样对于开发者也非常友好，同时对于大项目的管理、构建和测试也非常的友好。

由此，我书写了一个 [bazel-go-scaffold](https://github.com/weixiao-huang/bazel-go-scaffold.git)，基于 [repo-infra](https://github.com/kubernetes/repo-infra.git) 扩展了一个基于 Bazel 的 go 大项目模板，供大家参考