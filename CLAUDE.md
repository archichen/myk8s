# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# 守则
1. 修改完毕测试没有问题需要使用git commit提交修改
2. 遇到不懂的技术问题，可以使用`context7 mcp tool`查询文档
3. 使用kubectl管理rancher集群
4. 因为本项目会使用fleet在多集群上部署，请尽量使用fleet来部署
5. 通过region标签选择环境，一共三个环境：prod\test\dev
6. 三个环境都是单节点k8s集群，所以每个集群上的服务不需要考虑多副本

## 项目概述

这是一个基于 Rancher Fleet 的 Kubernetes 应用部署管理项目。项目使用 GitOps 模式，通过 Fleet Bundle 来管理多集群的应用部署，主要关注 PostgreSQL 数据库的高可用部署和 API 网关的统一管理。

## 核心架构

### 目录结构
- `apps/` - 应用级别的部署配置
  - `pg/` - PostgreSQL 数据库集群部署配置
  - `kong/` - Kong API 网关部署配置
  - `test/` - 测试应用配置
- `manifests/` - 基础设施和 Operator 部署配置
  - `cloudnative-pg/` - CloudNativePG Operator 部署配置
  - `kong/` - Kong Operator 部署配置
  - `test2/` - 调试测试配置

### 技术栈
- **Rancher Fleet** - GitOps 多集群管理工具
- **CloudNativePG** - PostgreSQL 在 Kubernetes 上的原生 Operator
- **Kong Gateway** - 云原生 API 网关
- **Kubernetes** - 容器编排平台
- **Helm** - 包管理器（用于部署 Operator）

### Bundle 依赖关系
1. `cloudnative-pg-bundle` - 部署 CloudNativePG Operator，作为数据库基础设施依赖
2. `kong-operator-bundle` - 部署 Kong Gateway Operator，作为 API 网关基础设施依赖
3. `pg-bundle` - 部署 PostgreSQL 数据库集群，依赖于 CloudNativePG Operator
4. `kong-bundle` - 部署 Kong Gateway 实例，依赖于 Kong Operator
5. 测试 bundles - 用于调试和验证 Fleet 配置

## 开发工作流程

### 部署顺序
1. 首先部署 `manifests/cloudnative-pg/fleet.yaml` 安装数据库 Operator
2. 然后部署 `manifests/kong/fleet.yaml` 安装 Kong Gateway Operator
3. 接着部署 `apps/pg/fleet.yaml` 创建数据库集群
4. 最后部署 `apps/kong/fleet.yaml` 创建 Kong Gateway 实例
5. 使用测试配置验证部署状态

### Bundle 配置规范
- 每个 `fleet.yaml` 文件必须定义唯一的 `name`
- 使用 `defaultNamespace` 指定目标命名空间
- Operator 部署使用 `helm` 配置块
- 应用部署使用 `yaml` 配置块定义 Kubernetes 资源
- 通过 `dependsOn` 建立依赖关系（目前被注释）

### 环境管理
- 使用 `targetCustomizations.clusterSelector` 进行环境区分
- 支持通过标签选择目标集群（如 `env: development`）
- 当前配置为部署到所有集群

## 配置要点

### PostgreSQL 集群配置
- 单节点实例（可扩展为高可用）
- 20Gi 存储空间
- 部署在 `database-apps` 命名空间
- 使用 CloudNativePG CRD `postgresql.cnpg.io/v1`

### Operator 版本管理
- CloudNativePG 版本锁定为 `0.21.3`
- 使用官方 Helm 仓库 `https://cloudnative-pg.github.io/charts`
- 部署在 `cnpg-system` 命名空间
- Kong Gateway 版本锁定为 `2.16.0`
- 使用官方 Helm 仓库 `https://charts.konghq.com`
- 部署在 `kong-system` 命名空间

### Kong Gateway 配置
- 无数据库模式部署（适合单节点环境）
- 支持开发、测试、生产三个环境的不同配置
- 集成 Ingress Controller，支持 Kubernetes Ingress 资源
- 提供管理员 API 和代理服务的 NodePort 访问
- 生产环境支持 LoadBalancer 类型服务
- 包含测试用的 Ingress 和 Service 配置

## 调试和测试

项目包含多个测试配置用于验证 Fleet 功能：
- `apps/test/fleet.yaml` - 应用级别测试
- `manifests/test2/fleet.yaml` - 基础设施级别测试
- `apps/kong/templates/test.yaml` - Kong Gateway 功能测试
- 使用 ConfigMap 验证部署成功状态