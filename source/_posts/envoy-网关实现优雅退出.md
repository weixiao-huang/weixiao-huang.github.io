---
title: envoy 网关实现优雅退出
date: 2023-03-07 16:30:33
tags: gateway
categories: Gateway
---

在我们的项目中，我们采用了 [gloo](https://github.com/solo-io/gloo) 作为我们最前端的网关，gloo 是基于 envoy 的云原生网关， gloo 实现了一套控制器，自定义了 `VirtualService`, `RouteTable`, `Upstream` 等 CRD，利用 envoy 的 [xDS](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) ，将 K8S 的 CRD 转化为了 envoy 自身的配置实现反向代理。

gloo 将 envoy 的反向代理通过 K8S 部署成为一个 `gateway-proxy` 的 Deployment，所有网关流量可以打到这个 Deployment 对应的 Pod 或者 Service 上即能实现反向代理。

但是需要注意的是，envoy 并没有实现完善的 Gracefully Stop 功能，在我们的项目中，利用 envoy 的 2222 端口代理了我们的一个 ssh 反向代理服务，用户会通过 envoy 的 2222 端口脸上 upstream 的 ssh 服务。但是如果不做优雅退出的话，一旦我们做 gateway-proxy 这个 Deployment 的滚动更新，就会导致所有用户的 ssh 连接断连，造成非常不好的体验，因此实现一个 Gracefully Stop 是迫在眉睫的事情。

初步调研看来， [gloo 有 PR](https://github.com/solo-io/gloo/pull/3458) 实现了一个简单的优雅退出的逻辑，也写了 [相关文档](https://docs.solo.io/gloo-edge/latest/operations/advanced/zero-downtime-gateway-rollout/#configuring-the-kubernetes-probes)。但仔细看了 PR 里的细节才发现，gloo 的这个功能比较鸡肋，可能会造成所有的 conn 都退出之后，还在等 sleep 的诡异景象，使得 gateway-proxy pod 迟迟处于 Terminating 状态，不是一个合理的解决方案。

经过调研，我们发现 istio 自己实现了一个优雅退出，[通过暴露一个 `EXIT_ON_ZERO_ACTIVE_CONNECTIONS ` 配置](https://www.zhaohuabing.com/istio-guide/docs/best-practice/graceful-termination/)。这个给了我们新思路，仔细查看它的代码，详细逻辑是 [`activeProxyConnections`](https://github.com/istio/istio/blob/1.17.2/pkg/envoy/agent.go#L170) 这个函数

```go
func (a *Agent) activeProxyConnections() (int, error) {
	activeConnectionsURL := fmt.Sprintf("http://%s:%d/stats?usedonly&filter=downstream_cx_active$", a.localhost, a.adminPort)
	stats, err := http.DoHTTPGet(activeConnectionsURL)
	if err != nil {
		return -1, fmt.Errorf("unable to get listener stats from Envoy : %v", err)
	}
	if stats.Len() == 0 {
		return -1, nil
	}
	// 后面的逻辑省略
}
```

可以看到可以采用 envoy 的 [/stats](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--stats) 接口了解到 downstream 的连接数，通过对监控指标的 filter 即可查询到连接数，如果连接数一直不清零，就一直等待即可。

最终我们通过给 gateway-proxy Pod 书写如下的 `preStop` 终于实现了优雅退出

```yaml
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - for i in $(seq 1 8640); do if [[ $(wget -O- "127.0.0.1:19000/stats?usedonly&filter=^listener\.\[__\]_2222\.downstream_cx_active$"
                | awk 'BEGIN{sum=0}{sum+=$2;}END{print sum;}') -ne 0  ]]; then sleep
                10; else break; fi done
```

稍微解释一下上面的 command
- for 循环的作用：我们采用了一个 for 循环，循环 8640 次，并且利用 sleep 命令实现 10s 检测一次，因此总共 86400s，即优雅退出 timeout 大概在 1d 以上。
- wget 的作用：在容器中并没有直接照抄 istio 的 filter，而是自己写了个 `filter=^listener\.\[__\]_2222\.downstream_cx_active$"`，这是因为 istio 检查了所有 downstream，但是我们没有这个必要，我们主要只需要检查 2222 端口的 downstream 即可，因此我们写了一个正则表达式，这样可以提高 wget 返回的速度。
- 这里的 usedonly 是必须的，如果不填的话在集群规模大的情况下，此接口会需要返回特别多的信息，特别耗时间，利用 usedonly 能够极大的减小请求时间。
- awk 的作用：wget 返回的结果类似这样：`listener.[__]_2222.downstream_cx_active: 52`，不排除会出现多行或者一行都没有，因此我们用 awk 获取了空格后面的数字，例如这里的 `52`，然后如果有多行，会将多行的数字相加并输出，如果一行都没有就输出 0，我们利用 awk 的结果判断是否是 0 即可知道连接数是否是 0，可谓精妙。

除了上面的配置之外，Pod 的 spec 层面需要配置 `terminationGracePeriodSeconds: 86400` 固定 1 天时间优雅退出，和上面的 86400s 需要匹配即可完美实现 envoy 监听的 2222 端口的优雅退出。