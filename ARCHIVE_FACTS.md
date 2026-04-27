# ARCHIVE FACTS — 归档知识库
# 用途：memory 放不下的长期事实、踩坑记录、定型模板
# 检索方式：search_files / grep

---

# 知识库操作约定
# 涉及服务器操作时，默认先扫 ~/.hermes/inventory/servers.yaml
# 长期踩坑记录统一放 ARCHIVE_FACTS.md，memory 放不下的来这里

---

## 服务器 SSH 连接
- Buffalo: `ssh root@172.245.159.219 -p 27391`
- LA-1: `ssh root@155.94.180.55 -p 58193`（cloudflared+Komari+TG）
- LA-2: `ssh root@23.95.201.153 -p 47283`
- Santa Clara: `ssh root@45.39.12.227 -p 63841`
- Atlanta: `ssh woioeow@23.95.218.144 -p 53621`（sudo: 4561834）

---

## sing-box Reality DNS 故障排查（容器类 VPS）
**症状**：REALITY 握手时 connection reset，sing-box 日志报 `context deadline exceeded` 或 `link has no DNS servers configured`

**根因**：systemd-resolved 在容器类 VPS 上不完整，DoH ("type": "https") 报 `org.freedesktop.resolve1 was not provided`

**修复步骤**：
1. 装 systemd-resolved：`apt install systemd-resolved`
2. 写 `/etc/systemd/resolved.conf`：`DNS=1.1.1.1 8.8.8.8`
3. 启用：`systemctl enable --now systemd-resolved`
4. 绑定网卡：`resolvectl dns eth0 1.1.1.1 8.8.8.8`（可能超时，重试）
5. 重启 sing-box：`systemctl restart sing-box`
6. 验证：`journalctl -u sing-box --since "-10s" | grep -i error`

**配置修复**：若上述不行，改 sing-box 配置：
- `dns.servers` 改 udp
- `dns.final` 改为 cf
- `route.default_domain_resolver.server` 改为 cf

---

## Komari 面板主题背景图
- PurCarte-Plus 主题配置存在**浏览器 localStorage**，不在服务端 SQLite
- 改背景图：浏览器开发者工具 → Application → Local Storage → 找 PurCarte 配置项 → 改 background 字段
- 不能通过服务端数据库修改

---

## 新服务器初始化标准流程
1. SSH 连上，改 SSH 端口（`Port 63847`），禁用密码登录
2. `apt update && apt install -y fail2ban ufw`
3. 配置 UFW：`ufw allow 22/tcp`（原端口）→ 改 sshd port → `ufw allow 63847/tcp` → `ufw enable`
4. 配置 fail2ban：`systemctl enable --now fail2ban`
5. 记录到 inventory/servers.yaml

---

## Atlanta (23.95.218.144) 重置风险

RackNerd VPS 系统模板会周期性重置，清空：UFW、fail2ban、sudoers.d NOPASSWD、sing-box。Docker 数据和 /home 用户目录不受影响。重置后需重装 UFW + fail2ban，重启 sing-box。

## InkOS 番茄小说流水线
- 状态：待执行
- 架构：Hermes Agent 外层规划 → InkOS 内部多Agent Pipeline（写/审/改）
- 模型：DeepSeek V3（已配置，比MiniMax便宜+中文更强）
- bin路径：`~/.hermes/node/bin/`
