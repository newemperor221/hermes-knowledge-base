# sing-box Reality 部署踩坑

**最近更新**：2026-04-27

---

## 踩坑记录

### 1. sing-box 1.9 字段名变更
**问题**：配置报错或无效
**原因**：1.9 版本字段名改了

| 旧字段（1.8） | 新字段（1.9） |
|-------------|-------------|
| `port` | `listen_port` |
| `server` | `server_name` |
| `outbounds` 是对象 | `outbounds` 是数组 |

**验证**：`sing-box check config.json` 无报错

---

### 2. api inbound 的 type 不存在
**问题**：配置了 api inbound 启动失败
**原因**：1.9 没有 `type: "api"`
**解决**：删掉 api inbound 整个块，或改成 `type: "direct"`（不需要 api 时直接删）

---

### 3. Reality 必须字段缺失
**问题**：REALITY 握手失败
**原因**：Reality 配置缺少必要字段

必须同时有：
```json
{
  "private_key": "xxx",           // 生成：sing-box generate reality-keypair
  "handshake": {
    "server": "www.cloudflare.com",
    "server_port": 443
  }
}
```
缺一不可。

---

### 4. 客户端 1.13.11 必须加 uTLS fingerprint
**问题**：`uTLS is required by reality client`
**原因**：Reality 需要 uTLS 模拟真实浏览器指纹
**解决**：客户端配置加
```json
"utls": {
  "enabled": true,
  "fingerprint": "chrome"
}
```
没有这行客户端会报错。

---

### 5. 容器类 VPS DNS 故障（connection reset）
**问题**：REALITY 握手时 `connection reset`，日志报 `org.freedesktop.resolve1 was not provided`
**原因**：systemd-resolved 在容器类 VPS 上不完整，DoH 方式无法工作
**解决**：改 sing-box DNS 配置
```json
"dns": {
  "servers": [{"address": "udp://1.1.1.1"}],
  "final": "cf"
},
"route": {
  "default_domain_resolver": {
    "server": "cf"
  }
}
```
用 `udp://` 而非 `https://`，final 改 `cf`（Cloudflare）

---

### 6. 客户端 public_key 和 short_id 要对上
**问题**：连接成功但流量不通
**原因**：客户端用的 pbk/short_id 和服务端不匹配
**解决**：服务端 `sing-box generate reality-keypair` 生成密钥对，公钥给客户端，short_id 用 UUID 前8位或自定义

---

## 服务端部署流程

1. 装包：`wget https://github.com/SagerNet/sing-box/releases/download/v1.9.4/sing-box-1.9.4-linux-amd64.tar.gz`
2. 生成密钥：`sing-box generate reality-keypair`
3. 写配置：`~/sing-box/config.json`（listen_port + Reality inbound + outbounds）
4. 建数据目录：`sudo mkdir /var/lib/sing-box && sudo chown woioeow:woioeow /var/lib/sing-box`
5. 开机启动：`sudo tee /etc/systemd/system/sing-box.service` → `systemctl enable --now sing-box`
6. 防火：`sudo ufw allow 44308/tcp`

## 验证

```bash
# 服务端
sudo systemctl status sing-box
sudo ss -tlnp | grep 44308
sing-box verify config.json

# 客户端（本地 /tmp/sing-box-1.13.11-linux-amd64/）
/tmp/sing-box-1.13.11-linux-amd64/sing-box run config.json
```
