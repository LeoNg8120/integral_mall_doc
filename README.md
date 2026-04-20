# 积分商城开发文档

[![GitBook](https://img.shields.io/badge/GitBook-Docs-blue)](https://www.gitbook.com)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

> 积分商城（Integral Mall）是一个面向企业内部的积分激励与商品兑换平台，基于 go-zero 微服务框架与 React 19 构建。

## 核心价值

- **积分激励闭环**：贡献申请 → AI 评分 → 双级审核 → 积分发放 → 商品兑换
- **微服务架构**：API 网关 + 四路 RPC（用户/积分/商品/订单）
- **前后端分离**：Go 后端 + React 19 前端，全栈 TypeScript

## 文档导航

- **快速入门**：项目概览、环境搭建
- **系统架构**：微服务设计、数据库、错误码规范
- **后端业务**：积分申请、AI 评分、订单兑换、RBAC 权限
- **API 网关**：JWT 认证、权限中间件、三层架构
- **前端工程**：React 架构、Zustand 状态管理、Ant Design 主题定制
- **数据模型**：GORM 模型、乐观锁、并发控制
- **测试体系**：单元测试、Playwright E2E、Vitest 组件测试
- **部署运维**：Docker Compose、Kubernetes CI/CD

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端框架 | Go 1.26 + go-zero |
| RPC | gRPC + Protobuf |
| 数据库 | MySQL 8.0 + GORM |
| 缓存 | Redis 7 |
| 前端框架 | React 19 + TypeScript |
| UI 库 | Ant Design 6 |
| 状态管理 | Zustand |
| 构建工具 | Vite + pnpm |
| 测试 | Go test + Playwright + Vitest |
| 部署 | Docker + Kubernetes + Kustomize |

## 快速开始

```bash
# 本地启动（需先安装 Docker）
make run

# 运行后端测试
make test-backend

# 运行前端测试
make test-frontend
```

## 贡献指南

本项目采用 OpenSpec 变更管理规范，详见 [OpenSpec 规范与功能基线管理](./开发工作流/增量变更提案：OpenSpec规范与功能基线管理.md)。

---
*由 Integral Mall 开发团队维护 · 生成日期：2026-04-19*
