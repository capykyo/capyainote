# PostgreSQL 命令行速查（配合 Directus Docker 环境）

> 前置：已按 [Docker 环境搭建](../docker-setup.md) 启动 Compose，数据库服务名为 **`database`**，库与用户均为 **`directus`**。

以下命令均在**项目根目录**（含 `docker-compose.yml` 的目录）执行；客户端实际跑在容器内，无需本机安装 `psql`（镜像已自带）。

---

## 进入交互式 psql

```bash
docker compose exec database psql -U directus -d directus
```

退出交互：`\q` 或 `Ctrl+D`。

---

## 非交互：执行单条 SQL

`-T` 避免分配伪终端，适合脚本与管道。

```bash
docker compose exec -T database psql -U directus -d directus -c "SELECT version();"
```

多行 SQL 可用 heredoc：

```bash
docker compose exec -T database psql -U directus -d directus <<'SQL'
SELECT current_database(), current_user;
SQL
```

---

## psql 常用元命令（交互内使用）

| 命令 | 作用 |
|------|------|
| `\l` | 列出数据库 |
| `\c 数据库名` | 切换数据库 |
| `\dn` | 列出 schema |
| `\dt` | 当前 schema 下的表 |
| `\dt *.*` | 所有 schema 的表 |
| `\d 表名` | 表结构、索引、约束 |
| `\d+ 表名` | 更详细（含存储等） |
| `\df` | 函数 |
| `\dv` / `\ds` | 视图 / 序列 |
| `\timing on` | 显示每条 SQL 耗时 |
| `\x on` | 宽表竖排显示（便于阅读） |
| `\?` | 帮助（元命令） |
| `\h SELECT` | SQL 语法帮助 |

---

## 常用 SQL 片段

```sql
-- 当前连接与权限
SELECT current_database(), current_user, inet_server_addr(), inet_server_port();

-- 表大致行数（大表慎用，会扫全表；可配合 reltuples 估算另查文档）
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 20;
```

---

## 备份与恢复（逻辑）

**备份**（自定义格式，可并行恢复）：

```bash
docker compose exec -T database pg_dump -U directus -d directus -Fc -f /tmp/directus.dump
docker compose cp database:/tmp/directus.dump ./directus-backup.dump
```

**恢复到同一 Compose 库（会写入现有库，注意冲突）**：

```bash
docker compose cp ./directus-backup.dump database:/tmp/directus.dump
docker compose exec database pg_restore -U directus -d directus --clean --if-exists /tmp/directus.dump
```

生产或重要数据前务必先确认参数含义；`--clean` 会 drop 对象，仅在明确需要时使用。

纯 SQL 文本导出：

```bash
docker compose exec -T database pg_dump -U directus -d directus --no-owner > directus.sql
```

---

## 本机直连数据库（可选）

默认 Compose **未** 把 5432 映射到宿主机。若要在本机用 GUI 或本地 `psql`，可在 `database` 服务下增加：

```yaml
ports:
  - "5432:5432"
```

然后：

```bash
psql -h localhost -p 5432 -U directus -d directus
```

---

## macOS 图形客户端（开源、体验较好）

PostgreSQL **没有**官方自带的 Web 后台；在 Mac 上想用「丝滑」的图形工具连库，可在下列**开源**方案里选（均需先在 Compose 里映射 `5432:5432`，见上一节）。

| 工具 | 许可证 | 特点 |
|------|--------|------|
| [Beekeeper Studio](https://github.com/beekeeper-studio/beekeeper-studio) | GPLv3（另有商业许可的构建） | 界面现代、交互轻，适合日常查表、写 SQL |
| [DBeaver Community](https://github.com/dbeaver/dbeaver) | Apache-2.0 | 功能全（ER 图、导入导出、多引擎），基于 Java，启动略重 |
| [pgAdmin 4](https://github.com/pgadmin-org/pgadmin4) | PostgreSQL 生态标配 | 官方路线、能力完整，UI 相对传统 |

**安装（示例）**：Beekeeper / DBeaver 均可从各自 [GitHub Releases](https://github.com/beekeeper-studio/beekeeper-studio/releases) 或官网下载 macOS 安装包；也可用 Homebrew（若已有 cask）：`brew install --cask beekeeper-studio` / `brew install --cask dbeaver-community`。

**连接参数**（与 [Docker 环境搭建](../docker-setup.md) 中示例一致）：

| 字段 | 值 |
|------|-----|
| Host | `localhost` |
| Port | `5432`（需在 `database` 上配置 `ports` 映射） |
| User | `directus` |
| Password | `directus` |
| Database | `directus` |
| SSL | 本地开发多选「禁用」或 `prefer`（见下） |

**SSL 是干嘛用的**：指客户端到 PostgreSQL 之间是否走 **TLS 加密**。开启后，网络上的账号、SQL、结果集不易被窃听或篡改；云上托管库、跨公网/跨机房连接时通常 **必须** 开 SSL 并校验服务端证书。你当前场景是 **本机 `localhost` → 映射端口 → 容器里的库**，数据基本不离开本机，所以一般可以关 SSL，或选 `prefer`（服务器支持则用 TLS，否则退回明文，本地默认配置下常见为明文）。

---

## 与 Directus 相关的表

Directus 会创建大量 `directus_*` 系统表及你自建 Collection 对应的表。排查时可：

```sql
\dt public.*
-- 或
SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;
```

---

## 延伸阅读

- 官方 psql 文档：https://www.postgresql.org/docs/current/app-psql.html
- `pg_dump` / `pg_restore`：https://www.postgresql.org/docs/current/app-pgdump.html
