# iptables NAT / 端口转发 知识点

## 一、核心概念

| 概念 | 作用 | 链 |
|------|------|----|
| **DNAT** (Destination NAT) | 修改报文**目的地址**，用于入站端口转发 | `PREROUTING` |
| **SNAT** (Source NAT) | 修改报文**源地址**，用于出站地址转换 | `POSTROUTING` |
| **MASQUERADE** | SNAT 的一种，自动用出口网卡的当前 IP，适合动态 IP | `POSTROUTING` |
| **FORWARD** | 转发链，处理既非本机产生也非发往本机的报文 | `FORWARD` |

## 二、典型场景

### 场景 A：Linux 网关（内网机器通过本机出去）

```
公网 eth0 (203.0.113.2)  ──→  Linux 防火墙  ──→  内网 eth1 (10.0.0.2)  ──→  Web Server (10.0.0.1)
```

```bash
# 1. 开启内核转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. DNAT：把外部访问本机 80 端口的请求转发到内网 10.0.0.1
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 10.0.0.1

# 3. SNAT：让回程报文正确路由回来（静态 IP 用 SNAT，动态 IP 用 MASQUERADE）
iptables -t nat -A POSTROUTING -o eth1 -p tcp --dport 80 -d 10.0.0.1 -j SNAT --to-source 10.0.0.2

# 4. FORWARD 规则（必须！）
iptables -A FORWARD -i eth0 -o eth1 -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -P FORWARD DROP
```

### 场景 B：端口映射（公网 2222 → 内网 SSH 22）

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to-destination 192.168.1.20:22
iptables -A FORWARD -p tcp -d 192.168.1.20 --dport 22 -j ACCEPT
```

### 场景 C：MASQUERADE（内网多台机器共用一个公网 IP 上网）

```bash
# 内网 192.168.1.0/24 通过 eth0 上网，自动用 eth0 当前 IP 做源地址转换
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# 允许转发
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## 三、iptables vs nftables

| | iptables | nftables |
|---|---|---|
| 核支持 | kernel 2.4+ | kernel 3.13+ |
| 规则数 | 每条规则单独匹配 | 一条规则可多重动作 |
| 计数器 | 内置每条规则计数 | 可选，按需开启 |
| 双栈 IPv4/IPv6 | 需要分别配置 | `inet` 表统一处理 |
| 复杂度 | 较简单 | 更简洁强大 |

**结论**：新系统用 nftables，老系统/兼容性用 iptables。Ubuntu 22.04+ 默认用 iptables-nft（底层是 nftables）。

## 四、Docker 对 iptables 的影响

Docker 会自动在 `nat` 表插入 MASQUERADE 和 PREROUTING 规则来实现容器网络：

```bash
# Docker 容器访问外部：自动 MASQUERADE
# 暴露容器端口：自动 DNAT
# 查看 Docker 写入的规则：
iptables -t nat -L -n -v | grep DOCKER
```

**踩坑**：Docker 的自动规则可能和手动规则冲突——Docker 规则优先级较高。

## 五 Atlanta 适用场景

Atlanta 是公网边缘服务器（23.95.218.144），不是内网网关，所以 NAT 场景主要是：

1. **端口转发**：把公网某个端口转发到 Docker 容器（Docker 自己会做，但可以手动增强）
2. **conntrack** 状态追踪：Atlanta 的 UFW FORWARD 链默认 DROP，需要状态追踪让回程包能回来
3. **1:1 NAT**：如果 Atlanta 分配了额外 IP，可以做双向映射

## 六、永久保存规则

```bash
# 保存
iptables-save > /etc/iptables/rules.v4

# 恢复
iptables-restore < /etc/iptables/rules.v4

# 开机自动加载（Debian/Ubuntu）
apt install iptables-persistent
netfilter-persistent save
```

## 七、常用查看命令

```bash
# 查看 NAT 表
iptables -t nat -L -n -v

# 查看 FORWARD 链
iptables -L FORWARD -n -v

# 查看活跃连接追踪
conntrack -L

# 按编号删除规则
iptables -t nat -D POSTROUTING 1
```
