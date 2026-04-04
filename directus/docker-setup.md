# 本地 Docker 环境搭建

> 目标：使用 Docker Compose 启动 PostgreSQL + Directus，建立本地可用的开发环境。
> 对应学习计划：阶段一前置 / 阶段四自托管部署预演

---

## 任务清单

- [X] 安装并确认 Docker Desktop 运行正常
- [ ] 编写 `docker-compose.yml`，包含 PostgreSQL + Directus 两个服务
- [ ] 配置 Directus 环境变量，连接 PostgreSQL
- [ ] 启动容器，验证 Directus 成功连接数据库
- [ ] 验证数据持久化：重启容器后数据不丢失

---

## docker-compose.yml 参考

```yaml
version: "3"

services:

  database:
    image: postgres:15
    # 如需地理空间扩展可改用：postgis/postgis:15-3.3
    volumes:
      - ./data/database:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: directus
      POSTGRES_PASSWORD: directus
      POSTGRES_DB: directus
    healthcheck:
      test: ["CMD", "pg_isready", "--host=localhost", "--username=directus"]
      interval: 10s
      timeout: 5s
      retries: 5

  directus:
    image: directus/directus:11   # 生产环境请锁定具体版本号
    ports:
      - "8055:8055"
    volumes:
      - ./uploads:/directus/uploads
      - ./extensions:/directus/extensions
    depends_on:
      database:
        condition: service_healthy
    environment:
      SECRET: "替换为随机字符串"        # openssl rand -hex 32
      PUBLIC_URL: "http://localhost:8055"

      DB_CLIENT: "pg"
      DB_HOST: "database"
      DB_PORT: "5432"
      DB_DATABASE: "directus"
      DB_USER: "directus"
      DB_PASSWORD: "directus"

      ADMIN_EMAIL: "admin@example.com"
      ADMIN_PASSWORD: "替换为强密码"

volumes:
  data:
```

---

## 启动命令

```bash
# 首次启动（后台运行）
docker compose up -d

# 查看日志确认连接成功
docker compose logs directus -f

# 停止
docker compose down

# 停止并删除数据（慎用）
docker compose down -v
```

---

## 验证步骤

1. 浏览器打开 http://localhost:8055
2. 用 `ADMIN_EMAIL` / `ADMIN_PASSWORD` 登录
3. 进入 Settings → Data Model，新建一个 Collection，确认操作无报错
4. 执行 `docker compose restart directus`，重启后数据仍在 → 持久化成功

---

## 常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| Directus 启动报 DB 连接失败 | PostgreSQL 未就绪 | 检查 `healthcheck`，等待 database healthy 后再启动 |
| 重启后数据丢失 | 未挂载 volume | 确认 `./data/database` 已挂载 |
| 端口 8055 被占用 | 本地已有服务 | 改为 `"8056:8055"` |
| SECRET 警告 | 使用了默认值 | 替换为随机字符串，可用 `openssl rand -hex 32` 生成 |

---

## 相关链接

- 官方部署文档：https://directus.io/docs/self-hosting/deploying
- 环境变量参考：https://directus.io/docs/self-hosting/environment-variables

---

## 附加任务：接入 Redis 缓存

> 完成主任务后进行，了解 Directus 缓存机制对性能的影响。

### 任务清单

- [ ] 在 `docker-compose.yml` 中加入 Redis 服务
- [ ] 配置 Directus 连接 Redis，开启缓存
- [ ] 验证缓存生效：观察重复请求响应速度变化
- [ ] 测试 `CACHE_AUTO_PURGE`：修改数据后确认缓存自动失效
- [ ] **边界测试**：停止 Redis 容器，观察 Directus 的降级行为

### 在现有 docker-compose.yml 中追加

在 `services` 下新增 `cache` 服务：

```yaml
  cache:
    image: redis:6
    healthcheck:
      test: ["CMD-SHELL", "[ $$(redis-cli ping) = 'PONG' ]"]
      interval: 10s
      timeout: 5s
      retries: 5
```

修改 `directus` 服务，加入 depends_on 和环境变量：

```yaml
  directus:
    depends_on:
      database:
        condition: service_healthy
      cache:                          # 新增
        condition: service_healthy
    environment:
      # ... 原有变量保持不变，追加以下内容：
      CACHE_ENABLED: "true"
      CACHE_AUTO_PURGE: "true"
      CACHE_STORE: "redis"
      REDIS: "redis://cache:6379"
```

### Redis 在 Directus 中的作用

| 功能 | 说明 |
|------|------|
| API 响应缓存 | 缓存 GET 请求结果，减少数据库查询 |
| 缓存自动失效 | 写操作后自动清除相关缓存（`CACHE_AUTO_PURGE`） |
| Rate Limiting | 可用 Redis 存储限流计数器 |
| Session 存储 | 多实例部署时共享 Session 状态（生产必备）|

### 验证命令

```bash
# 进入 Redis 容器查看缓存 key
docker compose exec cache redis-cli keys "*"

# 监控实时命令
docker compose exec cache redis-cli monitor
```
