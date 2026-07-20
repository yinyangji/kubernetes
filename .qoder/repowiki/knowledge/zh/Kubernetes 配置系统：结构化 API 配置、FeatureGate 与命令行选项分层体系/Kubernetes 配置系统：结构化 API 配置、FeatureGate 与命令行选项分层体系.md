---
kind: configuration_system
name: Kubernetes 配置系统：结构化 API 配置、FeatureGate 与命令行选项分层体系
category: configuration_system
scope:
    - '**'
source_files:
    - staging/src/k8s.io/apiserver/pkg/features/kube_features.go
    - staging/src/k8s.io/apiserver/pkg/server/options/recommended.go
    - staging/src/k8s.io/apiserver/pkg/server/options/feature.go
    - pkg/controller/apis/config/types.go
    - pkg/scheduler/apis/config/types.go
    - pkg/kubelet/apis/config/types.go
    - cmd/kubeadm/app/componentconfigs/configset.go
    - cmd/kubeadm/app/util/config/strict/strict.go
---

## 1. 整体架构概览

Kubernetes 的配置系统采用**“三层叠加”**模式，从底层到顶层依次为：

- **FeatureGate（功能开关）**：全局特性门控，控制新功能的启用/禁用及版本生命周期。
- **结构化配置文件（YAML/JSON）**：以 `k8s.io/apimachinery` 的 runtime.Object 形式声明式描述组件配置，支持多版本、默认值生成、严格反序列化校验。
- **命令行参数（pflag/cobra）**：通过 `AddFlags` + `ApplyTo` 模式将 flag 绑定到 Options 结构体，再应用到运行时配置对象。

这三层在启动时按顺序合并：先解析 FeatureGate，再加载配置文件并应用默认值，最后用命令行参数覆盖，形成最终运行期配置。

## 2. 核心子系统与关键文件

### 2.1 FeatureGate 系统
- **定义与注册**：`staging/src/k8s.io/apiserver/pkg/features/kube_features.go`
  - 每个 FeatureGate 以常量形式声明，附带 owner、kep、beta/deprecated 注释；
  - `defaultVersionedKubernetesFeatureGates` 中维护每个 gate 的版本化规格（Alpha/Beta/GA/Deprecated），支持 `LockToDefault` 锁定默认值。
- **使用方式**：各组件通过 `utilfeature.DefaultMutableFeatureGate` 读取当前状态。

### 2.2 结构化配置类型（API-style Config）
Kubernetes 为每个组件定义了独立的配置 API，遵循统一目录结构：

| 组件 | 配置包路径 | 主类型 |
|------|-----------|--------|
| kube-apiserver | `staging/src/k8s.io/apiserver/pkg/server/options/*.go` | 由 `RecommendedOptions` 组合多个子选项 |
| kube-controller-manager | `pkg/controller/apis/config/types.go` | `KubeControllerManagerConfiguration` |
| kube-scheduler | `pkg/scheduler/apis/config/types.go` | `KubeSchedulerConfiguration` |
| kubelet | `pkg/kubelet/apis/config/types.go` | `KubeletConfiguration` |
| kubeadm 组件配置 | `cmd/kubeadm/app/componentconfigs/*.go` | 基于 `ComponentConfig` 接口的插件式加载 |

每个配置包通常包含：
- `types.go`：定义配置结构体，带 `+k8s:deepcopy-gen` 等代码生成标签；
- `v1/v1alpha1/...`：版本化实现，含 `defaults.go`、`conversion.go`、`register.go`；
- `scheme/`：注册 Scheme，支持跨版本编解码；
- `validation/`：字段级校验逻辑。

### 2.3 命令行选项绑定（Options → Flags → ApplyTo）
- **推荐选项聚合器**：`staging/src/k8s.io/apiserver/pkg/server/options/recommended.go`
  - `RecommendedOptions` 聚合 Etcd/Serving/Authz/Audit/Features/CoreAPI/Admission/EgressSelector/Traces 等子选项；
  - 统一的 `AddFlags(fs)` 递归注册所有 flag；
  - `ApplyTo(config)` 按依赖顺序将选项应用到 `server.RecommendedConfig`。
- **典型模式**：每个 `*Options` 结构体实现 `AddFlags`、`ApplyTo`、`Validate` 三个方法，如 `FeatureOptions`（`feature.go`）。

### 2.4 配置加载与合并流程
- **kubeadm 组件配置集**：`cmd/kubeadm/app/componentconfigs/configset.go`
  - 通过 `handler` 抽象封装 GroupVersion、Unmarshal/Marshal、FromCluster/FromDocumentMap 等能力；
  - `known` 列表集中注册支持的组件配置处理器（kube-proxy、kubelet）；
  - `FetchFromCluster` / `FetchFromDocumentMap` 从 ConfigMap 或文档映射加载，并调用 `Default` 填充默认值。
- **严格反序列化校验**：`cmd/kubeadm/app/util/config/strict/strict.go`
  - 使用 `json.NewSerializerWithOptions(..., Strict: true)` 确保未识别字段报错，防止拼写错误静默忽略。

### 2.5 加密配置与外部密钥管理
- `staging/src/k8s.io/apiserver/pkg/server/options/encryptionconfig/`
  - 支持 AES-CBC、AES-GCM、KMS/KMSv2、SecretBox 等多种后端；
  - 提供控制器自动轮换密钥、指标采集等功能。

## 3. 设计约定与开发规则

1. **新增 FeatureGate 的步骤**
   - 在 `kube_features.go` 中声明常量，并在 `defaultVersionedKubernetesFeatureGates` 中登记版本化规格；
   - 在相关代码中使用 `utilfeature.DefaultFeatureGate.Enabled(featureName)` 判断。

2. **新增组件配置类型的步骤**
   - 在对应包的 `types.go` 中定义配置结构体，添加 `+k8s:deepcopy-gen` 标签；
   - 创建 `v1`（或 alpha/beta）版本目录，实现 `defaults.go`、`conversion.go`、`register.go`；
   - 在 `scheme/scheme.go` 中注册 Scheme；
   - 在 `validation/validation.go` 中添加字段校验；
   - 在组件启动入口中通过 `options.RecommendedOptions` 或自定义 Options 暴露 flag，并调用 `ApplyTo` 应用。

3. **Flag 绑定规范**
   - 使用 `github.com/spf13/pflag` 而非标准库 `flag`；
   - 每个 Options 结构体必须实现 `AddFlags(*pflag.FlagSet)`、`ApplyTo(interface{}) error`、`Validate() []error`；
   - 通过 `RecommendedOptions` 聚合，避免在 main 函数中散落 flag 注册。

4. **配置文件格式与兼容性**
   - 配置文件使用 YAML/JSON，通过 `runtime.Codec` 编解码；
   - 利用 `Strict` 模式反序列化，拒绝未知字段；
   - 多版本共存时，通过 conversion 链自动升级到首选版本。

5. **安全敏感字段标记**
   - 对证书私钥、令牌等敏感字段使用 `datapolicy:"security-key"` 标签，便于审计和日志脱敏。

## 4. 与其他系统的交互

- **FeatureGate 与 Admission/Webhook**：许多 admission 插件的行为受 FeatureGate 控制；
- **EncryptionConfiguration 与 etcd**：apiserver 在写入前对指定资源进行透明加密；
- **kubeadm ComponentConfigs 与 ClusterConfiguration**：kubeadm 将 kubelet/kube-proxy 的配置作为独立 ConfigMap 存储，支持升级时自动迁移。
