# git-pr

用斜杠命令完成「提交代码」和「创建 PR」。命令由 ZCode 代理执行真实的 `git` / `gh` / `gh api` 操作，而不是脚本模板。

## 命令

| 命令 | 作用 |
| --- | --- |
| `/commit` | 审查改动、安全扫描、暂存，并按仓库既有风格提交（只提交到本地，不推送） |
| `/create-pr` | 推送当前分支并创建 PR/MR；需要时会先补提交或创建特性分支 |
| `/commit-pr` | 一次完成提交、推送和创建 PR |

可选参数：

- `/commit src/`：只暂存 `src/` 下的改动
- `/commit 修复登录超时`：把参数当作提交说明提示
- `/create-pr develop`：以 `develop` 为目标分支（该分支必须存在于远端）
- `/create-pr 补充性能测试说明`：把参数当作 PR 标题/说明提示

## 行为约定

- 提交信息跟随 `git log` 中最近 20 条的风格与语言，不添加工具署名。
- 提交前扫描凭据文件、真实密钥字面量和超大二进制文件，命中即停止。
- 不使用 `--no-verify` 跳过钩子，不执行 force push。
- PR 正文只写能从提交和 diff 确认的内容，不编造测试结果。
- 支持仓库内的 PR 模板（`.github/PULL_REQUEST_TEMPLATE.md` 等）。

## PR 创建方式（自动降级）

1. GitHub：`gh pr create`（需要 `gh auth login`）
2. GitHub：`GITHUB_TOKEN` / `GH_TOKEN` + REST API
3. GitHub：输出预填的 compare 链接，网页点击创建
4. GitLab：`glab mr create`，否则输出 merge request 新建链接
5. Gitea / Forgejo / 其他：输出 compare 链接

所以没有安装 `gh` 也能用：推送完成后会给出预填标题和正文的链接。

## 可选依赖

- `git`：必需
- `gh`：推荐（Windows `winget install --id GitHub.cli`，macOS `brew install gh`）
- `glab`：仅 GitLab 用户需要
- `python` / `node` / `jq`（任选其一）：仅在用 token 调 GitHub API 或做 URL 编码时需要；都没有时自动退回纯链接

## 安全

插件不会读取或写入任何凭据文件，token 只从环境变量读取且不会回显。命令的所有 git 操作都在你当前打开的仓库中执行。
