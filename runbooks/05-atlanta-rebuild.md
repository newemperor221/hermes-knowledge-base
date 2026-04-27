# Atlanta 服务器重建手册

**适用场景**：Atlanta（Racknerd VPS）被重置后，从零恢复到可用状态。

**最近更新**：2026-04-27

---

## 快速检查

登录后先确认当前状态：
```bash
sudo ufw status
systemctl status fail2ban
systemctl status sing-box
docker ps
curl -sI https://vps.357561.xyz
```

---

## 1. UFW + fail2ban

```bash
sudo apt-get update -qq && sudo apt-get install -y -qq ufw fail2ban

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 53621/tcp   # SSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable

sudo systemctl enable --now fail2ban
```

---

## 2. sing-box 服务

### 检查是否已安装
```bash
test -f /usr/local/bin/sing-box && /usr/local/bin/sing-box version
```

### 如需安装
```bash
cd /tmp
wget https://github.com/SagerNet/sing-box/releases/download/v1.9.4/sing-box-1.9.4-linux-amd64.tar.gz
tar -xf sing-box-1.9.4-linux-amd64.tar.gz
sudo mv sing-box-1.9.4-linux-amd64/sing-box /usr/local/bin/sing-box
```

### 重建 systemd service（如丢失）
```bash
sudo tee /etc/systemd/system/sing-box.service > /dev/null << 'EOF'
[Unit]
Description=sing-box service
After=network.target

[Service]
Type=simple
User=woioeow
WorkingDirectory=/home/woioeow/sing-box
ExecStart=/usr/local/bin/sing-box run -c /home/woioeow/sing-box/config.json
Restart=on-failure
RestartSec=5
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

### 启动并修复端口占用
```bash
# 检查端口占用
sudo ss -tlnp | grep 44308

# 如有旧进程占用，杀掉再启动
sudo kill -9 $(sudo lsof -t -i:44308)
sleep 1

sudo systemctl daemon-reload
sudo systemctl enable --now sing-box
```

### 验证
```bash
sudo systemctl status sing-box --no-pager | head -6
sudo ss -tlnp | grep 44308
```

---

## 3. nginx（Docker 容器）

```bash
cd ~/docker/nginx
docker compose up -d --force-recreate nginx
docker ps --format "table {{.Names}}\t{{.Status}}"
```

验证：
```bash
curl -sI http://23.95.218.144
# 应返回 301 redirect to HTTPS

curl -sI https://vps.357561.xyz
# 应返回 200 OK
```

---

## 4. SSL 证书

检查是否存在：
```bash
sudo certbot certificates 2>/dev/null | grep vps.357561.xyz
```

如证书过期或不存在：
1. 关闭 Cloudflare 小黄云（DNS only）
2. 确认 80 端口可访问（docker nginx 在）
3. 申请：`sudo certbot certonly --webroot -w /var/lib/letsencrypt -d vps.357561.xyz`
4. 重建 nginx 容器加载证书
5. 开启 Cloudflare 小黄云

---

## 5. cloudflared Tunnel

**已确认用于 Dedirock（mon.357561.xyz），Atlanta 留空即可。**

如需启用：
```bash
# docker-compose 已配置，需填 token
# 环境变量：CLOUDFLARED_TOKEN=your-token
```

---

## 完整验证清单

- [ ] UFW `status: active`
- [ ] fail2ban `active (running)`
- [ ] sing-box `active (running)` + 44308 监听
- [ ] nginx 容器 `Up`
- [ ] vps.357561.xyz HTTPS 200
- [ ] VLESS Reality 节点连通（本地测）

---

## 已知问题

### 服务器重置后会丢失
UFW、fail2ban、sing-box systemd service 文件都会消失，docker nginx 容器可能健在。
原因：Racknerd 重装系统会清空所有手动安装的包和 systemd service。
