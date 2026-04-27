# Atlanta iptables 快照 (2026-04-27)

## 摸底结论

| 来源 | 职责 |
|------|------|
| **UFW** | INPUT/FORWARD 基础框架（ufw-before-* → ufw-user-* → ufw-after-*），用户自定义规则写 `ufw-user-*` 链 |
| **Docker** | `nat/PREROUTING` 的 DOCKER 链做 DNAT；`filter/FORWARD` 的 DOCKER-USER / DOCKER-FORWARD / DOCKER-CT 链处理容器流量 |
| **sing-box** | 纯监听 44308/TCP，未改动 iptables |

## 关键发现

- `ip_forward = 1`（内核转发已开启）
- FORWARD 链 `policy DROP`，但 DOCKER-USER 链优先匹配
- Docker 在 nat/OUTPUT 排除了 127.0.0.0/8（防止本地访问绕 NAT）
- DOCKER-USER 链目前**为空**——可用于手动端口转发规则

## filter 表（简化）

```
INPUT (policy DROP) → ufw-before-input → ... → ufw-user-input → ufw-after-input
FORWARD (policy DROP) → DOCKER-USER → DOCKER-FORWARD → ufw-before-forward → ...
OUTPUT (policy ACCEPT) → ufw-before-output → ...
```

## nat 表

```
PREROUTING → DOCKER (所有暴露端口的 DNAT)
  - tcp:80  → 172.18.0.2:80   (nginx)
  - tcp:443 → 172.18.0.2:443  (nginx)
  - tcp:3000 → 172.19.0.2:3000 (grafana)
  - tcp:9090 → 172.19.0.3:9090 (prometheus)

OUTPUT (排除了 127.0.0.0/8) → DOCKER

POSTROUTING
  - MASQUERADE 172.19.0.0/16 → !br-fb61fd4cc11a
  - MASQUERADE 172.18.0.0/16 → !br-885b2e0d4e9c
  - MASQUERADE 172.17.0.0/16 → !docker0
```

## 容器 IP

| 容器 | IP | 暴露端口 |
|------|-----|---------|
| nginx | 172.18.0.2 | 80, 443 |
| grafana | 172.19.0.2 | 3000 |
| prometheus | 172.19.0.3 | 9090 |

## 端口转发实战记录

**实验日期**：2026-04-27

**目标**：外部访问 `23.95.218.144:22080` → nginx 容器 `172.18.0.2:80`

**步骤**：
```bash
# 1. 先备份
sudo iptables-save > /tmp/iptables-backup-YYYYMMDD_HHMMSS.v4

# 2. 加 DNAT（PREROUTING 处理外部访问）
sudo iptables -t nat -I PREROUTING -p tcp --dport 22080 -j DNAT --to-destination 172.18.0.2:80

# 3. 加 OUTPUT DNAT（处理本机访问 127.0.0.1:22080）
sudo iptables -t nat -A OUTPUT -p tcp --dport 22080 -j DNAT --to-destination 172.18.0.2:80

# 4. 加 FORWARD 规则（DOCKER-USER 链）
sudo iptables -I DOCKER-USER -p tcp -d 172.18.0.2 --dport 80 -j ACCEPT

# 5. 验证（从外部）
curl http://23.95.218.144:22080/ -H 'Host: vps.357561.xyz'

# 6. 清理
sudo iptables -t nat -D OUTPUT -p tcp --dport 22080 -j DNAT --to-destination 172.18.0.2:80
sudo iptables -t nat -D PREROUTING -p tcp --dport 22080 -j DNAT --to-destination 172.18.0.2:80
sudo iptables -D DOCKER-USER -p tcp -d 172.18.0.2 --dport 80 -j ACCEPT
```

**踩坑记录**：
1. **iptables DNAT 不创建监听端口** —— DNAT 只改报文目的地址，没有服务绑定端口则 RST。真正端口转发用 socat：`socat TCP-LISTEN:22081,fork,reuseaddr TCP:172.18.0.2:80`
2. **OUTPUT DNAT 排除 127.0.0.0/8** —— Docker 规则防止本地访问绕 NAT，本机测试必须从外部发起
3. **外部访问走 PREROUTING，本机访问走 OUTPUT**
4. **DOCKER-USER 在 DOCKER-FORWARD 之前** —— 手动 FORWARD 规则写 DOCKER-USER 才生效

**结论**：实验成功，规则清理干净未留残留。
