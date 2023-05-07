---
title: gloo 源码解析
date: 2023-05-07 11:03:30
tags:
---

[gloo](https://github.com/solo-io/gloo) 是一款基于 envoy 的云原生的API网关，它能够非常方便的和 K8S 进行集成，通过监听相关的 CRD，基于 envoy 的 [xDS](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) 接口对 envoy 配置进行 hot reload。同时也能[很方便的集成 knative](https://docs.solo.io/gloo-edge/1.7.23/installation/knative/)。另外它的文档也比较完善，设计比较简单，易于上手。

关于如何上手并使用 gloo，相信网上也能搜到一些中文资料，当然最好的方式当然是直接去 [gloo 官方文档](https://docs.solo.io/gloo-edge/latest/) 查看。本文主要深入 gloo 的源码，探究 gloo 是如何实现对 envoy 的配置下发的。

从最基本意义上讲，gloo 是转化引擎，Envoy xDS 服务器为 Envoy 提供高级配置（包括 gloo 的自定义 Envoy 过滤器）。gloo 遵循基于事件的体系结构，监视各种配置源以进行更新，并立即通过 v2 版本的 gRPC 接口更新 Envoy 配置。

本文基于 [gloo 1.14.0](https://github.com/solo-io/gloo/tree/v1.14.0) 对 gloo 源码进行分析。

## 核心代码结构

### gloo 的核心概念

整体上，gloo 的核心概念如下图所示：

{% diagramsnet "/images/gloo-code-structure.drawio" %}


可以看到，主要核心控制器有 3 个：`Emitter`、`EventLoop` 和 `Syncer`。其中 `Emitter` 实现了 `Snapshots()` 函数，这个函数会返回一个 `Snapshot` channel，然后在主循环 `EventLoop` 中会执行一个 `Run()` 函数去监听 `Snapshot` channel，一旦接收到新的 `Snapshot`，就会把这个 `Snapshot` 发送给 `Syncer` 实现的 `Sync()` 函数去处理，而这个 `Sync()` 函数就是最主要的将 gloo 配置同步到 envoy 的核心函数。

在上面我们可以频繁提到一个 `Snapshot` 的概念，这个就是 gloo 的最核心的数据结构。`Snapshot` 是一堆资源的集合，包括 gloo 的各种 CR 以及 K8S 的 Services 等资源，下面是 `ApiSnapshot` 的一个定义，可以参考[源码](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot.sk.go#L24)：

```go
type ApiSnapshot struct {
	Artifacts          gloo_solo_io.ArtifactList
	Endpoints          gloo_solo_io.EndpointList
	Proxies            gloo_solo_io.ProxyList
	UpstreamGroups     gloo_solo_io.UpstreamGroupList
	Secrets            gloo_solo_io.SecretList
	Upstreams          gloo_solo_io.UpstreamList
	AuthConfigs        enterprise_gloo_solo_io.AuthConfigList
	Ratelimitconfigs   github_com_solo_io_gloo_projects_gloo_pkg_api_external_solo_ratelimit.RateLimitConfigList
	VirtualServices    gateway_solo_io.VirtualServiceList
	RouteTables        gateway_solo_io.RouteTableList
	Gateways           gateway_solo_io.GatewayList
	VirtualHostOptions gateway_solo_io.VirtualHostOptionList
	RouteOptions       gateway_solo_io.RouteOptionList
	HttpGateways       gateway_solo_io.MatchableHttpGatewayList
	GraphqlApis        graphql_gloo_solo_io.GraphQLApiList
}
```

上面的 `ApiSnapshot` 里储存了很多东西，包括 gloo 自己的 CRD，例如 `UpstreamList`、`VirtualServiceList`、`RouteTableList` 等，也有 K8S 自己的资源（当然 gloo 在这里稍微做了一层转换），例如 `EndpointList`、`SecretList` 等。

`Snapshot` 的作用如下：
- gloo 会通过一些 informer 接收到 K8S 中相应的 CRD 信息，并把这些信息汇总到 `Snapshot` 中
- gloo 会在 `Snapshots()` 函数中[一秒轮训一次](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L882)，周期性的把 `Snapshot` 送入 `Emitter` 的 `Snapshots()` 接口返回的 `Snapshot` Channel 中

## 核心 Translate 流程

下图是一个总体的服务入口和 Translate 流程

{% diagramsnet "/images/gloo-translate.drawio" %}

## gloo 如何把 K8S 的资源汇总转换到 Snapshot 中

在这里我们主要查看一下 `Emitter` 的逻辑，下面是一个 [`apiEmitter` 的定义](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L131)

```go
type apiEmitter struct {
	forceEmit            <-chan struct{}
	artifact             gloo_solo_io.ArtifactClient
	endpoint             gloo_solo_io.EndpointClient
	proxy                gloo_solo_io.ProxyClient
	upstreamGroup        gloo_solo_io.UpstreamGroupClient
	secret               gloo_solo_io.SecretClient
	upstream             gloo_solo_io.UpstreamClient
	authConfig           enterprise_gloo_solo_io.AuthConfigClient
	rateLimitConfig      github_com_solo_io_gloo_projects_gloo_pkg_api_external_solo_ratelimit.RateLimitConfigClient
	virtualService       gateway_solo_io.VirtualServiceClient
	routeTable           gateway_solo_io.RouteTableClient
	gateway              gateway_solo_io.GatewayClient
	virtualHostOption    gateway_solo_io.VirtualHostOptionClient
	routeOption          gateway_solo_io.RouteOptionClient
	matchableHttpGateway gateway_solo_io.MatchableHttpGatewayClient
	graphQLApi           graphql_gloo_solo_io.GraphQLApiClient
}
```

可以看到 `Emitter` 中存了很多 `Client`，需要注意的是这些 `Client` 并不是 K8S 的 client-go，而是 gloo 自己封装的 client。

`Emitter` 的 [`Snapshots()` 方法](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#LL259C11-L259C11)会做以下事情

1. 初始化各 `Client` 并 watch 其中数据变化
2. 一旦 Watch 到数据，对数据进行部分处理，然后塞入临时的 `currentSnapshot` 结构体中
3. 每一秒一次，把 `currentSnapshot` 送入 `Snapshot` channel

下面针对这 3 个过程从源码进行解析

### 1. 初始化各 `Client` 并 watch 其中数据变化

第一步，[`apiEmitter` 的 `Register()` 函数](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L150)会调用各个 client 的 `Register` 接口，这些 `Register` 接口会对各个 client 对应的 K8S informer 进行初始化。这些 `Register` 会最终调用到 [`ResourceClient` 这个接口的 `Register()`](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/clients/client_interface.go#L34)，这部分代码已经不在 gloo 里了，是在 gloo 依赖的 solo-kit 中。然后这个 `Register()` 最重要的是实现方式在 [kube/resource_client.go](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/clients/kube/resource_client.go#L154)，最终会调用到 [kube/resource_client_factory.go](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/clients/kube/resource_client_factory.go#L159) 中。这个函数里的逻辑就很清晰了，里面针对需要监听的 `namespace` 初始化了对应资源的 sharedInformer 并加入缓存中。

第二步，会调用各个 client 的 `List` 接口，对 `Register()` 中注册的 informer 进行 `Start`，等待数据 sync 到本地 cache 之后顺便 `List` 一份返回。[这里](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L416)是一个 artifact 资源的 `List` 调用的地方，往里看也会调用到 solo-kit 的 [kube/resource_client.go](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/clients/kube/resource_client.go#L288) 或 [kubesecret/resource_client.go](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/clients/kubesecret/resource_client.go#L244) 的 `List` 代码。

第三部，会调用各个 client 的 `Watch` 接口，监听该资源的情况，一旦有变化会反馈到 `Watch` 接口返回的 channel 中。例如[这里](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L423)是一个 artifact 资源的 `Watch` 调用的地方，最终也会调用到 solo-kit，此处不在赘述。

### 2. Watch 到数据，对数据进行部分处理

上面的 `Watch` 接口会返回一个 `artifactNamespacesChan`，在下面会[监听这个 channel](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L706)，一旦有数据过来，就会[汇总发布到 `artifactChan` 中](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L713)，然后另一个 goroutine 也[在监听 `artifactChan`](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L925)，一旦有数据产生，就会[放入 `currentSnapshot` 中](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L946)。

gloo 通过这种方式，把所有需要的资源逗整合到了 `Snapshot` 结构体中。

### 3. 把 `currentSnapshot` 送入 `Snapshot` channel

[每一秒一次](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L882)，将上面 `List` 以及 `Watch` 得到的结果[都放入 `currentSnapshot` 中](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L846)，最后[把 `currentSnapshot` 深拷贝一份](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#LL897C4-L897C16)，[放入 `Snapshot` channel 中](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/api/v1/gloosnapshot/api_snapshot_emitter.sk.go#L899)。

通过上面的三步，就实现了最核心的 K8S 资源到 `Snapshot` 的转换，最后会将这个 `Snapshot` 资源传入 `Sync()` 函数中，实现最核心的 K8S 配置到 envoy 的 xDS 转换流程。

## xDS 流程

gloo 的 Translate 最终做的工作是把 `ApiSnapshot` 转化为下面的 [`EnvoySnapshot`](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/xds/envoy_snapshot.go#L40)：

```go
// Snapshot is an internally consistent snapshot of xDS resources.
// Consistently is important for the convergence as different resource types
// from the snapshot may be delivered to the proxy in arbitrary order.
type EnvoySnapshot struct {
	// Endpoints are items in the EDS V3 response payload.
	Endpoints cache.Resources

	// Clusters are items in the CDS response payload.
	Clusters cache.Resources

	// Routes are items in the RDS response payload.
	Routes cache.Resources

	// Listeners are items in the LDS response payload.
	Listeners cache.Resources
}
```

Translate 转化完成之后，会[调用 `SetSnapshot` 方法](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/syncer/envoy_translator_syncer.go#L156)把 `EnvoySnapshot` 放到本地缓存中。一旦这个缓存被 Set，就会触发 xDS 服务和 envoy 的数据同步。

gloo 在初始化的时候会[初始化一个 `xdsServer`](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/xds/envoy.go#L27)，这个 `xdsServer` 会[实现下面的 interface](https://github.com/solo-io/solo-kit/blob/v0.31.0/pkg/api/v1/control-plane/server/generic_server.go#L52)：

```go
// Server is a collection of handlers for streaming discovery requests.
type Server interface {
	// StreamEnvoyV3 is the streaming method for Evnoy V3 XDS
	StreamEnvoyV3(
		stream StreamEnvoyV3,
		defaultTypeURL string,
	) error
	// StreamSolo is the streaming method for Solo discovery
	StreamSolo(
		stream StreamSolo,
		defaultTypeURL string,
	) error
	// Fetch is the universal fetch method.
	FetchEnvoyV3(
		context.Context,
		*envoy_service_discovery_v3.DiscoveryRequest,
	) (*envoy_service_discovery_v3.DiscoveryResponse, error)
	FetchSolo(
		context.Context,
		*sk_discovery.DiscoveryRequest,
	) (*sk_discovery.DiscoveryResponse, error)
}
```

然后会需要[利用这个 `xdsServer` 生成一个 `EnvoyServerV3`](https://github.com/solo-io/gloo/blob/v1.14.0/projects/gloo/pkg/xds/envoy.go#L30)，这个才是真正和 Envoy 做交互的，这个 Server 需要实现 Envoy xDS 相关接口

```go
// Server is a collection of handlers for streaming discovery requests.
type EnvoyServerV3 interface {
	envoy_service_endpoint_v3.EndpointDiscoveryServiceServer
	envoy_service_cluster_v3.ClusterDiscoveryServiceServer
	envoy_service_route_v3.RouteDiscoveryServiceServer
	envoy_service_listener_v3.ListenerDiscoveryServiceServer
	envoy_service_discovery_v3.AggregatedDiscoveryServiceServer
}
```

如果仔细查看这些接口的实现的话，会发现基本都是使用上面的 StramEnvoyV3 来实现的，这里就不列举了。

## 总结

以上就是 gloo 的一些源码的主要流程了，总体来说架构和思路还算清晰，但是代码可能有些复杂，我们在阅读 gloo 代码的时候需要注意的是，`*.sk.go` 的代码都是生成的代码，所以不用过于在乎这部分代码的写法，大致知道逻辑即可。