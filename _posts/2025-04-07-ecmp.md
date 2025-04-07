---
layout: post
title: ECMP
categories: [ network ]
description: 网络多路径算法
keywords: network
---

# ECMP

ECMP（Equal-Cost Multi-Path），中文通常叫做“等价多路径路由”或“等价多路径转发”。

它允许路由器/交换机对多个到达同一个目的地、开销相同（cost一样）的路径进行负载分担，以实现高可用和负载均衡。

假设你有一台路由器，它有两条路径到同一个目的地，代价（Cost）都为10。

```
   路由器A
   /     \
10/       \10
 /         \
B——→——→——→——→目的地C
```

在传统静态路由中，可能只能选一条。但有了 ECMP，A 可以在两条路径之间做流量分担，比如：

- 每个会话选一条路径（会话级 hash）
- 每个数据包选一条路径（更细粒度）
- 每 N 个流量分配一条路径（轮询）

## hash算法

当路径数量发生变化（增加或减少）时，哈希结果会“漂移”，导致原本的流量被打乱

### 方法一：Consistent Hash

- 哈希环（Hash Ring）中每条路径有多个虚拟节点
- 新路径加入时，只会影响少部分流量迁移
- 绝大多数流量的 hash 不变

### 方法二：Flow pinning / 会话固定

- 即使路径数量变化，已建立的会话仍然保持原路径
- 通过维护缓存或状态表（Flow table）来完成
- 特别适用于 NAT 场景、TCP 流等有状态的连接

### 方法三：逐步引入新路径（流量打温）

- 在生产环境中，新路径不是一次性加入的，而是：
- 先不参与转发
- 然后逐步增加其在 hash 中的权重
- 或者先用于新连接，避免影响旧连接

### 方法四：平滑哈希算法（HRW / Maglev）

- HRW（Highest Random Weight）

对于每个请求（如 IP 五元组）和每个可用路径（或服务器）组合，计算一个权重值，选择权重最大的那一条路径，当后端节点很多时，CPU占用会变得很高

1. 方法一：限制候选集
   不必对所有节点都 hash，一些实现只对同区域/同 rack内的节点做 HRW（比如服务网格的 locality aware）
   或者 hash 的时候做一次前置过滤（比如按权重分类）

2. 方法二：缓存结果 / lazy 计算
   对于热 key（比如固定连接），可以缓存 key → server 映射，避免每次都计算
   用 LRU cache 搭配 HRW 是非常常见的优化方式
3. 方法三：分层选择（Shard + HRW）
   假设你有 10,000 个 server，可以：
   先把它们按分组（比如按 rack、zone、hash shard）
   每组里用 HRW 算一遍，选出该组代表
   最后从代表里再选一个
   这样每次 hash 从 O(n) 降到 O(√n) 或更低
4. 方法四：干脆换成 Maglev 或 Power of Two Choices
   Maglev 预计算好查找表，查询是 O(1)，缺点是构表复杂，内存占用高
   Power of Two Choices（P2C）也是一种折中方案：只随机 hash 两个候选，看哪个负载低，用那个

- [Maglev Hash（Google LB）](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44824.pdf)

Maglev Hash 主要用于负载均衡系统中的请求路由，它的目的是高效且稳定地将请求路由到不同的后端服务器。Maglev Hash
旨在避免负载均衡过程中由于节点增加或删除引起的流量波动（比如 hash 漂移）。同时，它还保证了查询效率，可以实现 O(1)
的查找速度。


