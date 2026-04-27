# Runbook: sing-box Reality DNS 故障排查

> 适用场景：REALITY 握手时报 `connection reset`、`context deadline exceeded`、日志里 `link has no DNS servers configured`

---

## 快速诊断

```bash
# 1. 看最近错误日志
journalctl -u sing-box --since "-30s" | grep -iE "error|reset|timeout"

# 2. 查 DNS 配置
cat /etc/systemd/resolved.conf
resolvectl status

# 3. 验证 DNS 解析
dig @1.1.1.1 cloudflare.com +short
curl -v --dns-servers 1.1.1.1 https://www.cloudflare.com/ 2>&1 | head -20
```

---

## 症状 → 根因 → 修复

### 症状 A：握手时 `connection reset`

**根因**：systemd-resolved 在容器类 VPS 上不完整，DoH 报 `org.freedesktop.resolve1 was not provided`

**修复**：

```bash
# 方案 1：装 systemd-resolved
apt install -y systemd-resolved
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
DNSOverTLS=no
EOF
systemctl enable --now systemd-resolved
resolvectl dns eth0 1.1.1.1 8.8.8.8

# 方案 2：改 sing-box 配置（更简单）
# 编辑 /etc/sing-box/config.json，把 dns.servers 从 https://... 改成：
"dns": {
  "servers": [
    {
      "tag": "cf",
      "address": "https://1.1.1.1/dns-query",
      "detour": "direct"
    },
    {
      "tag": "local",
      "address": "udp://1.1.1.1",
      "detour": "direct"
    }
  ],
  "final": "cf"
}
```

### 症状 B：`context deadline exceeded`

**根因**：DNS 解析超时，sing-box 等不到响应

**修复**：

```bash
# 看 sing-box 配置里的 DNS servers
cat /etc/sing-box/config.json | jq '.dns'

# 把 DNS 改成国内可达的
# 不要用 8.8.8.8，用 1.1.1.1 或 223.5.5.5
```

### 症状 C：`link has no DNS servers configured`

**根因**：`dns.servers` 数组为空或配置错误

**修复**：

```bash
# 检查配置
cat /etc/sing-box/config.json | jq '.dns.servers'

# 至少要有一个 server
# 参考配置：
"dns": {
  "servers": [
    {
      "tag": "cf",
      "address": "https://cloudflare-dns.com/dns-query",
      "detour": "direct"
    },
    {
      "tag": "local",
      "address": "udp://1.1.1.1",
      "detour": "direct"
    }
  ],
  "final": "cf"
}
```

---

## 完整修复流程

```bash
# Step 1：确认系统 DNS 状态
resolvectl status 2>/dev/null || echo "systemd-resolved not running"

# Step 2：测试 DNS 解析
dig A cloudflare.com @1.1.1.1 +short

# Step 3：测试 curl 到 CF
curl -I https://www.cloudflare.com/ --dns-servers 1.1.1.1

# Step 4：重启 sing-box
systemctl restart sing-box

# Step 5：看日志
journalctl -u sing-box -f
```

---

## 配置模板（参考）

```json
{
  "dns": {
    "servers": [
      {
        "tag": "cf",
        "address": "https://1.1.1.1/dns-query",
        "detour": "direct"
      },
      {
        "tag": "local",
        "address": "udp://223.5.5.5",
        "detour": "direct"
      }
    ],
    "final": "cf",
    "strategy": "prefer_ipv4",
    "disable_cache": false,
    "verseip_detection": true
  },
  "route": {
    "default_domain_resolver": {
      "server": "cf"
    }
  }
}
```

---

## 已知受影响节点

| 节点 | 症状 | 状态 |
|------|------|------|
| Atlanta (23.95.218.144) | DoH 失败 | ✅ 修复：改用 udp://1.1.1.1 |
| LA-1 (155.94.180.55) | 正常 | - |
| 其他容器类 VPS | 可能中招 | 按上述流程修复 |
