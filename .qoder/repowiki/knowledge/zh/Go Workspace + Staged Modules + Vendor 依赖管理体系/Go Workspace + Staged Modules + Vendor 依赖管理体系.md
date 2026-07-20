---
kind: dependency_management
name: Go Workspace + Staged Modules + Vendor 依赖管理体系
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
    - go.work
    - go.work.sum
    - hack/update-vendor.sh
    - hack/pin-dependency.sh
    - hack/verify-vendor.sh
    - hack/lint-dependencies.sh
    - hack/update-vendor-licenses.sh
    - hack/tools/go.mod
    - hack/tools/go.sum
    - hack/tools/go.work
---

## 体系概述

Kubernetes 采用 Go workspace（go.work）+ staged Go modules + vendor 目录的三层依赖管理策略：顶层 k8s.io/kubernetes 作为聚合模块，通过 replace 指向 staging/src/k8s.io/* 下的独立子模块；所有模块统一由 go.work 编排，并通过 hack/update-vendor.sh 生成只读 vendor 树。

## 关键文件与位置

- 顶层模块声明与替换规则：go.mod、go.sum、go.work、go.work.sum
- 依赖更新/校验脚本：hack/update-vendor.sh、hack/pin-dependency.sh、hack/verify-vendor.sh、hack/lint-dependencies.sh
- 各 staging 子模块：staging/src/k8s.io/<module>/go.mod（api、apimachinery、client-go、apiserver、kubelet 等约 30 个）
- 第三方许可证清单：LICENSES/vendor/、LICENSES/third_party/
- 构建期工具集：hack/tools/go.mod、hack/tools/go.sum、hack/tools/go.work

## 架构与约定

1. Staged 模块化：核心库以独立 go.mod 发布到 staging/src/k8s.io/，对外通过 k8s.io/* 路径暴露；根模块通过 replace ( k8s.io/api => ./staging/src/k8s.io/api ... ) 在本地开发时指向源码，发布时再切回远端版本。

2. Workspace 统一编排：go.work 显式 use 所有 staging 子模块，使 go build/test 能在多模块工作区内解析跨模块引用，避免每个子模块重复维护 replace 链。

3. Vendor 快照：hack/update-vendor.sh 执行 go work sync -> go mod tidy -> go work vendor，将完整依赖图复制到 vendor/，CI 通过 hack/verify-vendor.sh 对比 diff 确保可复现。

4. 版本锁定与升级流程：新增/升级外部依赖使用 hack/pin-dependency.sh <module> <sha-or-tag>，自动向根与各 staging 模块注入 require/replace，随后运行 hack/update-vendor.sh 重建 vendor。禁止 GOPROXY=off 环境下运行依赖脚本，强制从网络拉取最新元数据。

5. 循环依赖防护：update-vendor.sh 末尾检查并拒绝任何 staging -> k8s.io/kubernetes 或间接回环依赖，保证 staging 模块单向向外暴露。

6. godebug 与 Go 版本一致性：根 go.mod 中 go 1.26.0 与 godebug default=go1.26 会被 update-vendor.sh 同步写入所有 staging 子模块及 go.work，确保全仓库语言行为一致。

7. 许可证合规：每次 vendor 更新后调用 hack/update-vendor-licenses.sh 汇总第三方 LICENSE 到 LICENSES/vendor/，并由 hack/verify-licenses.sh 校验。

## 开发者应遵循的规则

- 不要直接编辑 go.mod / go.sum / go.work 头部注释块，它们被标记为 generated，应通过 pin-dependency.sh 与 update-vendor.sh 变更。
- 引入新依赖必须走 hack/pin-dependency.sh，并在提交前运行 hack/verify-vendor.sh 确保 vendor 与 go.mod 一致。
- 新增 staging 子模块后，需在 go.work 中添加 use 指令（通常由 update-vendor.sh 自动完成）。
- 禁止在 staging 模块中反向 import k8s.io/kubernetes，否则 update-vendor.sh 会失败。
- 本地开发建议保持 GOWORK 开启，使用 go work 模式进行跨模块调试；发布构建则依赖 vendor 快照保证确定性。