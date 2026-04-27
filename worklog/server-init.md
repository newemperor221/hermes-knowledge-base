# 服务器初始化标准流程

**最近更新**：2026-04-27

---

## 标准流程

### 1. SSH 连上后先改 SSH 端口
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo sed -i 's/^#Port 22/Port 63847/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```
> ⚠️ **重要**：先开一个新 session 确认能连上，再断开旧 session！

### 2. 安装基础工具
```bash
sudo apt-get update && sudo apt-get install -y fail2ban ufw curl wget
```

### 3. 配置 UFW
```bash
# 默认策略
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许 SSH（新端口）
sudo ufw allow 53621/tcp

# 允许已建立连接
sudo ufw allow in on eth0 from 192.168.0.0/16 to any port 1:65535 proto tcp state established
# 简化版：只放 SSH 和必要端口
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 启用
sudo ufw --force enable
sudo ufw status verbose
```

### 4. 配置 fail2ban
```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

### 5. 禁用密码登录（密钥已配置好后）
```bash
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### 6. 记录到 servers.yaml
```yaml
atlanta:
  ip: 23.95.218.144
  ssh_port: 53621
  user: woioeow
  sudo_password: 4561834  # 仅存加密后或提醒用户
```

---

## 踩坑记录

### 1. Atlanta 服务器被重置后 UFW/fail2ban 消失
**问题**：Atlanta 重置后所有手动安装的东西都没了
**解决**：登录后重新走一遍流程，目前没有自动化脚本（待做）

### 2. SSH 改端口时把自己关外面
**问题**：改了端口但没验证就断开，旧 session 没了，新 session 连不上
**解决**：保留旧 session 到新 session 验证成功后再关

---

## 验证

```bash
sudo ufw status verbose  # 应该 active
sudo systemctl status fail2ban  # 应该 running
sudo systemctl status sshd  # 应该 running
```
