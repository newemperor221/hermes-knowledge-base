# GitHub CLI + Git 凭证互通

**最近更新**：2026-04-27

---

## 问题

`gh auth login` 成功后，git push 仍然报认证失败。

原因：gh 和 git 各用各的凭证存储，不自动打通。

## 解决

配置 git 使用 gh 作为 credential helper：

```bash
git config --global credential.helper "/usr/bin/gh auth git-credential"
```

验证：
```bash
git config --global --list | grep credential
# credential.helper=/usr/bin/gh auth git-credential
```

验证推送：
```bash
cd ~/.hermes/inventory && git push origin main
```

## 踩坑记录

### 1. gh auth git-credential 需要绝对路径
**问题**：credential.helper 用相对路径 `gh auth git-credential` 无效
**解决**：用绝对路径 `/usr/bin/gh auth git-credential`

### 2. git remote 用 HTTPS 而非 SSH
**问题**：remote 设置为 `git@github.com:xxx` 时，ssh 认证走 SSH key，不走 gh credential helper
**解决**：remote 设为 `https://github.com/xxx/xxx.git`，走 HTTPS 才用 credential helper

### 3. inventory 仓库 force push
**问题**：remote 有旧提交历史，force push 被拒
**解决**：`git push origin main --force`（本地的 worklog 是权威版本）
