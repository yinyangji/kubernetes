---
kind: external_dependency
name: etcd 分布式键值存储
slug: etcd
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
---

### etcd 分布式键值存储
- **角色**：Kubernetes 控制面的唯一数据源，所有 API Server 的读写操作都持久化到 etcd
- **集成点**：API Server 通过 go.etcd.io/etcd/client/v3 客户端库与 etcd 集群通信
- **使用模式**：作为声明式状态机的最终一致性存储，所有组件通过 Watch 机制监听 etcd 变更
- **关键特性**：强一致性、支持 Watch 长连接推送、Raft 协议保证分布式一致性
- **版本要求**：当前项目依赖 v3.7.0 版本（go.mod 中声明）
- **部署形态**：通常以独立集群形式部署，与 API Server 高可用部署在同一网络域内