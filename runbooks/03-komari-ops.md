# Runbook: Komari 面板运维

> 主控节点：155.94.180.55:58193（LA-1）

---

## 面板访问

- URL：`https://mon.357561.xyz`（cloudflared 隧道）
- 后端管理：直接连主控服务器

```bash
# cloudflared 日志
journalctl -u cloudflared -f

# Komari 服务状态
systemctl status komari
```

---

## 常见操作

### 查看当前在线节点

```bash
# 登录主控 MySQL
mysql -u komari -p -D komari

SELECT id, name, ip, last_seen, status FROM servers;
```

### 添加新节点（Agent 端）

```bash
# 在节点上安装 agent 后，从面板"添加服务器"获取 token
# Agent 自动注册流程：
# 1. Agent 启动 → 连接主控 API → 发送机器信息
# 2. 主控返回 server_id
# 3. Agent 保存 server_id 本地
# 4. 每 60s 上报心跳
```

### 改主题背景图（PurCarte-Plus）

> ⚠️ **配置在浏览器 localStorage，不在服务端数据库**

```javascript
// 浏览器控制台执行
// 1. 打开 https://mon.357561.xyz
// 2. F12 → Application → Local Storage → mon.357561.xyz
// 3. 找 PurCarte 相关配置项
// 4. 改 background 字段

// 示例
localStorage.setItem('komari_theme', JSON.stringify({
  theme: 'PurCarte-Plus',
  background: 'https://你的图片URL',
  ...
}))
```

### 排查节点掉线

```bash
# 1. 看 agent 日志（节点上）
journalctl -u komari-agent -f

# 2. 查网络连通性（主控上）
nc -zv <节点IP> <端口>

# 3. 防火墙是否拦截（节点上）
ufw status
iptables -L -n | grep <主控IP>

# 4. 重启 agent（节点上）
systemctl restart komari-agent
```

### 重启 Komari 主控

```bash
# 主控上
systemctl restart komari

# 确认状态
systemctl status komari
journalctl -u komari --since "-1m"
```

---

## IP Sentinel 主控

- 所在服务器：155.94.180.55（与 Komari 同机）
- 用途：IP 封禁/解封管理

```bash
# 查看当前封禁列表
# （需要查具体命令，视版本而定）

# 解封一个 IP
# （视版本而定）
```

---

## 备份

```bash
# 备份数据库
mysqldump -u komari -p komari > /tmp/komari_backup_$(date +%Y%m%d).sql

# 备份配置
cp -r /etc/komari /tmp/komari_config_$(date +%Y%m%d)/
```

---

## 故障对照表

| 症状 | 可能原因 | 修复 |
|------|------|------|
| 面板打不开 | cloudflared 隧道断了 | `systemctl restart cloudflared` |
| 节点一直离线 | agent 没启动 | `systemctl start komari-agent` |
| 节点离线但能 ping 通 | 防火墙拦了主控 | 检查 ufw / iptables |
| 主题改不了 | localStorage 写错了 | 清缓存重写 |
| 无法添加节点 | token 过期 | 从面板重新生成 |
