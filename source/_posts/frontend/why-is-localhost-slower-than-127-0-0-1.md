---
title: Localhost 为什么比 127.0.0.1 慢？
date: 2026-05-02 17:50:10
tags:
    - Node.js
    - PWA
    - DNS
categories:
  - [前端]
---

先说结论：`localhost` 和 `127.0.0.1` 本身没有谁天然更快。正常情况下，`localhost` 多出来的主机名解析，只是在本机 resolver 或 hosts 文件里查一下 `localhost` 对应的 IP 地址，通常只是微秒级差异，几乎可以忽略。

真正容易拉开差距的是另一件事：`localhost` 不一定只指向 `127.0.0.1`，它还可能优先解析到 IPv6 的 `::1`。如果你的服务只监听了 IPv4，客户端先尝试连 `::1`，再回退到 `127.0.0.1`，这一步就可能带来 200ms 到 300ms 级别的连接延迟。

## 现象

先看一个本地开发服务的例子。通过 `127.0.0.1` 访问：

![通过 127.0.0.1 访问本地服务](/img/why-is-localhost-slower-than-127-0-0-1/127server.png)

再通过 `localhost` 访问同一个服务：

![通过 localhost 访问本地服务](/img/why-is-localhost-slower-than-127-0-0-1/localhostserver.png)

同一个页面，同样 12 次请求，传输数据量也差不多，一个耗时 214ms，一个耗时 691ms，几乎差了三倍。

这个页面的请求不多，所以瀑布图里很容易看到有几个请求明显更慢。点开其中一个请求，问题集中在连接阶段：

![localhost 请求的连接耗时](/img/why-is-localhost-slower-than-127-0-0-1/TTFB1.png)

再放一张相同请求在另一个访问方式下的对比：

![127.0.0.1 请求的连接耗时](/img/why-is-localhost-slower-than-127-0-0-1/TTFB2.png)

这里需要注意一个细节：这不是传统意义上的“后端处理慢”，也不该简单归因成“DNS 查询慢”。DevTools 里看到的慢点更接近 `Initial connection`，也就是建立 TCP 连接时的地址选择和回退成本。

## 根因：IPv6 优先和 Happy Eyeballs

在很多系统里，`localhost` 会同时对应 IPv6 和 IPv4 地址。比如可以先看本机解析结果：

```shell
$ getent hosts localhost
::1             localhost
```

这只能说明当前环境里 `localhost` 的解析结果优先给了 `::1`。不同系统、不同 hosts 配置、不同 resolver 策略下，结果顺序可能不同。更完整的排查可以用：

```shell
getent ahosts localhost
curl -v http://localhost:4173
curl -4 -v http://localhost:4173
curl -6 -v http://localhost:4173
```

如果服务没有监听 `::1`，`curl -v` 里常见的现象会是这样：

```text
* Host localhost:4173 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:4173...
* connect to ::1 port 4173 from ::1 port 51430 failed: Connection refused
*   Trying 127.0.0.1:4173...
* Connected to localhost (127.0.0.1) port 4173
```

这就是 IPv6 到 IPv4 的回退。

curl 的 Happy Eyeballs 默认会先尝试 IPv6；如果 IPv6 连接还没有成功，默认 200ms 后会并行发起 IPv4 连接。Chrome/Chromium 也有类似策略，Chromium 源码里 `kIPv6FallbackTime` 当前是 300ms，和上面截图中慢出来的连接耗时基本对得上。

所以这里的“慢”不是 `localhost` 这个字符串慢，而是：

1. `localhost` 优先解析到 `::1`；
2. 服务实际只监听 `127.0.0.1`；
3. 客户端先尝试 IPv6，再回退到 IPv4；
4. 每新建一次 TCP 连接，就可能重复一次回退成本。

## 为什么会重复出现

如果只是第一次连接慢一次，影响还比较有限。但本地开发时经常能稳定复现，是因为请求没有复用连接。

在 Vite 的代理场景里，`server.proxy` / `preview.proxy` 的配置会传给底层 `http-proxy-3`。`http-proxy-3` 向上游服务发请求时会使用 Node 的 HTTP Agent。Node 文档里说明，当 `keepAlive=false` 且 `maxSockets=Infinity` 时，请求会使用 `Connection: close`。也就是说，请求结束后连接会关闭，下一次请求又要重新建连。

vite 默认没有开启 keep-alive。只要每个接口请求都要重新建连，就可能反复经历：

```text
localhost -> ::1 -> 连接失败/等待回退 -> 127.0.0.1 -> 成功
```

这也是为什么瀑布图里不止一个请求慢。

## 怎么处理

处理方法就是给 Vite proxy 新增一个 keep-alive Agent，让代理到上游服务的连接可以复用，避免每个接口都重新经历连接阶段：

```ts
import http from 'node:http'
import { defineConfig } from 'vite'

const keepAliveAgent = new http.Agent({
  keepAlive: true,
})

export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8080',
        changeOrigin: true,
        agent: keepAliveAgent,
      },
    },
  },
  preview: {
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8080',
        changeOrigin: true,
        agent: keepAliveAgent,
      },
    },
  },
})
```

如果上游是 HTTPS，则要换成 `https.Agent`。

## 排查建议

可以用下面这个命令把 DNS、连接、首字节和总耗时拆开看：

```shell
curl -o /dev/null -s -w 'namelookup=%{time_namelookup} connect=%{time_connect} starttransfer=%{time_starttransfer} total=%{time_total}\n' http://localhost:4173/
```

再分别对比：

```shell
curl -o /dev/null -s -w 'namelookup=%{time_namelookup} connect=%{time_connect} starttransfer=%{time_starttransfer} total=%{time_total}\n' http://127.0.0.1:4173/
curl -4 -o /dev/null -s -w 'connect=%{time_connect} total=%{time_total}\n' http://localhost:4173/
curl -6 -o /dev/null -s -w 'connect=%{time_connect} total=%{time_total}\n' http://localhost:4173/
```

如果 `localhost` 明显慢，而 `curl -4` 恢复正常，基本就可以确认是 IPv6 优先和 IPv4 回退带来的连接成本。

参考：

- everything curl: Happy Eyeballs: [https://everything.curl.dev/usingcurl/connections/happy.html](https://everything.curl.dev/usingcurl/connections/happy.html)
- Chromium `kIPv6FallbackTime`: [https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/socket/transport_connect_job.h#124](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/socket/transport_connect_job.h#124)
- Node.js HTTP Agent: [https://nodejs.org/api/http.html#new-agentoptions](https://nodejs.org/api/http.html#new-agentoptions)
- Vite server.proxy: [https://vite.dev/config/server-options.html#server-proxy](https://vite.dev/config/server-options.html#server-proxy)
