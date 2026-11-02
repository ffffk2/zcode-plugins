---
description: 推送当前分支并创建 Pull Request / Merge Request（GitHub 优先，支持 GitLab 与网页链接兜底）
argument-hint: "[目标分支] [可选：标题或补充说明]"
allowed-tools: Bash, Read, Grep, Glob
---

# 创建 PR / MR

把当前分支推送并创建 PR。优先使用 `gh`；不可用时依次降级为 GitHub API（需要 token）、预填网页链接、GitLab `glab` 或网页链接。全程禁止 force push。

## 1. 前置检查

- `git rev-parse --show-toplevel` 失败 → 停止，说明不是 Git 仓库。
- `git remote get-url origin`；没有 remote → 停止并提示先执行 `git remote add origin <url>`。
- 从 remote URL 解析主机与仓库路径，兼容两种写法：
  - `git@github.com:owner/repo.git`
  - `https://github.com/owner/repo.git`
  - 自建 GitLab 可能有多层命名空间（`https://host/group/sub/repo.git`），仓库路径要保留完整层级。
- 判断平台：主机含 `github` 视为 GitHub；含 `gitlab` 视为 GitLab；其余先按通用 Git 服务处理。

## 2. 确定 base 分支

- 如果第一个参数是一个真实存在的远端分支（`git ls-remote --heads origin <参数>` 有输出），把它作为 base；否则整个参数串都当作标题/说明提示，忽略其分支含义。
- 未指定时按顺序探测：
  1. `git symbolic-ref --short refs/remotes/origin/HEAD`（去掉 `origin/` 前缀）
  2. `git ls-remote --symref origin HEAD`（取 `ref:` 后面的分支名）
  3. `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`（仅在 gh 可用时）
  4. 回退到 `main`，再回退到 `master`
- 用 `git ls-remote --heads origin <base>` 确认 base 真实存在。若 main 和 master 都不存在，取 `git for-each-ref --format='%(refname:short)' refs/remotes/origin` 中第一个非 `origin/HEAD` 的分支；仍为空则停止，说明远端没有可用分支。
- 执行 `git fetch origin <base>` 更新基线。

## 3. 处理当前分支与提交

- `head=$(git branch --show-current)`；为空（detached HEAD）→ 停止。
- 若 `head` 等于 base：
  - 有未提交改动：基于改动生成 kebab-case 分支名（短英文关键词，如 `fix/login-timeout`），执行 `git switch -c <branch>`，然后按 `/commit` 的规则完成一次提交（同样的安全审查、仓库风格、无署名）。
  - 既无改动也无领先提交：停止，说明当前就在目标分支且没有可提交内容。
- 分支上有未提交改动（`git status --porcelain` 非空）：先按 `/commit` 的规则提交，不要带着未提交改动创建 PR。
- `git log origin/<base>..HEAD --oneline` 为空 → 停止，说明没有领先 base 的提交。

## 4. 推送

- 没有上游时 `git push -u origin HEAD`，否则 `git push`。
- 推送被拒绝（非快进、权限、保护分支）→ 原样报告错误并停止，**不要**使用 `--force` 或 `--force-with-lease`。

## 5. 生成标题与正文

- 标题：优先用分支上唯一提交的主题行；多个提交时归纳成一句能看懂的标题。语言跟随仓库，不超过 72 个字符。
- 先看仓库是否有 PR 模板：`.github/PULL_REQUEST_TEMPLATE.md`、`.github/pull_request_template.md`、`PULL_REQUEST_TEMPLATE.md`、`docs/PULL_REQUEST_TEMPLATE.md`。有模板就沿用其章节结构，没有则使用下面的结构。
- 正文只写能从提交与 diff 中确认的事实：

```markdown
## 变更内容
- ...

## 验证
- 已执行：<实际做过的检查>
- 未运行自动化测试（仅在确实没有运行时写）
```

- 不要编造测试结果。用 `git diff origin/<base>...HEAD --stat` 和提交正文核对内容。
- 把标题和正文写入临时文件（`tmp=$(mktemp -d)`），后续命令用 `--body-file` 或读取文件，避免引号与换行转义问题。

## 6. 按平台与可用工具创建

