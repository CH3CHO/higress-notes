# Wasm 插件本地测试时如何连接本地 Redis

## TL;DR

envoy.yaml 里配置 Redis cluster 时，socketAddr 要尽量用 IP，不要用主机名。

## 背景

在开发 Wasm 插件过程中，我们镜像会使用 Docker Compose + Envoy + Volume Mount 的方式测试本地构建出来的插件。如果插件需要连接 Redis，那么我们就需要在 envoy.yaml 中配置一个 Redis 的 cluster。如果配置中的 Redis 节点地址使用机器名，那么在启动插件的时候可能会出现初始化 Redis 客户端报“bad argument”的错误。

## 原因

这种错误一般只发生在插件在 `parseConfig` 阶段调用 `RedisClusterClient.Init()` 函数的时候。、

在 Envoy 初始化的过程中，集群信息的初始化与 Wasm 插件的初始化可以认为是并行进行的。如果使用主机名进行配置，要获取实例的实际 IP 就需要经过 DNS 解析。而 DNS 解析一般是需要一些时间的（笔者的测试机和网络条件下需要约 200ms），Redis 客户端的初始化又需要与 Redis 集群建立连接和通信。这一延迟就可能会导致 Wasm 插件进行初始化时 Redis 的集群信息还没有就绪，进而引发上述报错。

而在 Higress 的实际运行过程中，集群信息是通过 xDS 进行下发的，这个延迟的问题不会非常显著。

## 解决方案示例

Docker Compose 本身是支持给服务配置静态 IP 的。我们可以使用这一配置来让 Redis 集群获得静态 IP 以便我们更新 Envoy 配置。

```yaml
version: '3.7'
services:
  envoy:
    image: higress-registry.cn-hangzhou.cr.aliyuncs.com/higress/gateway:1.4.2
    entrypoint: /usr/local/bin/envoy
    command: -c /etc/envoy/envoy.yaml --log-level info --component-log-level wasm:debug
    depends_on:
      - http-echo
    networks:
      - wasmtest
    ports:
      - "10000:10000"
      - "9901:9901"
    volumes:
      - /home/me/higress/plugins/wasm-go/extensions/ai-proxy/test/envoy.yaml:/etc/envoy/envoy.yaml
      - /home/me/higress/plugins/wasm-go/extensions/ai-proxy/out/plugin.wasm:/etc/envoy/plugin.wasm

  redis:
    image: redis:latest
    networks:
      wasmtest:
        ipv4_address: 172.20.0.100
    ports:
      - "6379:6379"

  http-echo:
    image: mendhak/http-https-echo:latest
    networks:
      - wasmtest
    ports:
      - "12345:8080"

networks:
  wasmtest:
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

```yaml
    - name: outbound|6379||redis.dns
      connect_timeout: 30s
      type: LOGICAL_DNS
      dns_lookup_family: V4_ONLY
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: outbound|6379||redis.dns
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 172.20.0.100
                      port_value: 6379
```