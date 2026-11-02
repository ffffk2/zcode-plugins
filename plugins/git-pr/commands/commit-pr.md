---
description: 一条命令完成提交并创建 PR/MR：先按仓库风格提交，再推送并创建 PR
argument-hint: "[可选：提交说明 / 目标分支 / 标题补充]"
allowed-tools: Bash, Read, Grep, Glob
---

# 提交并创建 PR

等价于依次执行 `/commit` 和 `/create-pr`，但只做一次上下文检查。规则与两个命令完全一致：不跳过钩子、不 force push、不编造验证结果、不提交凭据文件。

## 1. 检查仓库

- `git rev-parse --show-toplevel` 失败 → 停止（非 Git 仓库）。
- `git remote get-url origin`；没有 remote → 停止并提示配置 origin。
- `git branch --show-current`；detached HEAD → 停止。

## 2. 提交（若有改动）

- 收集 `git status --short`、`git diff`、`git diff --cached`、`git log --oneline -20`、`git diff --stat`。
- 有改动时：
  - 先做安全审查：排除 `.env`、`*.pem`、`*.key`、`id_rsa*`、`credentials*`、真实密钥字面量、超过 5 MB 的二进制；命中则停止并列出文件。
  - 默认 `git add -A`；参数若是仓库内存在的路径则只暂存这些路径。
  - 提交信息跟随最近 20 条提交的语言与格式；主题行 ≤72 字符，必要时才写正文；不加 `Generated with ...` 署名。
  - 用 heredoc 执行 `git commit -F -`；pre-commit hook 失败时修复或原样报告，不要 `--no-verify`。
- 没有改动时：确认 `git log origin/<base>..HEAD` 仍有领先提交，否则停止。

## 3. 推送

- 确定 base：第一个参数若是远端真实分支就用它，否则按 `origin/HEAD` → `git ls-remote --symref origin HEAD` → `main` → `master` 探测；用 `git ls-remote --heads origin <base>` 确认存在，都不存在时取远端第一个非 HEAD 分支。
- 当前分支等于 base：
  - 有刚提交的内容 → `git switch -c <kebab-case 分支名>` 后再推送（提交已经产生，直接基于它建分支）；若提交已在 base 上，停止并说明需要先把提交移到新分支。
  - 无内容 → 停止。
- `git push -u origin HEAD`（已有上游则 `git push`）。推送失败原样报告，禁止 force。

## 4. 创建 PR

- 标题取提交主题或归纳多提交；若仓库存在 `.github/PULL_REQUEST_TEMPLATE.md` 等模板则沿用其结构。
- 正文只写：

```markdown
## 变更内容
- ...

## 验证
- 已执行：<实际做过的检查>
```

- 按以下优先级创建：
  1. GitHub + `gh` 已登录：`gh pr create --base <base> --head <head> --title <title> --body-file <file>`；已存在则汇报现有链接。
  2. GitHub + `GITHUB_TOKEN`/`GH_TOKEN`：用 REST API `POST https://api.github.com/repos/<owner>/<repo>/pulls`，JSON 用 python/node/jq 生成（优先 python），422 已存在时查询现有 PR。
  3. GitHub 无工具：输出预填 `compare/<base>...<head>?expand=1&title=...&body=...` 链接，让用户点 “Create pull request”。
  4. GitLab：`glab mr create ... --yes`，否则输出 `/-/merge_requests/new?...` 链接。
  5. 其他平台：输出 `compare/<base>...<head>` 链接。
- 全程不要回显 token。

## 5. 汇报

给出 PR/MR 链接、源分支 → 目标分支、提交哈希与标题、创建方式；兜底时说明用户唯一需要点的一步。若 `gh` 未安装，附一次安装建议即可。
