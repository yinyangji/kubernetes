# Kubernetes 底层原理与核心机制深度解析

> 基于 Kubernetes 源码分析，聚焦底层实现和调用逻辑。

---

## 目录

1. [总体架构概览](#1-总体架构概览)
2. [声明式 API 与控制循环：一切的基础](#2-声明式-api-与控制循环一切的基础)
3. [Master 如何把任务分发给 Worker 节点](#3-master-如何把任务分发给-worker-节点)
4. [Worker 节点如何接收和执行编排指令](#4-worker-节点如何接收和执行编排指令)
5. [节点之间如何通信](#5-节点之间如何通信)
6. [节点状态上报 vs Master 主动查询](#6-节点状态上报-vs-master-主动查询)
7. [守护进程：DaemonSet 机制](#7-守护进程daemonset-机制)
8. [集群动态伸缩机制](#8-集群动态伸缩机制)
9. [节点加入集群：硬件能力上报与 TEE/CVM 支持](#9-节点加入集群硬件能力上报与-teecvm-支持)
10. [非 IDC 场景组网与跨网络集群](#10-非-idc-场景组网与跨网络集群)

---

## 1. 总体架构概览

Kubernetes 的架构本质上是一个**分布式状态机**，核心思想是：

```
                    ┌──────────────────────────────────────────┐
                    │            Control Plane (Master)         │
                    │                                          │
                    │  ┌─────────┐  ┌──────────┐  ┌────────┐  │
                    │  │API Server│→│Scheduler │  │  etcd  │  │
                    │  └────┬────┘  └────┬─────┘  └────────┘  │
                    │       │            │                      │
                    │  ┌────┴────────────┴─────┐               │
                    │  │ Controller Manager    │               │
                    │  │ (Node/Deployment/     │               │
                    │  │  DaemonSet/HPA/...)   │               │
                    │  └───────────────────────┘               │
                    └─────────────┬────────────────────────────┘
                                  │ HTTPS (Watch/List)
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────┴────────┐ ┌───────┴────────┐ ┌────────┴─────────┐
    │   Worker Node 1  │ │ Worker Node 2  │ │   Worker Node N  │
    │  ┌─────────────┐ │ │                │ │                  │
    │  │   Kubelet   │ │ │   Kubelet      │ │    Kubelet       │
    │  │ (节点代理)  │ │ │                │ │                  │
    │  └──────┬──────┘ │ │                │ │                  │
    │  ┌──────┴──────┐ │ │   kube-proxy   │ │   kube-proxy     │
    │  │ kube-proxy  │ │ │                │ │                  │
    │  └─────────────┘ │ │  Container RT  │ │  Container RT    │
    │  Container RT    │ │  (containerd)  │ │  (containerd)    │
    │  (containerd)    │ │                │ │                  │
    └──────────────────┘ └────────────────┘ └──────────────────┘
```

**关键组件职责**：

| 组件 | 运行位置 | 核心职责 |
|------|---------|---------|
| **API Server** | Master | 所有操作的统一入口，RESTful 接口，数据持久化到 etcd |
| **Scheduler** | Master | 监听未调度 Pod，选择最优节点并绑定 |
| **Controller Manager** | Master | 运行数十个控制循环（Node/Deployment/DaemonSet/HPA 等） |
| **etcd** | Master | 分布式 KV 存储，集群唯一数据源 |
| **Kubelet** | Worker | 节点代理，管理 Pod 生命周期的守护进程 |
| **kube-proxy** | Worker | 维护 Service 的网络规则（iptables/IPVS） |

---

## 2. 声明式 API 与控制循环：一切的基础

理解 Kubernetes 底层机制的钥匙是理解其**声明式 + 控制循环**的编程范式：

```
用户声明期望状态（Desired State）
        ↓
写入 etcd（通过 API Server）
        ↓
控制循环持续比对 期望状态 vs 实际状态
        ↓
发现差异 → 执行动作 → 缩小差异 → 最终一致
```

### 2.1 Informer/Watch 机制——所有组件的通信基础

Kubernetes 中**几乎所有组件**都基于同一个通信模式与 API Server 交互：**List-Watch + 本地缓存**。

源码位于 `staging/src/k8s.io/client-go/tools/cache/controller.go`：

```go
// Reflector 负责 List-Watch API Server
func (c *controller) RunWithContext(ctx context.Context) {
    r := NewReflectorWithOptions(
        c.config.ListerWatcher,    // 负责 List 和 Watch API Server
        c.config.ObjectType,       // 资源类型（Pod/Node/Service...）
        c.config.Queue,            // DeltaFIFO 队列
        ...)
    // 启动 Reflector 的 List-Watch 循环
    wg.StartWithContext(ctx, r.RunWithContext)
    // 启动事件处理循环，从队列中取出事件并分发给 handler
    wait.UntilWithContext(ctx, c.processLoop, time.Second)
}
```

**数据流**：
```
API Server ──(Watch 推送)──→ Reflector ──→ DeltaFIFO ──→ 本地缓存(Store)
                                                      └──→ EventHandler (Add/Update/Delete)
```

**关键设计**：
1. **不是轮询，是 Watch（长连接推送）**：API Server 主动推送变更给客户端
2. **本地缓存**：所有数据在本地有一份完整镜像，读取操作不直接查 API Server
3. **Resync 机制**：定期（如 12h）全量同步一次，防止缓存漂移
4. **ResourceVersion**：通过资源版本号保证事件顺序和一致性

`SharedInformerFactory`（`staging/src/k8s.io/client-go/informers/factory.go`）统一管理所有 Informer，同一资源类型共享一个 Watch 连接：

```go
// 每种资源类型共享一个 Informer，避免重复 Watch
func (f *sharedInformerFactory) InformerFor(obj runtime.Object,
    newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
    // 已存在则直接返回，不存在则创建
    informerType := reflect.TypeOf(obj)
    informer, exists := f.informers[informerType]
    if exists { return informer }
    informer = newFunc(f.client, resyncPeriod)
    f.informers[informerType] = informer
    return informer
}
```

---

## 3. Master 如何把任务分发给 Worker 节点

### 3.1 核心流程：Scheduler 的工作流

Scheduler 运行在 Master 节点，它的核心职责是：**监听未被调度的 Pod → 找到最优节点 → 绑定**。

**关键事实：Master 不"推送"任务给 Worker，而是"绑定" Pod 到节点。**

流程如下（源码 `pkg/scheduler/schedule_one.go`）：

```
1. 用户创建 Pod（无 spec.nodeName）
2. Scheduler 通过 Informer Watch 到新 Pod
3. Pod 进入调度队列（SchedulingQueue）
4. ScheduleOne 从队列取出 Pod
5. schedulingAlgorithm：Filter（过滤）→ Score（打分）→ 选最优节点
6. Assume：在内存中假设绑定成功（性能优化）
7. Bind：异步将绑定写入 API Server（写入 etcd）
8. Kubelet 通过 Watch 发现"这个 Pod 被分配到了我的节点"
```

```go
// ScheduleOne 核心调度循环 (pkg/scheduler/schedule_one.go:66)
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
    entity, err := sched.NextEntity(logger)  // 从调度队列取出下一个 Pod
    // ...
    switch specificEntity := entity.(type) {
    case *framework.QueuedPodInfo:
        sched.scheduleOnePod(ctx, specificEntity)  // 调度单个 Pod
    }
}
```

### 3.2 调度算法：Filter + Score

调度算法的核心是两阶段模型：

```
所有节点 → [Filter 过滤] → 可行节点列表 → [Score 打分] → 最优节点
```

#### Filter 阶段（过滤不可用节点）

```go
// 并行过滤节点 (pkg/scheduler/schedule_one.go:778)
func (sched *Scheduler) findNodesThatPassFilters(...) {
    checkNode := func(i int) {
        nodeInfo := nodes[(sched.nextStartNodeIndex+i)%numAllNodes]
        // 并行执行所有 Filter 插件
        status := schedFramework.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
        if status.IsSuccess() {
            // 添加到可行节点列表
            feasibleNodes[length-1] = nodeInfo
        }
    }
    // 并行执行！
    schedFramework.Parallelizer().Until(ctx, numAllNodes, checkNode, metrics.Filter)
}
```

**Filter 插件包括**：
- `NodeResourcesFit`：节点资源是否满足 Pod 请求（CPU/内存/GPU）
- `TaintToleration`：节点 Taint 与 Pod Toleration 是否匹配
- `NodeAffinity`：节点亲和性（nodeSelector/nodeAffinity）
- `PodTopologySpread`：Pod 拓扑分布约束
- `VolumeRestrictions`：卷限制
- `NodePorts`：端口冲突检查
- `NodeUnschedulable`：节点是否标记为不可调度

#### Score 阶段（对可行节点打分排序）

```go
// 对可行节点打分 (pkg/scheduler/schedule_one.go:607)
priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
sortedPrioritizedNodes := framework.NewSortedScoredNodes(priorityList)
node := sortedPrioritizedNodes.Pop().Name  // 取得分最高的节点
```

**Score 插件包括**：
- `LeastRequestedPriority`：优先选择资源使用率低的节点（负载均衡）
- `InterPodAffinity`：Pod 间亲和性打分
- `TaintToleration`：容忍匹配度打分

### 3.3 调度框架插件体系

Scheduler 使用插件化框架（`pkg/scheduler/framework/`），定义了以下扩展点：

```
PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind
```

这是一个**管道**，Pod 依次通过每个阶段。任何一个阶段失败都会导致调度失败。

### 3.4 关键设计：异步绑定

```go
// 调度决策完成后，异步绑定 (schedule_one.go:140)
// bind the pod to its host asynchronously (we can do this b/c of the assumption step above)
go sched.runBindingCycle(ctx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
```

Scheduler **不等待绑定完成**就开始调度下一个 Pod。它通过"假设"（Assume）机制告诉调度缓存"这个 Pod 已经绑定到 Node X"，从而在下一个调度周期中正确计算节点资源。

---

## 4. Worker 节点如何接收和执行编排指令

### 4.1 Kubelet：Worker 节点上的守护进程

**Kubelet 就是那个专门负责 Pod 生命周期管理的守护进程。**

每个 Worker 节点上运行一个 Kubelet 进程，它的核心工作是：

1. **Watch API Server**：监听分配给本节点的 Pod
2. **管理 Pod 生命周期**：创建/启动/停止容器
3. **上报节点状态**：定期向 Master 汇报节点健康状态

### 4.2 Kubelet 如何获知任务——Watch 机制

源码 `pkg/kubelet/config/apiserver.go`：

```go
// Kubelet 通过 Watch API Server 获取分配给本节点的 Pod
func NewSourceApiserver(logger klog.Logger, c clientset.Interface,
    nodeName types.NodeName, ...) {
    // 关键：只 Watch 本节点的 Pod（通过 field selector 过滤）
    lw := cache.NewListWatchFromClient(
        c.CoreV1().RESTClient(), "pods", metav1.NamespaceAll,
        fields.OneTermEqualSelector("spec.nodeName", string(nodeName)))
    // 使用 Reflector 持续 Watch
    r := cache.NewReflector(lw, &v1.Pod{}, ...)
    go r.Run(wait.NeverStop)
}
```

**关键事实：是 Worker 主动 Watch（长连接订阅），不是 Master 推送。**

工作原理：
1. Kubelet 向 API Server 发起 Watch 请求：`/api/v1/pods?fieldSelector=spec.nodeName=<my-node>&watch=true`
2. API Server 维持 HTTP 长连接，当有 Pod 变更时推送事件
3. Kubelet 的 Reflector 将事件写入 `PodConfig` 的 updates channel

### 4.3 PodConfig：多源配置聚合器

`pkg/kubelet/config/config.go` 中的 `PodConfig` 是一个多源配置合并器：

```go
type PodConfig struct {
    pods    *podStorage    // 当前所有 Pod 状态
    mux     *mux           // 多路复用
    updates chan PodUpdate // 变更通知通道
}
```

支持三个配置源：
- **API Server**（主要来源）：通过 Watch 获取远程调度的 Pod
- **文件**（Static Pods）：从本地目录读取 YAML 文件
- **HTTP URL**：从远程 HTTP 端点拉取配置

所有源的变更最终合并到 `updates` channel。

### 4.4 syncLoop：Kubelet 的主循环

`pkg/kubelet/kubelet.go:2671`：

```go
func (kl *Kubelet) syncLoop(ctx context.Context,
    updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
    // 多个事件源
    syncTicker := time.NewTicker(time.Second)         // 1秒定期同步
    housekeepingTicker := time.NewTicker(housekeepingPeriod) // 内务整理
    plegCh := kl.pleg.Watch()                          // PLEG 容器事件

    for {
        kl.syncLoopIteration(ctx, updates, handler,
            syncTicker.C, housekeepingTicker.C, plegCh)
    }
}
```

`syncLoopIteration` 从多个 channel 读取事件并分发：

```go
func (kl *Kubelet) syncLoopIteration(...) bool {
    select {
    case u := <-configCh:
        switch u.Op {
        case ADD:    handler.HandlePodAdditions(ctx, u.Pods)  // 新 Pod
        case UPDATE: handler.HandlePodUpdates(ctx, u.Pods)    // Pod 更新
        case REMOVE: handler.HandlePodRemovals(ctx, u.Pods)   // Pod 删除
        }
    case e := <-plegCh:      // 容器状态变更事件
    case <-syncCh:           // 定期同步
    case <-housekeepingCh:   // 清理孤儿容器
    }
}
```

### 4.5 PodWorkers：每个 Pod 一个 Goroutine

Kubelet 为**每个 Pod 维护一个独立的 Goroutine**（`pkg/kubelet/pod_workers.go`）：

```
Pod 调度到节点
    ↓
UpdatePod() → 写入该 Pod 的 channel
    ↓
Pod Worker Goroutine 从 channel 读取
    ↓
调用 syncPod / syncTerminatingPod / syncTerminatedPod
    ↓
通过 CRI 与容器运行时交互（创建/启动/停止容器）
```

Pod 的生命周期状态机：

```
Sync（启动运行）→ Terminating（停止容器）→ Terminated（清理资源）
```

```go
type podWorkers struct {
    podUpdates    map[types.UID]chan struct{}   // 每个 Pod 的通知 channel
    podSyncStatuses map[types.UID]*podSyncStatus // 每个 Pod 的状态
    podSyncer     podSyncer                      // 实际的同步操作
}
```

### 4.6 Kubelet 与容器运行时的交互

Kubelet 不直接操作容器，而是通过 **CRI（Container Runtime Interface）** 与容器运行时通信：

```
Kubelet ──(gRPC)──→ CRI ──→ containerd/cri-o ──→ runc/kata ──→ 容器
```

- `RuntimeService`：管理容器生命周期（Create/Start/Stop/Remove）
- `ImageService`：管理镜像（Pull/Remove）

---

## 5. 节点之间如何通信

### 5.1 控制面通信：全部经过 API Server

Kubernetes 中**控制面通信全部走 HTTPS**，经过 API Server：

```
Kubelet ──(HTTPS)──→ API Server ──→ etcd
Scheduler ──(HTTPS)──→ API Server ──→ etcd
Controller ──(HTTPS)──→ API Server ──→ etcd
```

**所有通信都走 API Server 的内网地址**。Kubelet 启动时需要配置 API Server 地址（`--server` 参数或 kubeconfig），后续所有通信都通过这个地址。

### 5.2 数据面通信：Pod-to-Pod 和 Service 路由

Pod 之间直接通信，走的是 **CNI 插件配置的 Overlay 网络**（如 Flannel/Calico/Cilium）。

而 Service 到 Pod 的路由由 **kube-proxy** 维护：

```go
// kube-proxy 监听 Service 和 Endpoints 变化，维护转发规则
// 支持多种后端：iptables / IPVS / nftables
```

**iptables 模式**：kube-proxy 写入 iptables 规则实现 DNAT
**IPVS 模式**：kube-proxy 创建 IPVS 虚拟服务，内核级负载均衡
**nftables 模式**：新一代规则引擎

```
客户端 Pod ──(访问 ServiceIP:Port)──→ kube-proxy 规则 ──(DNAT)──→ 后端 Pod IP:Port
```

节点间 Pod 通信的前提是 **CNI 插件建立了一个覆盖网络**，使得不同节点的 Pod 处于同一个 IP 网段（或通过路由可达）。

### 5.3 是否使用内网地址

**是的，Kubernetes 默认使用节点的内网地址通信**：
- Kubelet 注册节点时上报节点的 Internal IP
- API Server 使用 Kubelet 的 Internal IP 与 Kubelet 通信（如 `kubectl logs`、`kubectl exec`）
- Pod 之间通过 CNI 分配的 Pod IP 直接通信
- 跨节点通信依赖 CNI 的隧道/VXLAN/BGP 路由

---

## 6. 节点状态上报 vs Master 主动查询

**答案：主要是 Worker 节点主动上报，Master 被动监控。**

### 6.1 Worker 主动上报（Kubelet → API Server）

#### 节点注册

Kubelet 启动时主动注册自己到 API Server（`pkg/kubelet/kubelet_node_status.go:52`）：

```go
func (kl *Kubelet) registerWithAPIServer(ctx context.Context) {
    for {
        node, err := kl.initialNode(ctx)  // 构造 Node 对象
        registered := kl.tryRegisterWithAPIServer(ctx, node)
        if registered {
            kl.registrationCompleted = true
            return
        }
        // 指数退避重试
        step = step * 2
        if step >= 7*time.Second { step = 7 * time.Second }
        time.Sleep(step)
    }
}
```

#### 节点状态定期更新

```go
// 定期同步节点状态到 Master (kubelet_node_status.go:449)
func (kl *Kubelet) syncNodeStatus(ctx context.Context) {
    kl.registerWithAPIServer(ctx)  // 确保已注册
    kl.updateNodeStatus(ctx)       // 更新状态
}
```

#### Lease 心跳

Kubelet 每隔几秒（默认 10s）更新一次 NodeLease，这是轻量级心跳：

```
Kubelet ──(每 10s)──→ 更新 Lease 对象 ──→ API Server
```

#### 上报的内容

```go
func (kl *Kubelet) initialNode(ctx context.Context) (*v1.Node, error) {
    node := &v1.Node{
        ObjectMeta: metav1.ObjectMeta{
            Name: string(kl.nodeName),
            Labels: map[string]string{
                v1.LabelHostname:   kl.hostname,       // 主机名
                v1.LabelOSStable:   goruntime.GOOS,    // 操作系统
                v1.LabelArchStable: goruntime.GOARCH,  // CPU 架构
            },
        },
    }
    // 合并用户自定义 Labels（--node-labels）
    for k, v := range kl.nodeLabels {
        node.ObjectMeta.Labels[k] = v
    }
    // 设置 Taints（如 --register-with-taints）
    node.Spec.Taints = nodeTaints
    // 设置 Capacity/Allocatable（CPU/内存/扩展资源）
    kl.setNodeStatus(ctx, node)
    return node, nil
}
```

### 6.2 Master 被动监控（Node Controller）

Node Controller（`pkg/controller/nodelifecycle/node_lifecycle_controller.go`）**不主动查询节点**，而是**监听 Kubelet 上报的 Lease 和 NodeStatus 变化**：

```go
// Node Controller 监控节点健康 (node_lifecycle_controller.go)
type Controller struct {
    nodeMonitorPeriod      time.Duration  // 检查间隔（默认 5s）
    nodeMonitorGracePeriod time.Duration  // 超时容忍（默认 40s）
    // ...
}
```

**超时判定逻辑**：
```
Kubelet 每 10s 更新 Lease
     ↓
Node Controller 每 5s 检查 Lease 最后更新时间
     ↓
如果超过 40s 没有更新 → 标记节点为 Unknown/NotReady
     ↓
对节点添加 Taint（如 node.kubernetes.io/unreachable:NoExecute）
     ↓
TaintEvictionController 驱逐节点上的 Pod
```

```go
// 节点状态到 Taint 的映射
nodeConditionToTaintKeyStatusMap = map[v1.NodeConditionType]map[v1.ConditionStatus]string{
    v1.NodeReady: {
        v1.ConditionFalse:   v1.TaintNodeNotReady,     // 节点未就绪
        v1.ConditionUnknown: v1.TaintNodeUnreachable,   // 节点不可达
    },
    v1.NodeMemoryPressure: {
        v1.ConditionTrue: v1.TaintNodeMemoryPressure,   // 内存压力
    },
    v1.NodeDiskPressure: {
        v1.ConditionTrue: v1.TaintNodeDiskPressure,     // 磁盘压力
    },
}
```

---

## 7. 守护进程：DaemonSet 机制

**是的，Kubernetes 有守护进程机制，就是 DaemonSet。**

DaemonSet Controller（`pkg/controller/daemon/daemon_controller.go`）确保每个（或指定的）节点上运行一个 Pod 副本。

### 7.1 工作原理

```
DaemonSet Controller Watch 节点和 DaemonSet 变更
    ↓
对每个 DaemonSet，遍历所有节点
    ↓
检查节点是否应该运行该 DaemonSet（NodeSelector/Affinity/Tolerations）
    ↓
如果节点上缺少该 Pod → 创建
如果节点上有多余的 Pod → 删除
```

```go
// DaemonSet Controller 监听多种资源变更
type DaemonSetsController struct {
    dsLister    appslisters.DaemonSetLister  // DaemonSet 列表
    podLister   corelisters.PodLister        // Pod 列表
    nodeLister  corelisters.NodeLister       // Node 列表
    // ...
}
```

### 7.2 DaemonSet 的调度方式

与普通 Pod 不同，DaemonSet Pod 有两种调度方式：

1. **Controller 直接设置 nodeName**（默认）：DaemonSet Controller 直接将 Pod 的 `spec.nodeName` 设为目标节点名，绕过 Scheduler
2. **通过 Scheduler 调度**：某些场景下通过 Scheduler 的正常调度流程

典型用途：kube-proxy、日志采集（Fluentd）、监控 Agent（Prometheus Node Exporter）、CNI 插件。

---

## 8. 集群动态伸缩机制

### 8.1 Pod 级别：Horizontal Pod Autoscaler (HPA)

HPA Controller（`pkg/controller/podautoscaler/horizontal.go`）根据指标自动调整 Pod 副本数：

```
HPA Controller 定期（默认 15s）执行
    ↓
从 Metrics Server / Custom Metrics API 获取指标
    ↓
计算期望副本数：desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
    ↓
更新 Deployment/ReplicaSet 的 replicas 字段
    ↓
ReplicaSet Controller 创建/删除 Pod
    ↓
Scheduler 调度新 Pod 到节点
```

```go
type HorizontalController struct {
    scaleNamespacer  scaleclient.ScalesGetter     // 用于调整副本数
    metricsClient    metricsclient.MetricsClient  // 指标客户端
    replicaCalc      *ReplicaCalculator           // 副本计算器
    // ...
}
```

### 8.2 节点级别：Cluster Autoscaler

**Cluster Autoscaler 不在 Kubernetes 核心代码中**，它是一个独立项目（`kubernetes/autoscaler`），但机制如下：

```
Cluster Autoscaler 持续监控
    ↓
发现有 Pod 处于 Pending 状态（无法调度）
    ↓
模拟调度：判断如果增加一个节点，该 Pod 能否被调度
    ↓
如果可以 → 调用云提供商 API 创建新节点
    ↓
新节点启动 Kubelet → 注册到集群 → Scheduler 调度 Pod

反向：
    ↓
发现某节点上所有 Pod 都可以迁移到其他节点
    ↓
标记节点为可缩容 → 驱逐 Pod → 调用云 API 删除节点
```

### 8.3 Vertical Pod Autoscaler (VPA)

VPA 调整 Pod 的资源请求（request/limit），而非副本数。同样不在核心仓库中。

---

## 9. 节点加入集群：硬件能力上报与 TEE/CVM 支持

### 9.1 节点加入流程

```
新节点启动 Kubelet
    ↓
registerWithAPIServer()：构造 Node 对象并注册
    ↓
上报 Labels（OS/Arch/Hostname/自定义 Labels）
    ↓
上报 Taints
    ↓
上报 Capacity/Allocatable（CPU/内存/大页）
    ↓
Device Plugin Manager 启动，注册设备插件
    ↓
设备插件上报扩展资源（GPU/FPGA/TEE 等）
    ↓
Kubelet 更新 Node 的 Capacity（包含扩展资源）
    ↓
定期 syncNodeStatus() 保持状态同步
```

### 9.2 Device Plugin 机制：硬件能力上报的核心

Device Plugin 是 K8s 中**节点上报硬件特性**的标准机制（`pkg/kubelet/cm/devicemanager/`）。

**架构**：
```
硬件设备（GPU/FPGA/TEE 设备）
    ↓
Device Plugin（独立进程，运行在节点上）
    ↓ gRPC
Device Plugin Manager（Kubelet 子组件）
    ↓
更新 Node.Status.Capacity
    ↓
Scheduler 在调度时考虑扩展资源
```

**Device Plugin 的生命周期**：
1. **Register**：Device Plugin 通过 Unix Socket 向 Kubelet 注册自己
2. **ListAndWatch**：上报设备列表和设备健康状态
3. **Allocate**：当 Pod 请求该资源时，执行设备分配逻辑

```go
// Device Plugin Manager 启动 (pkg/kubelet/cm/devicemanager/manager.go:366)
func (m *ManagerImpl) Start(logger klog.Logger, ...) error {
    m.readCheckpoint(logger)  // 恢复之前的分配信息
    return m.server.Start(logger)  // 启动 gRPC 注册服务
}
```

### 9.3 在集群中支持 CVM（机密虚拟机）

CVM（Confidential VM）需要 TEE（Trusted Execution Environment）硬件支持，如 Intel TDX、AMD SEV-SNP。在 K8s 中支持 CVM 的实现路径：

#### 方案 1：Device Plugin + 扩展资源

```
1. 拥有 TEE 能力的节点上部署 CVM Device Plugin
2. Device Plugin 注册扩展资源（如 intel.com/tdx 或 amd.com/sev）
3. Kubelet 更新 Node Capacity，如：
   node.Status.Capacity["intel.com/tdx"] = 4
4. Scheduler 在 Filter 阶段检查 Pod 是否请求了 TEE 资源
5. 只有拥有 TEE 资源的节点才能通过 Filter
```

#### 方案 2：Label + NodeSelector/NodeAffinity

```yaml
# 1. 节点加入时设置 Label（通过 --node-labels 或 cloud-provider）
# kubelet --node-labels=tee=true,tee-type=tdx

# 2. Pod 指定调度到 TEE 节点
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    tee: "true"
  # 或使用 nodeAffinity
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tee
            operator: In
            values: ["true"]
```

#### 方案 3：Taint + Toleration

```yaml
# 1. TEE 节点设置 Taint（通过 --register-with-taints）
# kubelet --register-with-taints=tee=true:NoSchedule

# 2. 只有声明了 Toleration 的 Pod 才能调度到 TEE 节点
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "tee"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### 9.4 Label 指定为 CVM 后如何路由到 TEE 节点

**路由机制的完整调用链**：

```
1. 节点加入集群
   Kubelet.initialNode() 设置 Labels（--node-labels=tee=tdx）
   → API Server 存储 Node 对象
   → Scheduler Informer 缓存更新

2. 用户创建 Pod（nodeSelector: tee=tdx）
   → API Server 存储 Pod
   → Scheduler Informer 检测到新 Pod

3. Scheduler 调度 Pod
   → findNodesThatFitPod()
   → findNodesThatPassFilters()
   → NodeAffinity Filter 插件执行：
     检查 pod.spec.nodeSelector 或 pod.spec.affinity
     对比每个节点的 Labels
     不匹配 → 返回 Unschedulable
   → 只有 Label 匹配的节点通过 Filter
   → Score 阶段打分 → 选最优节点
   → Bind 绑定 Pod 到节点

4. Kubelet Watch 到新 Pod
   → syncLoop 接收 ADD 事件
   → PodWorker 启动容器
   → CRI 调用容器运行时创建 CVM（如 kata-containers + TEE）
```

### 9.5 节点是否需要主动上报硬件特性

**是的，必须。** K8s 没有自动探测硬件能力的机制。节点加入时需要：

1. **Labels**：通过 `--node-labels` 参数或 cloud-provider 自动设置
2. **扩展资源**：通过 Device Plugin 注册
3. **Taints**：通过 `--register-with-taints` 参数

**云环境中的自动化**：
- Cloud Controller Manager 可以自动为节点设置 Labels（如 `node.kubernetes.io/instance-type`）
- 某些云提供商会自动检测 VM 的硬件特性并设置对应 Labels
- Node Feature Discovery (NFD) 是一个常用的开源工具，自动探测节点硬件并设置 Labels

---

## 10. 非 IDC 场景组网与跨网络集群

### 10.1 不用 IDC 机房如何组成集群

核心要求：**所有节点能通过内网/专线到达 API Server**。

几种方案：

#### 方案 1：Overlay VPN（如 Tailscale/ZeroTier）

```
节点 A (家庭网络) ──Tailscale──┐
节点 B (办公室)   ──Tailscale──┼── Tailscale 虚拟网络 ──→ Master API Server
节点 C (云服务器) ──Tailscale──┘
```

所有节点通过 Tailscale/ZeroTier 获得虚拟内网 IP，K8s 组件通过这个虚拟 IP 通信。

#### 方案 2：WireGuard + 自建 CA

```
各节点运行 WireGuard → 组成虚拟内网
→ K8s 组件使用 WireGuard 接口 IP 通信
→ 证书 SAN 包含 WireGuard IP
```

#### 方案 3：公网暴露 API Server + TLS 认证

```
Master 的 API Server 暴露在公网
→ 通过 TLS 双向认证保护
→ Kubelet 使用 kubeconfig 连接
```

### 10.2 关键挑战与解决方案

| 挑战 | 解决方案 |
|------|---------|
| 节点间 Pod 网络互通 | 使用支持跨网络的 CNI（如 Cilium with WireGuard）|
| API Server 高可用 | 多 Master + 负载均衡器 |
| 证书管理 | kubeadm 自动生成，或使用 cert-manager |
| 网络延迟 | 控制面可以容忍较高延迟，数据面需低延迟 |
| NAT 穿透 | Tailscale/WireGuard 自动处理 NAT 穿透 |

### 10.3 CNI 在跨网络场景下的选择

```
同一局域网：Flannel (VXLAN) → 简单高效
跨网络/跨机房：Cilium (WireGuard tunnel) → 安全 + 性能
大规模集群：Calico (BGP) → 路由直连，无封装开销
```

---

## 附录 A：核心源码文件索引

| 功能模块 | 源码路径 |
|---------|---------|
| Scheduler 主循环 | `pkg/scheduler/scheduler.go` |
| 调度算法 (Filter/Score) | `pkg/scheduler/schedule_one.go` |
| 调度框架接口 | `pkg/scheduler/framework/interface.go` |
| Kubelet 主逻辑 | `pkg/kubelet/kubelet.go` |
| Kubelet 节点注册 | `pkg/kubelet/kubelet_node_status.go` |
| Pod Workers | `pkg/kubelet/pod_workers.go` |
| Pod Config (Watch) | `pkg/kubelet/config/apiserver.go` |
| Node Lifecycle Controller | `pkg/controller/nodelifecycle/node_lifecycle_controller.go` |
| DaemonSet Controller | `pkg/controller/daemon/daemon_controller.go` |
| HPA Controller | `pkg/controller/podautoscaler/horizontal.go` |
| Device Plugin Manager | `pkg/kubelet/cm/devicemanager/manager.go` |
| Container Manager | `pkg/kubelet/cm/container_manager.go` |
| Informer/Reflector | `staging/src/k8s.io/client-go/tools/cache/controller.go` |
| SharedInformerFactory | `staging/src/k8s.io/client-go/informers/factory.go` |
| kube-proxy (iptables) | `pkg/proxy/iptables/proxier.go` |
| kube-proxy (IPVS) | `pkg/proxy/ipvs/proxier.go` |

## 附录 B：关键问题速答

| 问题 | 答案 |
|------|------|
| Master 怎么把任务分发给 Worker？ | Scheduler 将 Pod 绑定（Bind）到节点，Kubelet Watch 到变更后执行 |
| 集群如何动态伸缩？ | HPA 调 Pod 数，Cluster Autoscaler 调节点数 |
| Worker 如何根据指令操作 Pod？ | Kubelet 的 syncLoop + PodWorkers + CRI |
| 有没有守护进程？ | 有，DaemonSet 机制 |
| Worker 主动上报还是 Master 主动查？ | **Worker 主动上报**（Lease 心跳 + NodeStatus 更新），Master 被动监控 |
| 节点间如何通信？ | 控制面走 API Server（HTTPS），数据面走 CNI Overlay + kube-proxy |
| 用内网地址吗？ | 是的，默认使用节点内网 IP |
| 不用 IDC 能组集群吗？ | 能，用 Tailscale/WireGuard 组虚拟内网 |
| 如何支持 CVM？ | Device Plugin 注册扩展资源 + Label/Taint 调度 |
| 节点需要主动上报硬件特性吗？ | **是的**，通过 Labels + Device Plugin + Taints |
