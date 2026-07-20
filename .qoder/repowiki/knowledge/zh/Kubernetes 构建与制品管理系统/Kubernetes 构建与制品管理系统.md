---
kind: build_system
name: Kubernetes 构建与制品管理系统
category: build_system
scope:
    - '**'
source_files:
    - Makefile
    - go.work
    - hack/lib/golang.sh
    - hack/make-rules/build.sh
    - hack/make-rules/cross.sh
    - hack/make-rules/test.sh
    - hack/make-rules/update.sh
    - hack/make-rules/verify.sh
    - hack/build-go.sh
    - hack/build-cross.sh
    - hack/verify-all.sh
    - hack/update-all.sh
    - .go-version
---

## 系统概述

Kubernetes 采用 Go Workspace + Bash 脚本驱动的混合构建系统，以 Makefile 为统一入口，通过 hack/ 目录下的 Shell 脚本协调 Go toolchain、代码生成、测试和发布流程。整个构建体系围绕 go.work 多模块工作区展开，将核心运行时（pkg）、可执行组件（cmd）与独立发布的 staging 子模块统一管理。

## 核心架构

### 1. 多模块工作区管理
- 根 go.work: 聚合顶层 k8s.io/kubernetes 与 staging/src/k8s.io 下 30+ 个独立 Go 模块（api、apimachinery、client-go、kubelet 等），实现本地开发时源码替换
- 版本锁定: .go-version 指定 Go 工具链版本，hack/lib/golang.sh 中的 kube::golang::internal::verify_go_version 自动下载并切换至匹配版本
- 依赖隔离: hack/tools/go.mod 单独管理构建期工具依赖，避免污染主工作区

### 2. 构建目标分层
构建系统按产物类型严格分类：
- Server targets: kube-apiserver、kube-controller-manager、kube-scheduler、kube-proxy、kubeadm、kubelet 等控制面与节点侧二进制
- Client targets: kubectl、kubectl-convert 跨平台客户端
- Test targets: e2e.test、conformance image、ginkgo 等测试二进制
- Static binaries: 默认静态链接（CGO_ENABLED=0），支持 CGO 覆盖列表

### 3. 交叉编译矩阵
Server: linux/amd64, linux/arm64, linux/s390x, linux/ppc64le
Node:   + windows/amd64
Client: + darwin/*, windows/*, linux/386, linux/arm
Test:   同 Server + darwin/*, windows/*
通过 KUBE_BUILD_PLATFORMS 环境变量控制，hack/make-rules/cross.sh 按目标类别分批调用 make all。

## 关键文件与职责

### 入口层
- Makefile: 定义所有用户可见目标（all/test/update/verify/release/cross），转发到具体脚本
- hack/build-go.sh / build-cross.sh: 遗留兼容脚本，重定向到 make 目标

### 构建规则层
- hack/make-rules/build.sh: 设置环境并调用 kube::golang::build_binaries
- hack/make-rules/cross.sh: 按 server/node/client/test 四类目标分别触发交叉编译
- hack/make-rules/test.sh: 单元测试执行器，支持覆盖率、race detector、JUnit 报告
- hack/make-rules/update.sh: 批量运行 update-* 脚本（codegen、docs、openapi 等）
- hack/make-rules/verify.sh: 批量运行 verify-* 脚本（gofmt、typecheck、imports 等）

### 核心库
- hack/lib/golang.sh: 构建系统核心，包含平台枚举与目标分类函数、kube::golang::setup_platforms、kube::golang::build_binaries、kube::golang::is_statically_linked、kube::golang::set_platform_envs 等关键函数

### 验证与更新
- hack/verify-all.sh: 重定向到 make verify
- hack/update-all.sh: 重定向到 make update
- hack/verify-*.sh: 单个检查项（gofmt、typecheck、imports、boilerplate 等 40+ 个）
- hack/update-*.sh: 单个生成任务（codegen、docs、featuregates、openapi 等 20+ 个）

## 构建流程详解

### 本地开发构建
make all WHAT=./cmd/kubelet          # 构建单个组件
make test WHAT=./pkg/kubelet         # 运行单包测试
make ginkgo                          # 构建 ginkgo CLI

### 交叉编译
make cross                           # 全平台交叉编译
KUBE_BUILD_PLATFORMS="linux/amd64 darwin/arm64" make all  # 指定平台

### 测试执行
make test                            # 全量单元测试（含 race detector）
make test WHAT=./test/integration    # 集成测试
KUBE_COVER=y make test               # 带覆盖率收集

### 代码生成与校验
make update                          # 运行所有 update-* 脚本
make verify                          # 运行所有 verify-* 脚本
make quick-verify                    # 仅快速检查

## 设计决策与约定

1. Shell 优先: 构建逻辑全部用 Bash 实现，便于在 CI 容器环境中直接执行
2. 模块化组织: hack/lib/*.sh 提供可复用函数，make-rules/*.sh 封装具体流程
3. 输出隔离: 所有构建产物输出到 _output/ 目录，保持源码树干净
4. 工具链锁定: 通过 GOTOOLCHAIN 机制确保 Go 版本一致性，支持 devel/master 分支构建
5. 并行优化: 根据物理内存自动决定多平台并行度（阈值 20GB）
6. 覆盖率注入: 通过临时生成 dummy test 文件的方式对 main 包进行覆盖率收集
7. CGO 策略: 默认静态链接，通过 KUBE_CGO_OVERRIDES 精确控制例外

## 开发者规范

- 新增组件需在 kube::golang::server_targets() 或对应分类函数中注册
- 修改平台支持需同步更新 KUBE_SUPPORTED_*_PLATFORMS 数组
- 添加新 verify 脚本应遵循 hack/verify-*.sh 命名与返回值约定
- 新 update 脚本应加入 update.sh 的 BASH_TARGETS 列表
- 所有生成的代码需通过 make update 重新生成并提交变更