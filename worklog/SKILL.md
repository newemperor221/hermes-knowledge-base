# Worklog — 实操踩坑记录

**用途**：所有实际操作中踩过的坑、解决方案、工作流程，全部归档到这里。
**原则**：同一个坑不犯第二次。

---

## 索引

| 域 | 文件 | 关键问题 |
|----|------|---------|
| GitHub 凭证（gh vs git） | `gh-git-auth.md` | gh 已登录但 git push 失败 |
| sing-box Reality 部署 | `sing-box-reality.md` | 字段名/结构/utls fingerprint |
| SSL + Let's Encrypt | `ssl-letsencrypt.md` | 80端口占用/webroot/docker volume |
| Docker + nginx | `docker-nginx.md` | docker-compose/健康检查/重启策略 |
| 服务器初始化 | `server-init.md` | UFW/fail2ban/SSH加固 |
| Atlanta 练手总结 | `atlanta-summary.md` | 全流程复盘 |

---

## 添加新踩坑

**强制规则**：踩坑后立即写入，不得"之后再说"。

触发条件（满足任一即写）：
- 工具调用报错，涉及之前没见过的错误 → 立即查因并写入
- 跑完一个完整流程（5+ 工具调用）→ 回顾有没有新坑，有则写
- 用户说"这个你之前不是弄过吗" → 说明没存档，立即补

写入时机：问题解决后的第一个空闲窗口，立即写文件，不要延后。

格式：
```markdown
### N. [简短标题]
**问题**：...
**原因**：...
**解决**：...
**验证**：...
**相关文件**：~/.hermes/inventory/worklog/xxx.md
```

---

## 检索

```bash
grep -r "关键词" ~/.hermes/inventory/worklog/
```
