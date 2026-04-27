# Runbook: Atlanta 测试机基线

> 23.95.218.144:53621，woioeow 用户，2C2.5G45G，Debian 12
> 用途：练手/测试环境，2026-04-26 交接

---

## 初始基线（交接时）

```
OS: Debian GNU/Linux 12 (bookworm)
Kernel: 6.1.0-9-amd64
CPU: 2C / 内存: 2.4GiB (303Mi used) / 磁盘: 43G (5% used)
Swap: 1.2Gi (未使用)
SSH: 端口 53621（woioeow 用户）
UFW: 未安装（交接时）
fail2ban: 已在运行

交接前安装的 Agent（已清除）：
- Komari Agent → 已 stop + disable
- IP Sentinel Agent → 已 stop + disable
```

## 当前基线（2026-04-26 最新）

```
UFW: active
放行：53621/tcp (SSH)
fail2ban sshd: 0 failed, 0 banned
Docker: 29.4.1 installed ✅
woioeow 已加入 docker 组 ✅
sudo NOPASSWD: 已配置 ✅
```

## 待做（TODO）

- [ ] 确认是否有 root 远程登录
- [ ] 查 29818 端口来历（历史记录中未见监听）

---

## Docker 环境

```
路径：~/docker/nginx/
文件：
  docker-compose.yml  (nginx:alpine + cloudflared)
  nginx.conf
  html/index.html
  .env (CLOUDFLARED_TOKEN=YOUR_TUNNEL_TOKEN_HERE)

当前运行：
  nginx-nginx-1 ✅ (80/443 端口映射)
  cloudflared    ❌ (未配置 token)

健康检查：/health → ok
```

---

## 操作日志

### 2026-04-26: Docker + nginx 部署

```
nginx:alpine 拉取+启动成功 ✅
容器状态：health: starting
端口映射：0.0.0.0:80->80, 0.0.0.0:443->443
健康检查端点：/health → "ok"
静态页面：index.html 部署成功

cloudflared: docker-compose 已配置，token 未填，暂未启动
```

### 2026-04-26: UFW 防火墙配置

```
UFW 已在运行
放行端口：
  53621/tcp (SSH) ✅

已删除规则：
  22/tcp (SSH 标准端口已废弃) ✅
  29814/tcp (来历不明，无服务监听) ✅

fail2ban status sshd:
  Currently failed: 0
  Total failed: 0
  Currently banned: 0
```

### 2026-04-26: sudo 免密配置

```
woioeow ALL=(ALL) NOPASSWD: ALL
权限：440（只读）
文件：/etc/sudoers.d/woioeow
验证：sudo -n ufw status → ok ✅
```

### 2026-04-26: Docker 安装

```
版本：Docker Engine 29.4.1
组件：docker-ce, containerd.io, docker-compose-plugin, buildx, buildx-plugin
hello-world 测试：✅ 拉取+运行成功
woioeow 已加入 docker 组 ✅
```

---

## 踩坑记录

**坑1**: woioeow 用户 PATH 不含 `/usr/sbin`，ufw 命令直接执行报 not found
→ 解决：用 `sudo ufw` 而不是直接 `ufw`

**坑2**: 之前误以为 29814 在监听，实际是 UFW 规则残留，无服务绑定
→ 解决：删除该规则，无影响

**坑3**: woioeow 用户 sudo 需要密码，但 `-S` stdin 方式需要 tty
→ 解决：配置 sudo NOPASSWD，以后直接 `sudo` 不用输密码
