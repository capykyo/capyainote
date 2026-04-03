# Data Studio 实操任务清单

> 目标：全面了解并掌握 Directus Data Studio 的操作规则、边界与能力。
> 对应学习计划：阶段一 — 熟悉 Data Studio 界面操作
>
> **前置条件：** 完成 PostgreSQL + Directus 本地环境搭建 → [docker-setup.md](./docker-setup.md)

---

## 模块一：Settings — 数据建模

> 理解数据结构是一切操作的基础

- [ ] 在 Settings → Data Model 新建 Collection `articles`，观察数据库同步生成对应表
- [ ] 为 `articles` 添加以下字段，覆盖主要类型：
  - `title`（Input，必填）
  - `body`（WYSIWYG 富文本）
  - `status`（Dropdown，值：draft / published / archived）
  - `published_at`（Datetime）
  - `cover`（Image，关联 Directus Files）
- [ ] 新建 Collection `categories`，在 `articles` 中添加 M2O 关系字段指向 `categories`
- [ ] 修改字段顺序、设置默认值、设置必填验证
- [ ] **边界测试**：删除一个有数据的字段，观察会发生什么

---

## 模块二：Explore — 内容浏览与筛选

> 掌握如何查找、筛选、排序数据

- [ ] 切换 `articles` 视图模式：Table / Card / Gallery，感受区别
- [ ] 使用筛选器：按 `status = published` 过滤，叠加 `published_at` 范围筛选
- [ ] 保存自定义视图布局（Filter Preset），供下次复用
- [ ] 使用搜索功能，观察搜索范围覆盖哪些字段
- [ ] 批量选中多条记录，测试批量编辑和批量删除
- [ ] **边界测试**：对空 Collection 执行筛选、导出操作

---

## 模块三：Editor — 内容创建与编辑

> 掌握单条记录的完整操作能力

- [ ] 新建一篇 `articles`，填写所有字段，关联一个 `category`，保存
- [ ] 测试 Revision（历史版本）：修改内容后保存，查看修改历史，尝试回滚
- [ ] 测试文件上传：在 `cover` 字段上传图片，观察文件存入 Directus Files
- [ ] 添加 O2M 字段（如 `comments`），测试嵌套记录的新建与编辑
- [ ] **边界测试**：不填必填字段直接保存，观察验证提示

---

## 模块四：Files — 文件管理

> 理解 Directus 统一文件管理机制

- [ ] 手动上传多种类型文件（图片、PDF、视频）
- [ ] 新建文件夹，将文件分类整理
- [ ] 点击图片，查看自动生成的 transforms URL（尺寸、格式参数）
- [ ] 将同一张图片引用于多篇 `articles` 的 `cover`，然后删除原图，观察引用关系
- [ ] **边界测试**：上传超大文件，观察是否有大小限制提示

---

## 模块五：User Management & Roles — 权限体系

> 商用部署最关键的模块

- [ ] 新建角色 `editor`，配置权限：
  - 只能读写 `articles`，不能访问 `categories` 和 Settings
  - 字段级权限：隐藏 `published_at` 的写权限
- [ ] 新建用户，分配 `editor` 角色，用该账号登录验证权限是否生效
- [ ] 配置 Public 角色：开放 `articles` 只读权限，用浏览器无登录状态调用 API 验证
- [ ] **边界测试**：editor 角色尝试访问 Settings，观察被拒绝的方式

---

## 模块六：Insights — 数据看板

> 了解数据可视化能力

- [ ] 新建 Dashboard，添加 Metric 面板（统计 articles 总数）
- [ ] 添加 Table 面板，显示最新 5 篇 articles
- [ ] 观察看板的刷新机制

---

## 完成标准

完成以上任务后，应能回答：

- Collection / Field / Item 如何对应数据库结构？
- 角色权限最细可以控制到什么粒度（字段级 / 记录级）？
- Directus Files 与普通字段存储有何不同？
- Public API 访问的边界在哪里？

---

## 进度

| 模块 | 状态 | 备注 |
|------|------|------|
| 一：Settings 数据建模 | 未开始 | |
| 二：Explore 浏览筛选 | 未开始 | |
| 三：Editor 内容编辑 | 未开始 | |
| 四：Files 文件管理 | 未开始 | |
| 五：Roles 权限体系 | 未开始 | 重点 |
| 六：Insights 看板 | 未开始 | |
