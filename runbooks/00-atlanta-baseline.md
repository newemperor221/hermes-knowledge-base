# Atlanta 基线文档

**最后更新**：2026-04-27

## 服务器信息

| 项目 | 值 |
|------|-----|
| IP | 23.95.218.144 |
| SSH 端口 | 53621 |
| 用户 | woioeow |
| sudo 密码 | 4561834 |
| 系统 | Debian 12 (racknerd-f97303b) |
| 核心 | 6.1.0-9-amd64 x86_64 |
| CPU | 2vCPU |
| 内存 | 2.4GB |
| 磁盘 | 43GB (/dev/vda1) |

## 网络监听

| 端口 | 服务 | 备注 |
|------|------|------|
| 80 | docker-proxy → nginx | HTTP |
| 443 | docker-proxy → nginx | HTTPS |
| 44308 | sing-box | VLESS Reality |
| 53621 | sshd | SSH |

## 已部署服务

### nginx:alpine
- 方式：Docker Compose (`~/docker/nginx/`)
- 端口：80, 443
- SSL：Let's Encrypt（vps.357561.xyz）
- 状态：✅ 运行中

### sing-box 1.9.4
- 路径：`/usr/local/bin/sing-box`
- 配置：`/home/woioeow/sing-box/config.json`
- 数据目录：`/var/lib/sing-box`
- systemd service：`/etc/systemd/system/sing-box.service`
- VLESS Reality 端口：44308
- SOCKS5 出港：127.0.0.1:1080
- 状态：✅ 运行中

## 安全配置

### UFW
- 状态：✅ active
- 规则：仅允许 53621(SSH), 80(HTTP), 443(HTTPS), 44308(VLESS)

### fail2ban
- 状态：✅ running
- 开机启动：enabled

### SSH 加固
- 端口：53621（非22）
- Root 登录：禁止
- 密码登录：禁止
- 密钥登录：启用（ed25519）

## SSL 证书

| 域名 | 路径 | 到期 |
|------|------|------|
| vps.357561.xyz | /etc/letsencrypt/live/vps.357561.xyz/ | 2026-07-26 |

- 申请方式：certbot --webroot
- 自动续期：certbot.timer enabled

## VLESS Reality 节点

| 项目 | 值 |
|------|-----|
| 地址 | 23.95.218.144:44308 |
| UUID | 04814792-272a-42f2-8715-4225a8e68934 |
| PublicKey | 7j6iS2pU56dCil6Zc_Y-rKc6DOnWh_FfnfuvOAilsmQ |
| ShortID | 04814792 |
| SNI | www.cloudflare.com |
| 协议 | VLESS+Reality+xtls-rprx-vision |

**链接**：
```
vless://04814792-272a-42f2-8715-4225a8e68934@23.95.218.144:44308?type=tcp&encryption=none&security=reality&flow=xtls-rprx-vision&sni=www.cloudflare.com&fp=chrome&pbk=7j6iS2pU56dCil6Zc_Y-rKc6DOnWh_FfnfuvOAilsmQ&sid=04814792#亚特兰大
```

## 待完成

- [ ] Cloudflare Tunnel（token 待用户提供）
- [ ] Prometheus + Grafana 监控
- [ ] GitHub Actions CI/CD 流水线
- [ ] iptables 深度规则（NAT/端口转发）
