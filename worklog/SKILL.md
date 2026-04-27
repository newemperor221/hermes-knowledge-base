# Worklog — 实操踩坑记录

**用途**：所有实际操作中踩过的坑、解决方案、工作流程，全部归档到这里。
**原则**：同一个坑不犯第二次。

---

## 索引

| 域 | 文件 | 关键问题 |
|----|------|---------|
| sing-box Reality 部署 | `sing-box-reality.md` | 字段名/结构/utls fingerprint |
| SSL + Let's Encrypt | `ssl-letsencrypt.md` | 80端口占用/webroot/docker volume |
| Docker + nginx | `docker-nginx.md` | docker-compose/健康检查/重启策略 |
| 服务器初始化 | `server-init.md` | UFW/fail2ban/SSH加固 |
| Atlanta 练手总结 | `atlanta-summary.md` | 全流程复盘 |

---

## 添加新踩坑

每次遇到新坑时：
1. 找到对应域的文件
2. 追加到 `## 踩坑记录` 板块
3. 格式：`**问题** → **解决方案** + 验证方法`

---

## 检索

```bash
grep -r "关键词" ~/.hermes/inventory/worklog/
```
