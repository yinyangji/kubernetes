---
kind: error_handling
name: Kubernetes 错误处理体系：StatusError、Aggregate 与 API 响应转换
category: error_handling
scope:
    - '**'
source_files:
    - staging/src/k8s.io/apimachinery/pkg/api/errors/errors.go
    - staging/src/k8s.io/apimachinery/pkg/util/errors/errors.go
    - staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/status.go
    - staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/writers.go
    - staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/errors.go
---

## 1. 系统/方法概述

Kubernetes 的错误处理围绕三层协作展开：

- **API 层错误类型**：`staging/src/k8s.io/apimachinery/pkg/api/errors` 定义 `StatusError`，实现 `APIStatus` 接口，将 Go error 映射到 `metav1.Status`（包含 HTTP 状态码、Reason、Details/Causes），并提供大量工厂函数（`NewNotFound`、`NewForbidden`、`NewConflict`、`NewInvalid`、`NewTooManyRequests`、`NewServerTimeout`、`NewInternalError` 等）和判断辅助函数（`IsNotFound`、`IsAlreadyExists`、`IsConflict`、`IsUnauthorized`、`IsForbidden`、`IsTimeout`、`IsServiceUnavailable`、`SuggestsClientDelay` 等）。所有 Is* 函数均支持 errors.As 包装链。
- **聚合错误**：`staging/src/k8s.io/apimachinery/pkg/util/errors` 提供 `Aggregate` 接口与 `NewAggregate`，用于在并行 goroutine 或批量操作中汇总多个错误；提供 `FilterOut`、`Flatten`、`Reduce`、`AggregateGoroutines` 等工具，并兼容 `errors.Is()`。
- **API Server 响应转换**：`staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters` 中的 `ErrorToAPIStatus` 负责把任意 error 转换为 `metav1.Status`，再经 `ErrorNegotiated`/`WriteObjectNegotiated` 序列化为 JSON/Protobuf 返回；`Forbidden`/`InternalError` 等快捷函数直接写出标准 REST 错误体。

非 API 路径的组件（如 kubelet、controller-manager、kubeadm）则使用 `utilruntime.HandleError` 记录结构化日志，并在配置/初始化阶段用 `panic` 表达不可恢复的编程错误。

## 2. 关键文件与包

- `staging/src/k8s.io/apimachinery/pkg/api/errors/errors.go` — StatusError、APIStatus、所有 New*/Is* 工厂与判定器
- `staging/src/k8s.io/apimachinery/pkg/util/errors/errors.go` — Aggregate、NewAggregate、FilterOut、Flatten、Reduce、AggregateGoroutines
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/status.go` — ErrorToAPIStatus：error → metav1.Status
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/writers.go` — ErrorNegotiated / WriteObjectNegotiated：序列化并写出错误响应
- `staging/src/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/errors.go` — Forbidden / InternalError 快捷写入
- `staging/src/k8s.io/apimachinery/pkg/util/runtime/runtime.go` — HandleError / HandleErrorWithContext（全局错误日志入口）
- `pkg/apiserver/handlers/responsewriters/...`（顶层 pkg 中对应代码已迁移至 staging）

## 3. 架构与约定

- **REST 边界统一为 StatusError**：所有通过 API Server 暴露的错误必须构造为 `StatusError`，以便客户端按 Reason/Code/Causes 做精确分支；内部业务错误应在存储/注册表层转换为 StatusError。
- **可组合错误用 Aggregate**：多 goroutine 或批量操作收集到的错误应聚合为 `Aggregate`，上层再用 `Reduce` 或逐条处理；避免直接拼接字符串。
- **错误诊断信息走 Details.Causes**：字段级校验错误通过 `field.ErrorList` 转为 `StatusCause`；通用附加信息通过 `NewGenericServerResponse` 注入 `UnexpectedServerResponse` Cause。
- **HTTP 语义与重试提示**：`RetryAfterSeconds` 由 `ErrorNegotiated` 自动写回 `Retry-After` 头；`SuggestsClientDelay` 供上层决定是否退避。
- **未知错误兜底**：`ErrorToAPIStatus` 对非 `APIStatus` 错误一律降级为 500 + `ReasonUnknown`，并通过 `utilruntime.HandleError` 记录堆栈，防止静默失败。
- **panic 的使用范围**：仅用于启动期参数校验、控制器注册、测试断言等“不可能发生”的编程错误；请求路径不 panic。

## 4. 开发者规则

1. **对外 API 错误**：使用 `apimachinery/pkg/api/errors.New*` 系列工厂返回 `*StatusError`，不要直接返回 `fmt.Errorf`。
2. **错误判断**：使用对应的 `Is*` 辅助函数（`IsNotFound`、`IsConflict`、`IsUnauthorized` 等），它们能穿透 `errors.Wrap`。
3. **批量/并发错误**：收集到 `[]error` 后调用 `utilerrors.NewAggregate`，必要时用 `Reduce` 解包单元素情况。
4. **字段校验**：累积 `field.ErrorList`，最终通过 `NewInvalid(qualifiedKind, name, errs)` 一次性返回。
5. **API Server 中间件/处理器**：遇到无法转换为 Status 的错误，交由 `responsewriters.ErrorToAPIStatus` + `ErrorNegotiated` 处理，不要自行写 http.Error。
6. **非 API 路径组件**：使用 `utilruntime.HandleError(err)` 记录错误；仅在初始化/配置阶段用 `panic` 表达致命错误。
7. **禁止裸 fmt.Errorf 跨 API 边界传播**：若需携带上下文，使用 `%w` 包装，让上层 `Is*` 仍能匹配根因。
8. **利用 Retry-After**：当返回 `ServerTimeout`/`TooManyRequests` 时设置 `RetryAfterSeconds`，上层可通过 `SuggestsClientDelay` 自动退避。
