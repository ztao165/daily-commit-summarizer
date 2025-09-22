# Daily Commit Summarizer

![cover](./cover.png)

[English](./README_en.md)

## 📌 使用场景

软件团队往往希望快速了解一天内代码库里发生了什么，而不是翻遍冗长的 git log 或大型 PR。

这个项目提供了一个 GitHub Actions 工作流 和 TypeScript 脚本，实现以下功能：
1. 每天北京时间 18:00（UTC+8）自动运行。
2. 收集当天在所有远程分支上的提交。
3. 借助 LLM（例如 OpenAI GPT-4.1-mini）：
	* 将大型 diff 拆分为可管理的片段。
	* 为每个提交单独生成摘要（包含变更内容、影响、风险、测试建议）。
	* 最后合并成一份每日总结报告。
4. 通过 Webhook 将总结发送到飞书群聊。

这样，团队每天都能收到一份简明、人类可读的变更日志，提高透明度，减少代码审查的时间成本。

<br/>

## 🚀 功能特点
1. 跨分支覆盖：支持分析所有 origin/* 分支上的提交。
2. 大 diff 切分：安全处理大规模提交，避免超出 LLM 上下文限制。
3. 多层次总结：单个 diff 片段 → 单次提交 → 每日汇总。
4. 飞书通知：每日简报自动推送至群聊。
5. 高度可配置：可调整模型、分支过滤、diff 拆分大小等参数。

<br/>

## ⚙️ 使用方法

**1. 克隆或 Fork 仓库**

```bash
git clone https://github.com/nanbingxyz/daily-commit-summarizer.git
cd daily-commit-summarizer
```

**2. 添加 GitHub Actions 工作流**

在 .github/workflows/daily-summary.yml 中加入：
```yaml
name: Daily LLM Commit Summary

on:
  schedule:
    - cron: "0 10 * * *"   # 10:00 UTC = 18:00 北京时间
  workflow_dispatch: {}     # 手动触发

jobs:
  run:
    runs-on: ubuntu-latest
    env:
      TZ: Asia/Shanghai
    steps:
      - name: Checkout summarizer
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Checkout target repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          repository: ${{ secrets.REPO }}
          token: ${{ secrets.REPO_CLONE_TOKEN }}
          path: target-repo

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Run summarizer
        working-directory: target-repo
        env:
          OPENAI_BASE_URL: ${{ secrets.OPENAI_BASE_URL }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          LARK_WEBHOOK_URL: ${{ secrets.LARK_WEBHOOK_URL }}
          REPO: ${{ secrets.REPO }}
        run: |
          npx --yes tsx ../scripts/daily-summary.ts
```

**3. 添加仓库密钥**

进入 Repo → Settings → Secrets and variables → Actions → New repository secret：
1. OPENAI_API_KEY → OpenAI 或兼容 LLM 服务的 API Key
2. OPENAI_BASE_URL → LLM 服务的基础 URL
3. LARK_WEBHOOK_URL → 飞书群自定义机器人 Webhook 地址
4. REPO → 你的 GitHub 仓库名
5. REPO_CLONE_TOKEN → 用于 checkout 目标仓库的 GitHub Personal Access Token（至少需要读取代码的权限）

> 小贴士：脚本通过 `npx --yes tsx` 直接运行，无需在 Workflow 中执行 `npm install`，可避免目标仓库旧依赖的构建失败；请保证 `working-directory` 与上方 `Checkout target repository` 的 `path` 保持一致（示例中均为 `target-repo`）。

<br/>

## 📄 脚本说明

scripts/daily-summary.ts 是核心逻辑：
1. 获取所有远程分支 (git fetch --all)。
2. 收集当天的提交 (git log --since "midnight" --until "now" --all)。
3. 生成 diff 并进行切分。
4. 调用 LLM API，生成结构化的提交摘要。
5. 合并为当日报告。
6. 通过 Webhook 推送至飞书。

## 🖥 飞书示例输出
```markdown
# 2025-08-22 每日提交报告 (your-repo)

1. 总览
- 修复登录流程中的 bug
- 新增发票相关 API
- 调整开发流水线配置

2. 按分支的主要改动
- origin/feature/auth: 新增 JWT 校验中间件
- origin/hotfix/payment: 修复货币转换的舍入错误

3. 风险与回滚
- 鉴权中间件可能影响旧客户端 → 建议在预发环境验证
- 支付修复涉及公共工具 → 需要回归测试

4. 测试建议
- 增加 JWT 过期的单元测试
- 新增发票创建 API 的集成测试

5. 其他说明
- 忽略了 lockfile 更新
```

<br/>

## 🔧 配置项

|变量名|默认|说明|
|---|---|---|
MODEL_NAME|gpt-4.1-mini|使用的 LLM 模型
PER_BRANCH_LIMIT|200|每个分支每日最多分析的提交数
DIFF_CHUNK_MAX_CHARS|80000|每次请求最大 diff 字符数
TZ|Asia/Shanghai|定义 “今天” 的时区

<br/>

## 📌 注意事项
* 飞书纯文本消息不支持 Markdown。
* 如果需要富文本格式（标题、链接、列表等），请考虑在 postToLark() 中使用 msg_type: post。
* 私有仓库需注意：不要将代码上传至第三方 LLM 服务，除非符合公司合规要求。可替换为内部 LLM 网关。
* 由于本人使用的是 Azure OpenAI， 请求路径与 OpenAI API 的不同，若使用其他服务请自行调整。

<br/>

## 🤝 贡献方式

欢迎贡献！一些扩展方向：
* 增加 Slack / Discord / MS Teams 的适配器。
* 除了每日摘要，还支持在 PR 中直接生成评论。
* 输出扩展：比如展示修改文件数、代码行数统计等。

<br/>

## 📜 许可证

MIT License，自行承担使用风险。

## 🔍 你可能感兴趣

[![issue2task](https://img.shields.io/badge/GitHub-issue2task-blue?logo=github)](https://github.com/nanbingxyz/issue2task)

**[issue2task](https://github.com/nanbingxyz/issue2task)** —— 一个 Python 工具，可以把冗长的 GitHub Issue（含全部评论）通过 AI 总结为简洁、可执行的任务，  
并可自动添加到 GitHub Project v2 看板中。  
非常适合将复杂的讨论转化为清晰的下一步行动。
