# GitHub Actions self-hosted runner 踩坑

**最近更新**：2026-04-27

---

## 踩坑记录

### 1. runner SSH 无法连接 — Permission denied (publickey)
**问题**：workflow 里 SSH 连接到 Atlanta 失败
**原因**：runner 进程用的是 `/home/woioeow/actions-runner/` 下的 SSH 配置，没有私钥
**解决**：把 SSH 私钥存为 GitHub Secret（`ATLANTA_SSH_KEY`），workflow 运行时写入 `~/.ssh/id_ed25519`

```yaml
- name: Setup SSH key
  env:
    ATLANTA_SSH_KEY: ${{ secrets.ATLANTA_SSH_KEY }}
  run: |
    mkdir -p ~/.ssh
    echo "$ATLANTA_SSH_KEY" > ~/.ssh/id_ed25519
    chmod 600 ~/.ssh/id_ed25519
    ssh-keyscan -H -p 53621 23.95.218.144 >> ~/.ssh/known_hosts 2>/dev/null
```

**相关文件**：`~/.hermes/inventory/.github/workflows/deploy.yml`

---

### 2. 浅克隆导致 `HEAD~1` 未知
**问题**：`git diff --name-only HEAD~1` 报错 `unknown revision`
**原因**：`actions/checkout@v4` 默认 `fetch-depth: 1`，只有一个 commit
**解决**：加 `fetch-depth: 0` 获取完整历史

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

---

### 3. SSH heredoc 需要 `StrictHostKeyChecking=no`
**问题**：第一次 SSH 连接问 `Are you sure you want to continue connecting`
**原因**：workflow 非交互式，stdin 不是 terminal
**解决**：加 `-o StrictHostKeyChecking=no`

---

### 4. runner gh 认证需要 token 环境变量
**问题**：runner 上 `gh api` 报错 `GH_TOKEN` not set
**原因**：runner 的 gh 用自己的凭证存储，但某些环境变量读取不到
**解决**：`GH_TOKEN=$(gh auth token) && ssh ... "GH_TOKEN='$GH_TOKEN' gh api ..."`

---

### 5. self-hosted runner 下载 URL
**问题**：`gh run self-hosted add` 不是正确命令
**解决**：手动下载 runner 包并注册
```bash
# 获取 runner 包
RUNNER_URL=$(curl -s -H "Authorization: token $GH_TOKEN" \
  https://api.github.com/repos/actions/runner/releases/latest | \
  grep -o '"browser_download_url": "[^"]*linux-x64[^"]*"' | cut -d'"' -f4)

# 下载并注册
curl -L -o runner.tar.gz $RUNNER_URL
tar xzf runner.tar.gz
./config.sh --name atlanta-runner --url <repo-url> --token <token> --labels atlanta --unattended
./svc.sh install && ./svc.sh start
```

---

## 完整 workflow 模板

```yaml
name: Atlanta Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: [self-hosted, atlanta]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup SSH key
        env:
          ATLANTA_SSH_KEY: ${{ secrets.ATLANTA_SSH_KEY }}
        run: |
          mkdir -p ~/.ssh
          echo "$ATLANTA_SSH_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H -p 53621 23.95.218.144 >> ~/.ssh/known_hosts 2>/dev/null

      - name: Deploy
        run: |
          ssh -o StrictHostKeyChecking=no woioeow@23.95.218.144 -p 53621 << 'EOF'
            # deploy commands here
          EOF
```

---

## 验证

```bash
# runner 状态
gh api /repos/<owner>/<repo>/actions/runners

# 查看 workflow 日志
gh run view <run-id> --log

# 手动触发
gh run watch <run-id>
```
