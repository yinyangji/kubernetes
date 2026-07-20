---
kind: logging_system
name: Kubernetes 日志系统：基于 klog/v2 的统一结构化日志框架
category: logging_system
scope:
    - '**'
source_files:
    - staging/src/k8s.io/component-base/logs/logs.go
    - staging/src/k8s.io/component-base/logs/api/v1/types.go
    - staging/src/k8s.io/component-base/logs/api/v1/options.go
    - staging/src/k8s.io/component-base/logs/json/json.go
    - staging/src/k8s.io/component-base/logs/klogflags/klogflags.go
---

## 系统概述

Kubernetes 使用 `k8s.io/klog/v2` 作为底层日志后端，并通过 `staging/src/k8s.io/component-base/logs` 提供统一的配置、注册表和运行时控制层。该框架支持 text/json 两种输出格式，通过 FeatureGate 控制高级特性，并兼容 Go 1.21+ 的 `log/slog`。

## 核心架构

### 配置模型（LoggingConfiguration）
- **Format**: `text`（默认）或 `json`，由 `--logging-format` 控制
- **Verbosity**: 0-3 级，对应 Info/Error/Warning/VDebug
- **VModule**: 按文件模式精细控制详细级别（仅 text 格式）
- **FlushFrequency**: 日志刷新间隔，默认 5 秒
- **Options**: 各格式的扩展选项（SplitStream、InfoBufferSize）

### 初始化流程
```go
// 标准组件启动顺序
logs.AddFlags(pflag.CommandLine)                    // 注册命令行参数
opts := logs.NewOptions()                           // 创建默认配置
opts.ValidateAndApply(featureGates)                 // 应用配置
defer logs.FlushLogs()                              // 程序退出时刷新
```

### 日志格式工厂
- **Text 格式**: 基于 klog 原生文本输出，支持结构化键值对
- **JSON 格式**: 基于 zap + logr 适配器，输出结构化 JSON 对象
- **Factory 接口**: 统一创建 logger 和 RuntimeControl 实例

## 关键文件与包

- `staging/src/k8s.io/component-base/logs/logs.go`: 顶层 API，AddFlags/InitLogs/FlushLogs
- `staging/src/k8s.io/component-base/logs/api/v1/types.go`: 配置结构体定义
- `staging/src/k8s.io/component-base/logs/api/v1/options.go`: 配置验证和应用逻辑
- `staging/src/k8s.io/component-base/logs/json/json.go`: JSON 格式实现
- `staging/src/k8s.io/component-base/logs/klogflags/klogflags.go`: klog 标志兼容层
- `staging/src/k8s.io/component-base/logs/example/`: 使用示例和测试

## 设计决策

1. **向后兼容**: 保留 `-v`、`-vmodule` 等传统 klog 标志
2. **渐进增强**: 通过 FeatureGate 逐步启用 ContextualLogging、Alpha/Beta 选项
3. **流分离**: 支持将错误信息重定向到 stderr，信息日志到 stdout
4. **性能优化**: JSON 格式禁用 fsync，text 格式支持缓冲写入
5. **测试友好**: 提供 ResetForTest 重置全局状态

## 开发者规范

- 使用 `klog.InfoS()` / `klog.ErrorS()` 进行结构化日志记录
- 避免在热路径中使用高 verbosity 级别
- 通过 context 传递上下文信息（需启用 ContextualLogging feature gate）
- 生产环境建议使用 JSON 格式便于日志收集和分析
- 自定义组件应遵循相同的日志配置和初始化模式