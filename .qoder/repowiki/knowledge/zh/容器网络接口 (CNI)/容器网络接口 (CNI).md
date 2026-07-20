---
kind: external_dependency
name: 容器网络接口 (CNI)
slug: cni-container-network-interface
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

### 容器网络接口 (CNI)
- **角色**：Kubernetes 中 Pod 网络能力的标准化插件接口
- **集成点**：kubelet 在 Pod 创建时调用 CNI 插件配置网络命名空间
- **使用模式**：CNI 插件通过标准 JSON 配置文件接收网络配置，返回分配的 IP 地址
- **框架行为**：支持多种网络方案（Flannel VXLAN、Calico BGP、Cilium eBPF），通过插件机制动态加载
- **跨节点通信**：不同节点的 Pod 通过 CNI 建立的 Overlay 网络或路由直连进行通信
- **典型插件**：Flannel、Calico、Cilium、Weave Net 等第三方网络插件
- **注意**：CNI 是接口规范，具体实现由第三方提供，K8s 本身不包含网络实现