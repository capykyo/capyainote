# 从 MacBook 通过局域网访问 Windows WSL2 中的 Docker

## 场景说明

| 项目 | 说明 |
|------|------|
| 服务端 | Windows 11，WSL2 中运行 Docker |
| 客户端 | 同一路由器下的 MacBook |
| 目标 | 在 Mac 上用 `docker` 命令操控 Windows WSL2 里的 Docker |

---

## 核心难点

WSL2 运行在 Hyper-V 虚拟机内，拥有独立 IP（如 `172.x.x.x`），与 Windows 宿主机不在同一网段。
外部设备无法直接访问 WSL2 的 IP，需要在 Windows 上做**端口转发**将流量转进去。

---

## 方案：SSH 转发 + Docker Context（推荐）

不暴露 Docker TCP 端口（安全），通过 SSH 隧道让 Mac 的 docker CLI 远程操控 WSL2 Docker。

### 第一步：WSL2 内安装并启动 SSH 服务

```bash
# 在 WSL2 终端内执行
sudo apt update && sudo apt install -y openssh-server

# 编辑 sshd 配置，允许密码或密钥登录
sudo nano /etc/ssh/sshd_config
# 确认或修改以下行：
#   Port 2222          ← 避免与 Windows 自带 SSH 冲突，改用非标准端口
#   PasswordAuthentication yes
#   ListenAddress 0.0.0.0

# 启动 SSH 服务
sudo service ssh start

# 查看 WSL2 的内部 IP
ip addr show eth0 | grep 'inet '
# 示例输出：inet 172.26.48.100/20
```

> **每次重启 WSL2 都需要重新执行 `sudo service ssh start`**（WSL2 没有 systemd 默认自启，见下方"自动启动"章节）。

---

### 第二步：Windows 上做端口转发

在 **Windows 的 PowerShell（管理员）** 中执行：

```powershell
# 将 Windows 的 2222 端口转发到 WSL2 内部 IP 的 2222 端口
# 将下面的 172.26.48.100 替换为第一步查到的实际 WSL2 IP
netsh interface portproxy add v4tov4 `
    listenport=2222 listenaddress=0.0.0.0 `
    connectport=2222 connectaddress=172.26.48.100

# 查看已有的转发规则
netsh interface portproxy show all

# 如需删除规则（重置时用）
# netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0
```

> **注意**：WSL2 每次重启后内部 IP 可能变化，届时需要删除旧规则、重新添加。

---

### 第三步：Windows 防火墙放行端口

```powershell
# 管理员 PowerShell
New-NetFirewallRule -DisplayName "WSL2 SSH" `
    -Direction Inbound -Protocol TCP -LocalPort 2222 -Action Allow
```

---

### 第四步：Mac 上测试 SSH 连接

```bash
# 将 192.168.x.x 替换为 Windows 机器的局域网 IP（路由器分配的 IP）
# 查看 Windows IP：Win+R → cmd → ipconfig，找 Ethernet 或 Wi-Fi 的 IPv4 地址
ssh -p 2222 your_wsl_username@192.168.x.x
```

登录成功后说明链路通畅。

---

### 第五步：Mac 上配置 Docker Context

```bash
# 在 Mac 终端执行
docker context create wsl2-win \
    --docker "host=ssh://your_wsl_username@192.168.x.x:2222"

# 切换到该 context
docker context use wsl2-win

# 验证
docker ps
docker info
```

此后在 Mac 上执行的所有 `docker` 命令均作用于 Windows WSL2 中的 Docker。

```bash
# 切回本地 Mac Docker
docker context use default
```

---

## WSL2 SSH 自动启动（可选）

WSL2 默认无 systemd，SSH 需手动启动。可在 Windows 启动项中加一条命令自动拉起：

**方法：Windows 任务计划程序**

1. 打开"任务计划程序" → 创建基本任务
2. 触发器：用户登录时
3. 操作：启动程序
   - 程序：`C:\Windows\System32\wsl.exe`
   - 参数：`-u root service ssh start`

或在 PowerShell 中一键创建：

```powershell
$action = New-ScheduledTaskAction -Execute "wsl.exe" -Argument "-u root service ssh start"
$trigger = New-ScheduledTaskTrigger -AtLogOn
Register-ScheduledTask -TaskName "Start WSL2 SSH" -Action $action -Trigger $trigger -RunLevel Highest
```

---

## Windows IP 固定（可选）

家庭路由器通常通过 DHCP 分配 IP，Windows 的局域网 IP 可能变化。
建议在路由器后台为 Windows 网卡的 MAC 地址绑定固定 IP（DHCP 静态绑定），避免每次改 Mac 的 SSH 配置。

---

## WSL2 内部 IP 变化问题

WSL2 重启后内部 IP（`172.x.x.x`）会变化，需要更新 `netsh portproxy` 规则。

**自动更新脚本**（保存为 `update-wsl-proxy.ps1`，管理员运行）：

```powershell
$wslIp = (wsl hostname -I).Trim().Split()[0]
netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0
netsh interface portproxy add v4tov4 `
    listenport=2222 listenaddress=0.0.0.0 `
    connectport=2222 connectaddress=$wslIp
