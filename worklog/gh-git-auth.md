# gh vs git 凭证冲突

**问题**：gh 已登录（`gh auth status` ✅），但 `git push` 报 `Authentication failed`

**原因**：gh 和 git 用不同的凭证存储，gh 登录不让 git 自动获得 GitHub 写权限

**解决**：让 git 借用 gh 的 SSH 方式
```bash
# 改 remote 为 SSH URL
git remote set-url origin git@github.com:newemperor221/hermes-knowledge-base.git

# 强制 git 用 SSH 密钥
GIT_SSH_COMMAND='ssh -i ~/.ssh/id_ed25519' git push -u origin main
```

**验证**：
```bash
gh auth status   # 确认 gh 已登录
git remote -v   # 确认是 git@github.com 格式
git push        # 确认成功
```
