# SSL + Let's Encrypt 踩坑

**最近更新**：2026-04-27

---

## 踩坑记录

### 1. Docker nginx 占用 80 端口，certbot --nginx 失败
**问题**：运行 `certbot --nginx` 报错端口被占用
**原因**：docker nginx 容器占着宿主机的 80 端口
**解决**：用 `--webroot` 方式
```bash
certbot certonly --webroot -w /var/lib/letsencrypt -d vps.357561.xyz
```
需要：
- docker-compose 添加 volume：`/var/lib/letsencrypt:/var/lib/letsencrypt`（可写，不能 `:ro`）
- 宿主机关闭占用 80 的进程（docker nginx 转发 80 到容器，certbot 需要宿主机能访问 80）

---

### 2. acme challenge volume 不能是只读
**问题**：certbot 报错 `Could not write to /var/lib/letsencrypt/.well-known/acme-challenge/`
**原因**：docker-compose volume 挂了 `:ro`（只读）
**解决**：去掉 `:ro`
```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro   # 证书输出（可读）
  - /var/lib/letsencrypt:/var/lib/letsencrypt  # acme challenge（必须可写）
```

---

### 3. 证书申请完后 nginx 配置 HTTPS
**问题**：证书申请下来了但 HTTPS 跑不通
**解决**：分两步
1. 先申请证书（停 docker nginx 或用 --webroot）
2. 再配置 nginx HTTPS 并重启
3. 最后开 Cloudflare 小黄云

nginx HTTPS 配置模板：
```nginx
server {
    listen 443 ssl;
    server_name vps.357561.xyz;

    ssl_certificate /etc/letsencrypt/live/vps.357561.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vps.357561.xyz/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
}

server {
    listen 80;
    server_name vps.357561.xyz;
    return 301 https://$host$request_uri;
}
```

---

### 4. Cloudflare 小黄云关闭时证书能申请，但开启后 CDN 变更需要重新验证
**问题**：小黄云关闭时证书申请成功，开启后要等 CDN 同步
**解决**：申请证书前关闭小黄云，证书生效后再开启

---

### 5. 自动续期
**问题**：证书 90 天过期，需要自动续期
**解决**：certbot 自带定时器
```bash
sudo systemctl enable --now certbot.timer
sudo systemctl status certbot.timer
```

---

## 完整流程（Docker + Cloudflare + --webroot）

1. 关闭 Cloudflare 小黄云（DNS only）
2. docker-compose 配置 acme volume（可写）
3. 申请证书：`certbot certonly --webroot -w /var/lib/letsencrypt -d vps.357561.xyz`
4. docker-compose 配置证书 volume（`:ro`）
5. 配置 nginx HTTPS，重启容器
6. 开启 Cloudflare 小黄云
7. 验证：`curl -sI https://vps.357561.xyz`

---

## 验证

```bash
# 证书存在
sudo ls /etc/letsencrypt/live/vps.357561.xyz/

# 证书信息
sudo certbot certificates

# HTTPS 可访问
curl -sI https://vps.357561.xyz

# 自动续期定时器
sudo systemctl status certbot.timer
```
