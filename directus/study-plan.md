# Directus 学习计划

> **最终目标：** 能够独立部署、配置并维护一套用于商业用途的 Directus 实例。

---

## 阶段一：认识 Directus（基础概念）

**目标：** 理解 Directus 是什么、能做什么、适合什么场景。

- [x] 阅读平台概览：Data Engine vs Data Studio
- [x] 了解核心概念：Collections、Fields、Items、Roles & Permissions
- [x] 跑通官方 Quickstart（本地 Docker 启动）
- [ ] 熟悉 Data Studio 界面基本操作 → 详见 [datastudio-tasks.md](./datastudio-tasks.md)

**参考：** https://directus.io/docs/getting-started/overview

---

## 阶段二：API 与数据建模

**目标：** 能用 REST/GraphQL API 进行 CRUD，理解数据模型设计。

- [ ] REST API 基本用法（Items、Files、Auth）
- [ ] GraphQL API 基本用法
- [ ] JavaScript SDK 使用
- [ ] 关系型字段：M2O、O2M、M2M、M2A
- [ ] 自定义字段类型与验证规则

**参考：** https://directus.io/docs/api

---

## 阶段三：权限与认证

**目标：** 配置多角色权限体系，理解生产环境的认证安全。

- [ ] Roles & Permissions 配置（字段级权限）
- [ ] 用户认证：Token、Cookies、SSO
- [ ] Public 角色安全配置
- [ ] IP 白名单 / Rate Limiting 策略

**参考：** https://directus.io/docs/auth

---

## 阶段四：自托管部署

**目标：** 在服务器上完成生产级部署，可对外提供稳定服务。

- [ ] Docker Compose 部署方案
- [ ] 环境变量配置（数据库、缓存、邮件、存储）
- [ ] 数据库选型与连接（PostgreSQL 推荐）
- [ ] 文件存储配置：本地 / S3 / 云存储
- [ ] Redis 缓存与 Session 配置
- [ ] 反向代理配置（Nginx / Caddy）+ HTTPS
- [ ] 备份与迁移策略

**参考：** https://directus.io/docs/self-hosting

---

## 阶段五：自动化与扩展

**目标：** 利用 Flows 和 Extensions 满足业务定制需求。

- [ ] Flows（自动化工作流）：触发器、操作节点
- [ ] Webhooks 配置
- [ ] Extensions 开发：自定义 Hook / Endpoint / Display
- [ ] Insights（数据看板）搭建

**参考：** https://directus.io/docs/automate

---

## 阶段六：生产运维

**目标：** 保障商用实例的稳定性、安全性与可扩展性。

- [ ] 日志监控配置
- [ ] 数据库定期备份方案
- [ ] 升级策略与版本管理
- [ ] 多环境管理（开发 / 测试 / 生产）
- [ ] 性能优化：查询优化、缓存策略

---

## 进度追踪

| 阶段 | 状态 | 备注 |
|------|------|------|
| 一：基础概念 | 进行中 | Docker 已跑通，Data Studio 实操进行中 → [任务清单](./datastudio-tasks.md) |
| 二：API 与数据建模 | 未开始 | |
| 三：权限与认证 | 未开始 | |
| 四：自托管部署 | 未开始 | 核心目标 |
| 五：自动化与扩展 | 未开始 | |
| 六：生产运维 | 未开始 | |