先探测：`command -v gh`、`command -v glab`、`[ -n "$GITHUB_TOKEN" ] || [ -n "$GH_TOKEN" ]`。

### A. GitHub 且 gh 可用且已登录（`gh auth status` 成功）

```bash
gh pr create --base "<base>" --head "<head>" --title "<title>" --body-file "$tmp/body.md"
```

- 已存在同 head 的 PR 时该命令会失败：改用 `gh pr list --head "<head>" --state open --json url,title`，汇报现有 PR 链接，不要重复创建。

### B. GitHub 且无 gh 但有 token（`GITHUB_TOKEN` 或 `GH_TOKEN`）

用 REST API 创建。先用脚本生成 JSON 请求体，再 `curl`。按 `python` → `python3` → `node` → `jq` 的可用顺序选择，不要手工拼接字符串：

```bash
# python（Windows 上通常是 python）
python -c "import json,sys; json.dump({'title':open(sys.argv[1],encoding='utf-8').read().strip(),'head':sys.argv[2],'base':sys.argv[3],'body':open(sys.argv[4],encoding='utf-8').read()}, open(sys.argv[5],'w',encoding='utf-8'), ensure_ascii=False)" "$tmp/title" "$head" "$base" "$tmp/body.md" "$tmp/req.json"

# 或 jq：jq -n --rawfile body "$tmp/body.md" --rawfile title "$tmp/title" \
#   --arg head "$head" --arg base "$base" \
#   '{title:($title|rtrimstr("\n")),head:$head,base:$base,body:$body}' > "$tmp/req.json"

curl -sS -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -H "Content-Type: application/json" \
  --data @"$tmp/req.json" \
  "https://api.github.com/repos/<owner>/<repo>/pulls"
```

- GitHub Enterprise 用 `https://<host>/api/v3/repos/<owner>/<repo>/pulls`。
- 返回 422 且提示 PR 已存在时，用 `GET .../pulls?head=<owner>:<head>&state=open` 取现有 PR 链接并汇报。
- 不要在输出里回显 token。

### C. GitHub 且既无 gh 也无 token

推送后生成预填链接，并明确告诉用户点开后点击 “Create pull request”：

```bash
enc() {
  if command -v python >/dev/null 2>&1; then
    python -c "import sys,urllib.parse;print(urllib.parse.quote(sys.argv[1]))" "$1"
  elif command -v python3 >/dev/null 2>&1; then
    python3 -c "import sys,urllib.parse;print(urllib.parse.quote(sys.argv[1]))" "$1"
  elif command -v node >/dev/null 2>&1; then
    node -e "console.log(encodeURIComponent(process.argv[1]))" "$1"
  elif command -v jq >/dev/null 2>&1; then
    jq -rn --arg v "$1" '$v|@uri'
  else
    printf '%s' "$1"
  fi
}
echo "https://<host>/<owner>/<repo>/compare/<base>...<head>?expand=1&title=$(enc "<title>")&body=$(enc "$(cat "$tmp/body.md")")"
```

- 如果链接过长或编码不可用，退化为不带参数的 `https://<host>/<owner>/<repo>/compare/<base>...<head>`，把标题和正文直接给用户复制。

### D. GitLab

- `glab` 可用且已登录：`glab mr create --source-branch "<head>" --target-branch "<base>" --title "<title>" --description "$(cat "$tmp/body.md")" --yes`
- 否则输出：
  `https://<host>/<group>/<repo>/-/merge_requests/new?merge_request[source_branch]=<head>&merge_request[target_branch]=<base>`

### E. Gitea / Forgejo / 其他

推送后输出 `<host>/<owner>/<repo>/compare/<base>...<head>` 链接，说明需要在网页上创建。

## 7. 汇报

- 成功：给出 PR/MR 链接、源分支 → 目标分支、标题，以及创建方式（gh / API / 网页链接）。
- 兜底：给出预填链接和用户唯一需要做的一步操作。
- 失败：原样给出错误信息，并说明卡在哪一步（检查、提交、推送还是创建）。
- 如果 `gh` 不存在，可在结尾附一句安装建议（Windows：`winget install --id GitHub.cli`；macOS：`brew install gh`），同一个会话里不要重复提醒。
