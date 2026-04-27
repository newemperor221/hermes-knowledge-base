# Runbook: InkOS 番茄小说流水线

> 状态：**待执行**（架构已设计，尚未实际跑通）
> 外层调度：Hermes Agent，内层执行：InkOS 多Agent Pipeline
> 模型：DeepSeek V3，bin路径：`~/.hermes/node/bin/`

---

## 目标

自动化生产番茄小说平台的网文内容：
1. 生成章节初稿
2. 审校（错别字、逻辑）
3. 改写（提升文笔）
4. 输出可投稿版本

---

## 架构图

```
Hermes Agent（外层规划）
    │
    ├── 接收用户指令（"写一个都市异能小说"）
    │
    ├── 调 InkOS Pipeline
    │       ├── Writer Agent（DeepSeek V3）→ 初稿
    │       ├── Reviewer Agent（DeepSeek V3）→ 审校意见
    │       └── Rewriter Agent（DeepSeek V3）→ 改写稿
    │
    └── 输出最终章节
```

---

## 前置条件

- [ ] InkOS v1.3.6 已安装：`ls ~/.hermes/node/bin/inkos*`
- [ ] DeepSeek V3 API Key 已配置（在环境变量或配置文件中）
- [ ] 番茄小说账号 + 后台权限

---

## 启动流程（待补全）

```bash
# 查看 InkOS 版本
inkos --version

# 查看可用 pipeline
inkos list

# 启动番茄小说 pipeline（具体命令待补）
inkos start --pipeline novel --model deepseek-v3
```

---

## 配置 DeepSeek V3（待补全）

```bash
# 环境变量
export DEEPSEEK_API_KEY="sk-xxxx"
export DEEPSEEK_BASE_URL="https://api.deepseek.com"
```

---

## 投稿番茄小说（待补全）

- 番茄作家后台：https://writer.fantech.com
- 分类选择
- 更新频率设置
- 签约条件

---

## 已知问题

| 问题 | 状态 |
|------|------|
| InkOS 多Agent 通信机制未验证 | ❌ 待测 |
| DeepSeek V3 并发限制未知 | ❌ 待测 |
| 番茄反抄袭检测应对策略 | ❌ 待研究 |
| 单章节生成耗时（影响更新频率） | ❌ 待测 |

---

## 待办

- [ ] 跑通 InkOS 单章节生成
- [ ] 验证多Agent Review/Rewrite 流程
- [ ] 测试 DeepSeek V3 并发上限
- [ ] 摸清番茄小说更新频率与推荐关系
- [ ] 写正式 runbook 填充上述"待补全"
