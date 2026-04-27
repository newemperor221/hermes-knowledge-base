# Runbook: 新服务器初始化

> 每次拿到新 VPS，从头跑一遍，确保基线安全。

---

## 检查清单

- [ ] SSH 连接成功
- [ ] SSH 端口改为非标准端口
- [ ] 禁用密码登录（仅密钥）
- [ ] fail2ban 已安装并启用
- [ ] UFW 防火墙已配置并启用
- [ ] 记录到 `servers.yaml`

---

## 详细步骤

### 1. SSH 连接测试

```bash
ssh root@<新服务器IP> -p <端口>
```

连上后先看系统信息：

```bash
cat /etc/os-release
free -h
df -h
uname -a
```

### 2. 修改 SSH 端口

```bash
# 编辑 sshd 配置
vim /etc/ssh/sshd_config
# 找到 #Port 22，改为：
Port 63847
# 找到 #PasswordAuthentication yes，改为：
PasswordAuthentication no
# 找到 #PermitRootLogin yes，改为：
PermitRootLogin yes  # 或 no，取决于你的需求

# 重启 sshd
systemctl restart sshd
```

> ⚠️ **先别退出当前会话**，新端口连上了再退。

### 3. 配置 UFW 防火墙

```bash
apt update && apt install -y ufw

# 先放行 SSH（新端口）
ufw allow 63847/tcp

# 放行已有服务的端口（按需）
# ufw allow 80/tcp
# ufw allow 443/tcp

# 启用 UFW（输入 y 确认）
ufw enable

# 检查状态
ufw status verbose
```

### 4. 配置 fail2ban

```bash
apt install -y fail2ban
systemctl enable --now fail2ban

# 检查状态
systemctl status fail2ban
fail2ban-client status
```

### 5. 安装常用工具（可选）

```bash
apt install -y curl wget git htop bmon tree
```

### 6. 记录到 servers.yaml

```yaml
# 编辑 ~/.hermes/inventory/servers.yaml，添加节点：
nodes:
  - name: "新节点昵称"
    ip: "<IP>"
    ssh_port: 63847
    ssh_user: root
    sudo_password: "4561834"   # 占位，实际从 keychain 取
    proxy_port: <端口>          # 如果装了代理
    uuid: "<节点UUID>"         # 如果是 V2Ray 节点
    sid: ""                    # 如果是 shadowtls
    pbk: ""                    # 如果是 sing-box reality
    tags: [new]
```

### 7. 验证基线

```bash
# 防火墙
ufw status verbose
# → 应该只有 63847/tcp ALLOW，22/tcp 应该没有

# fail2ban
fail2ban-client status sshd
# → 应该看到 0 banned

# SSH 端口验证（开一个新会话）
ssh root@<新服务器IP> -p 63847
# → 应该能连上，且不能用密码
```

---

## 回滚步骤

如果 SSH 连不上了（防火墙规则写错）：

```bash
# 物理 console / VNC 登录
# 关闭 UFW
ufw disable

# 检查规则
ufw status

# 重新配置后开启
ufw enable
```

---

## 常见错误

| 症状 | 原因 | 修复 |
|------|------|------|
| `Connection refused` | sshd 没重启 | `systemctl restart sshd` |
| `Permission denied (publickey)` | 密钥没传 | `ssh-copy-id` |
| `ufw: command not found` | 没装 ufw | `apt install ufw` |
| `fail2ban not found` | 没装 fail2ban | `apt install fail2ban` |
