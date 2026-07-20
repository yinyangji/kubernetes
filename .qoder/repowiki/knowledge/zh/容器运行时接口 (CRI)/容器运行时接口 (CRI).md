---
kind: external_dependency
name: 容器运行时接口 (CRI)
slug: cri-container-runtime-interface
category: external_dependency
category_hints:
    - sdk_real_api
    - framework_behavior
scope:
    - '**'
---

### 容器运行时接口 (CRI)
- **角色**：Kubelet 与容器运行时之间的标准化 gRPC 接口抽象层
- **集成点**：kubelet 通过 CRI API 与 containerd、cri-o 等容器运行时交互
- **核心服务**：RuntimeService（管理容器生命周期）、ImageService（管理镜像）、StreamingInterface（流式传输）
- **使用模式**：gRPC 协议通信，定义 Create/Start/Stop/Remove 等标准方法
- **框架行为**：Kubelet 不直接操作容器，必须通过 CRI 间接调用，实现运行时解耦
- **生态支持**：支持 containerd、cri-o、docker shim（已弃用）等多种运行时实现
- **注意**：CRI 是 K8s 内部接口规范，不是外部依赖，但定义了与外部运行时的契约