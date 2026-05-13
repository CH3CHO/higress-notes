# 常见问题

## 前置阅读

基本上所有的问题都可以通过查看 Higress 各个组件的运行时日志、网关的访问日志以及核对它们的运行时配置是否符合预期来解决。理解日志所输出信息的 **“字面含义”** 是排障过程中的重中之重。

- [如何查看日志？](https://higress.cn/docs/latest/ops/how-tos/view-logs/)
- [如何查看运行时配置？](https://higress.cn/docs/latest/ops/how-tos/view-configs/)

## 一般问题

### 为什么我的网关本来运行正常，但重启之后就起不来了？

原因是当前的最新配置有问题，网关加载失败。正常运行时，网关会拒绝控制面推送过来的异常配置，保留之前获取到的可用配置并继续运行。但在启动过程中，网关必须要拿到一份正常配置才能够完成启动。这时也没有可用配置的缓存了。所以异常配置会直接阻塞网关的正常启动。

解决方法：查看 gateway 的运行日志，找到加载失败的错误配置并修正。

### 为什么我的配置没有生效？

网关的很多配置都是打包推送的。在某一类配置加载出现问题时，网关就会拒绝该类配置的推送，保留之前获取到的可用配置并继续运行。所以在同类配置中存在错误配置的情况下，哪怕后续的配置更新本身是没有问题的，网关也不会接受这些新的配置。

解决方法：
1. 查看 gateway 的运行日志，找到加载失败的错误配置并修正。

备注：如果日志太多不便于查询，在不影响业务的情况下，可以通过重启网关来清理日志。

### Higress Console 路由配置中的 Header、Query 等参数是做什么用的？

路由配置中的域名、Path、Header、Query 都是这条路由的匹配条件，也就是说只有当请求满足这些匹配条件时，网关才会按照这条路由的各项配置对请求进行修改和转发，并执行该路由所关联的插件。

### 请求网关返回 404 怎么办？

请查看访问日志，通过 404 状态码、请求路径等方式找到报错请求所对应的日志条目。查看其中的 `response_code_details` 字段。其不同的取值就代表不同的 404 原因。

- `route_not_found`：没有找到对应的路由。需要检查路由列表，判断是否配置了能够匹配到该请求的路由规则。
  - 如果确认配置无误，那么可以拉取 controller 和 gateway 的运行时配置，查看配置中是否包含期望中的路由。
  - 如果 controller 中包含该路由但 gateway 中没有，那基本可以确定是有配置加载失败了。可以查看 gateway 的日志，找到加载失败的原因并对其进行修正。
- `via_upstream`：404 是后端服务返回的。建议查看后端服务的相关日志并判断原因。

## AI 相关问题

### AI 路由相关问题

注意：AI 路由功能极大的依赖 Higress 的各类 Wasm 插件。如果你的服务器无法访问外网，经查看下方“插件相关问题”一节中的对应问题来解决在内网下载 Wasm 插件的问题。

#### 我的大模型部署方式不在大模型提供方类型列表中，怎么办？

如果你的大模型兼容 OpenAI API，那么可以直接使用“OpenAI/OpenAI兼容服务”这里类型，在下方“OpenAI服务类型”下拉框中选择“自定义服务”，并填入服务的 URL。

### MCP Server 相关问题

注意：MCP Server 功能极大的依赖 Higress 的各类 Wasm 插件。如果你的服务器无法访问外网，经查看下方“插件相关问题”一节中的对应问题来解决在内网下载 Wasm 插件的问题。

#### mcpServer 配置中的 Redis 是做什么用的？

这里的 Redis 仅在存量 REST API 服务转 MCP Server 的时候进行 SSE 会话保持使用。如果被代理服务的已经是 MCP Server，或者仅需通过 Streamable HTTP 方式来请求存量服务转化而成的 MCP Server，那么是不需要配置 Redis 的。

#### 存量 API 转 MCP Server 可以使用 Streamable HTTP 方式请求，但使用 SSE 方式请求就返回 405，怎么办？

请通过 Higress Console 的系统设置页面检查 `higress-config` ConfigMap 中 `mcpServer` 的各项配置是否已正确配置：

- `enable` 需要设置为 `true`
- `redis` 需要正确配置，并确认网关内可以访问。可以进入网关容器内使用 `telnet` 命令来确认端口可以正常连接。不要用 `ping`。
- `match_list` 中配置的匹配规则可以覆盖到客户端请求。

#### 每加一个 MCP Server 就要改一次 `match_list`，太麻烦了

可以直接使用从 Higress 2.1.5 起支持的“MCP管理”功能直接进行配置。这样就完全不需要配置 `match_list` 了（但需要的话 `redis` 还是得配）。

如果现有的“MCP管理”功能无法满足需求，那么可以考虑通过给相应的路由添加注解的方式进行配置。

| 注解名称                                    | 注解描述                                                                                                                                        | 示例         |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| `higress.io/mcp-server`                     | 是否为该路由启用 MCP Server 功能                                                                                                                | true         |
| `higress.io/mcp-server-match-rule-domains`  | 等同于 `match_list` 中的 `match_rule_domain`                                                                                                    | *            |
| `higress.io/mcp-server-match-rule-type`     | 等同于 `match_list` 中的 `match_rule_type`                                                                                                      | prefix       |
| `higress.io/mcp-server-match-rule-value`    | 等同于 `match_list` 中的 `match_rule_path`                                                                                                      | /mcp/example |
| `higress.io/mcp-server-upstream-type`       | 等同于 `match_list` 中的 `upstream_type`。仅当后端为使用 SSE 传输方式的 MCP Server 时需要配置，值为 `sse`                                       | sse          |
| `higress.io/mcp-server-enable-path-rewrite` | 等同于 `match_list` 中的 `enable_path_rewrite`。仅当后端为使用 SSE 传输方式的 MCP Server 且要求重写请求路径时需要配置，值为 `true`              | true         |
| `higress.io/mcp-server-path-rewrite-prefix` | 等同于 `match_list` 中的 `path_rewrite_prefix`。仅当后端为使用 SSE 传输方式的 MCP Server 且要求重写请求路径时需要配置，值为重写后的请求路径前缀 | /mcp         |

编辑路由并在“附加注解”列表中添加上述注解即可。

## 插件相关问题

### 配置的插件功能不工作或非预期

Higress 上所有的插件配置是打包下发的，也就是意味着如果你同时配置了 A、B 两个插件，只要有一个插件的配置加载异常，两个插件的配置都不会正常生效。

解决方法：查看 gateway 的运行日志，找到配置加载失败的插件及相关的失败原因，并修正错误的配置。

备注：如果日志太多不便于查询，在不影响业务的情况下，可以通过重启网关来清理日志。

### 网关服务器无法访问外网，怎么使用插件？

Higress 默认从公网镜像仓库来下载插件镜像。如果你的服务器无法访问外网，那么所有的插件都将不能正常工作。

解决办法：
1. 在内网部署社区开源的 [plugin-server](https://github.com/higress-group/plugin-server/)，并根据其文档修改内置插件的镜像地址。
2. 如果你已经配置了一些插件（含 AI 路由、MCP Server），这些插件配置中已经记录了默认的外网镜像地址。你需要将已有 `WasmPlugins` 资源中的 `url` 字段改为对应插件在 plugin-server 上的 URL 才行。如果你使用的是容器方式来部署的 Higress，请查看此文档了解修改各种资源配置的方法：[链接](./standalone-crs.md)

### 插件请求 API 或 Redis 要怎么配，为什么会遇到 `bad argument` 错误？

这里 `bad argument` 错误在绝大部分场景下都是由于配置了网关并不知晓的服务信息。

Wasm 插件只能请求网关拿到的 Cluster 信息，也就是 Higress Console 的服务列表页面所列出的服务，而不能直接指定目标服务的地址和端口来请求。

针对要通过 IP 或域名访问的服务，我们需要为其创建对应的服务来源，然后到服务列表中查看它所对应的服务名称和服务端口。这两个信息也是我们需要写入 Wasm 插件的配置里的。其中固定地址类型的服务来源所生成服务的端口固定为 80，请务必注意。

针对 K8s 内的 Service，Higress 默认只会下发关联了路由的 K8s 服务。这里有两个解决方法：

1. 可以为要调用的服务创建一个路由，通过配置一些特殊的匹配规则使其并不会有任何请求匹配到它即可；
2. 为 K8s 配置一个 DNS 类型的服务来源，在插件中配置该服务来源所对应的服务信息。

### 为什么 WasmPlugin 配置里的镜像地址都已经改了，网关还在去官方仓库下载 mcp-server 插件？

这种问题一般发生于网关对接了 Nacos 3.x 的 MCP Server 功能后。请查看文档：[链接](https://higress.cn/docs/latest/ops/how-tos/builtin-plugin-url/#%E5%AF%B9%E6%8E%A5-nacos-3x-%E6%89%80%E7%94%9F%E6%88%90%E7%9A%84-mcp-server-%E6%8F%92%E4%BB%B6%E5%9C%B0%E5%9D%80%E9%85%8D%E7%BD%AE)

## 常见错误

### 使用 `127.0.0.1` 来配置网关要访问的服务

在网关的各项配置中，除非 100% 确定，请不要使用 `127.0.0.1` 这个 loopback IP。因为当网关访问这个 IP 时，实际被访问的是网关自己，而不是网关容器所在的宿主机。请使用其他网关可以正常访问的 IP 进行配置。

### 使用 `ping` 来检查后端服务的连通性

`ping` 用的是 ICMP 协议，而网关转发请求用的是 TCP 协议。能 `ping` 通不代表目标服务的地址和端口可以连接。请使用 `telnet` 或 `nc` 命令进行检查。