Write-Host "WSL2 IP: $wslIp → portproxy updated"
```

将此脚本加入任务计划程序（用户登录时运行），与 SSH 自启脚本配合使用。

---

## 访问容器内的 Web 服务（如 Directus）

控制 Docker 只是第一步。启动容器后，还需要把容器端口打通到 Mac 浏览器。
端口传递链为：**容器 → WSL2 → Windows → Mac 浏览器**，共需两段转发。

### 第一段：Docker 把容器端口暴露到 WSL2

启动容器时用 `-p` 绑定到 WSL2 的所有网卡（`0.0.0.0`）：

```bash
# 以 Directus 为例，默认端口 8055
docker run -d -p 8055:8055 directus/directus

# 或 docker-compose.yml 中：
# ports:
#   - "8055:8055"
```

验证 WSL2 内部可访问：

```bash
curl http://localhost:8055
```

### 第二段：Windows 把该端口转发出来

在 **Windows PowerShell（管理员）** 中添加 portproxy 规则（与 SSH 的 2222 规则同理）：

```powershell
$wslIp = (wsl hostname -I).Trim().Split()[0]

# 转发 Directus 端口
netsh interface portproxy add v4tov4 `
    listenport=8055 listenaddress=0.0.0.0 `
    connectport=8055 connectaddress=$wslIp

# 防火墙放行
New-NetFirewallRule -DisplayName "WSL2 Directus" `
    -Direction Inbound -Protocol TCP -LocalPort 8055 -Action Allow
```

之后 Mac 浏览器直接访问：

```
http://192.168.x.x:8055
```

> 每增加一个新服务（如 pgAdmin:5050、MinIO:9001）都要重复上面两条命令，端口号替换即可。

---

### 替代方案：SSH 本地端口转发（临时调试用）

不改 Windows 防火墙，仅在 SSH 连接时用 `-L` 参数将远端端口映射到 Mac 本机：

```bash
# Mac 终端：SSH 连接的同时转发 8055
ssh -p 2222 -L 8055:localhost:8055 your_wsl_username@192.168.x.x

# 或同时转发多个端口
ssh -p 2222 \
    -L 8055:localhost:8055 \
    -L 9000:localhost:9000 \
    your_wsl_username@192.168.x.x
```

SSH 会话保持期间，Mac 浏览器访问 `http://localhost:8055` 即可。
关闭终端窗口后转发失效，适合临时排查，不适合长期使用。

---

### 将服务端口加入自动更新脚本

把上文 `update-wsl-proxy.ps1` 扩展，登录时一并刷新所有服务端口：

```powershell
$wslIp = (wsl hostname -I).Trim().Split()[0]

# 需要转发的端口列表
$ports = @(2222, 8055)   # SSH, Directus；按需增删

foreach ($port in $ports) {
    netsh interface portproxy delete v4tov4 listenport=$port listenaddress=0.0.0.0
    netsh interface portproxy add v4tov4 `
        listenport=$port listenaddress=0.0.0.0 `
        connectport=$port connectaddress=$wslIp
}

Write-Host "WSL2 IP: $wslIp → portproxy updated for ports: $($ports -join ', ')"
```

---

## 总结流程

```
Mac 浏览器 http://192.168.x.x:8055
  └─ Windows 防火墙 + netsh portproxy :8055
       └─ WSL2 IP:8055
            └─ Docker 容器 :8055 (Directus 等)

Mac docker CLI
  └─ docker context (SSH)
       └─ ssh :2222 → Windows → WSL2 sshd → Docker socket
```

| 操作 | 走哪条路 |
|------|---------|
| `docker ps` / `docker run` 等 CLI 命令 | SSH 隧道（docker context） |
| 浏览器访问容器 Web 页面 | netsh portproxy 直连 |
