# Docker Hub / OrbStack 网络与代理配置备忘

本文记录在本机（macOS + OrbStack + 本地代理）下 **无法直连 Docker Hub** 时的原因与可行解决办法，便于日后复现或排查。

## 现象

- `docker pull` 报错：`TLS handshake timeout`、`context deadline exceeded`、`EOF` 等。
- 直连 `registry-1.docker.io:443` 超时；经本机 HTTP 代理访问同一地址可返回 **401**（说明代理与注册表连通正常）。

## 原因概要

1. **网络环境**：到 Docker 官方注册表直连不稳定或被限制，需走本机代理（如 Clash 等）。
2. **OrbStack / Docker 引擎在 Linux VM 内运行**：把代理写成 **`http://127.0.0.1:端口`** 时，流量指向 **虚拟机自己的回环**，**无法到达 macOS 上监听的代理**，导致握手失败。
3. **部分 `registry-mirrors` 域名**（如个别第三方镜像站）若在你当前网络下不可达，会加重失败或拖慢拉取，需按实测增删。

## 解决办法（已验证）

### 1. OrbStack：显式指定走宿主机代理

使用 **`host.docker.internal`** 指向 Mac 上的代理服务（端口以本机为准，示例为 **7897**）：

```bash
orb config set network_proxy http://host.docker.internal:7897
```

说明：`auto` 在部分场景下仍可能未让引擎侧正确走代理；显式 URL 更稳。

### 2. Docker 引擎配置：`~/.orbstack/config/docker.json`

在 **`proxies`** 中为守护进程配置 HTTP/HTTPS 代理（同样使用 **`host.docker.internal`**，不要用 VM 内的 `127.0.0.1`）：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://docker.nju.edu.cn"
  ],
  "proxies": {
    "http-proxy": "http://host.docker.internal:7897",
    "https-proxy": "http://host.docker.internal:7897",
    "no-proxy": "localhost,127.0.0.1,::1"
  }
}
```

- **镜像源**：按实际可用性配置；若某 mirror 长期超时，可从列表中移除。
- 修改后需 **重启 OrbStack 服务** 使引擎重新加载，例如：

```bash
orbctl stop
orbctl start
```

### 3. Docker CLI（macOS）：`~/.docker/config.json`

在 **本机终端** 使用 Docker 客户端时，代理在 Mac 上应使用 **`127.0.0.1`**：

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://127.0.0.1:7897",
      "httpsProxy": "http://127.0.0.1:7897",
      "noProxy": "localhost,127.0.0.1,::1"
    }
  }
}
```

（需与 `currentContext`、`auths` 等现有字段合并，保持合法 JSON。）

### 4. 验证命令

```bash
# 经代理探测官方注册表（401 为正常未认证响应）
curl -sS -o /dev/null -w "%{http_code}\n" -x http://127.0.0.1:7897 https://registry-1.docker.io/v2/

docker pull hello-world
docker pull busybox:latest
```

## 注意事项

- **本机代理程序需运行**，且监听地址/端口与配置一致（示例 **7897**）。关闭代理后，`docker pull` 可能再次失败。
- 从容器内访问 Mac 服务时，可使用 **`host.docker.internal`**；引擎侧代理也必须指向能到达 Mac 上代理的地址，而不是 VM 的 `127.0.0.1`（除非代理确实跑在 VM 内）。
- 若不再需要固定代理，可将 OrbStack 改回跟随系统或关闭，例如：

```bash
orb config set network_proxy auto
# 或
orb config set network_proxy none
```

并相应删除或调整 `~/.orbstack/config/docker.json` 与 `~/.docker/config.json` 中的 `proxies`。

## 参考

- OrbStack 文档：[Container networking / Proxies](https://docs.orbstack.dev/docker/network)
- Docker 文档：[Daemon proxy configuration](https://docs.docker.com/engine/daemon/proxy/)

---

*记录日期：2026-04-04*
