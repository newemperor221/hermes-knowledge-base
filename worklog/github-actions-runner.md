# GitHub Actions Self-Hosted Runner 踩坑

**最近更新**：2026-04-27

---

## 踩坑记录

### 1. runner 无 SSH 私钥
**问题**：workflow 里 ssh 报 `Permission denied (publickey)`
**原因**：runner 在服务器上但没有 SSH 私钥，本地私钥不会自动同步
**解决**：私钥存 GitHub secret，workflow 里写入 `~/.ssh/id_ed25519`

### 2. 浅克隆没有 HEAD~1
**问题**：`git diff HEAD~1` 报错 `unknown revision`
**解决**：checkout 加 `fetch-depth: 0`

### 3. SSH host key 未信任
**问题**：`Host key verification failed`
**解决**：`ssh-keyscan -H -p PORT IP >> ~/.ssh/known_hosts`

### 4. gh 和 git 凭证不互通
**问题**：gh 登录后 git push 仍认证失败
**解决**：`git config --global credential.helper "/usr/bin/gh auth git-credential"`，remote 用 HTTPS

---

## 验证

```bash
gh api /repos/OWNER/REPO/actions/runners
# 应显示: atlanta-runner online
gh run list --repo OWNER/REPO --limit 3
```
