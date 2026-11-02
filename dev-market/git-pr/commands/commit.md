---
description: 审查并暂存当前改动，按仓库既有风格生成提交信息并完成本地 git commit（不推送）
argument-hint: "[可选：路径范围（如 src/）或提交说明补充]"
allowed-tools: Bash, Read, Grep, Glob
---

# 提交代码

执行一次完整的本地提交：确认仓库状态 → 审查改动 → 安全扫描 → 暂存 → 生成符合仓库风格的提交信息 → 提交 → 校验汇报。

## 1. 确认仓库与分支

- `git rev-parse --show-toplevel` 失败时停止，告诉用户当前目录不是 Git 仓库。
- `git branch --show-current`：为空说明处于 detached HEAD，停止并提示先切换到分支。

## 2. 收集上下文

并行执行并阅读结果：

- `git status --short`
- `git diff --stat`
- `git diff`（未暂存内容）
- `git diff --cached`（已暂存内容）
- `git log --oneline -20`（学习提交信息的风格与语言）
- 仓库还没有任何提交时 `git log` 会报错，忽略错误继续。

## 3. 没有改动就停止

`git status --porcelain` 为空时，直接说明「没有可提交的改动」，不要创建空提交。

## 4. 安全审查

暂存前检查改动中是否包含敏感内容：

- `.env`、`.env.*`、`*.pem`、`*.key`、`*.p12`、`id_rsa*`、`credentials*` 等凭据文件
- 新增行中出现 `password`、`secret`、`token`、`api_key`、`private_key` 后面的真实字面量（占位符和读取环境变量的写法不算）
- 超过 5 MB 的二进制文件

命中时不要自动提交：列出文件与命中原因，停止并等待用户处理。只有当用户在参数中明确要求包含某个文件时才继续。

## 5. 暂存

- 默认 `git add -A`（包含新增、修改、删除）。
- 如果参数指向的是仓库内已存在的路径（例如 `src/`、`backend`），只暂存这些路径：`git add -- <paths>`。
- 如果参数是提交说明而不是路径，仍使用 `git add -A`，并把它作为第 6 步的写作提示。
- 暂存后再次用 `git status --short` 确认实际进入提交的内容。

## 6. 生成提交信息

- 语言和格式跟随 `git log --oneline -20`：历史用 Conventional Commits（`feat:`/`fix:`/`refactor:`…）就沿用；历史是纯中文描述就用中文；两者都不明显时默认 Conventional Commits，主题行可用中文。
- 主题行格式：`type(scope): 简述`，不超过 72 个字符，动词开头，写清「做了什么」，不要写「修改了一些代码」这类空话。
- 只有需要解释原因、影响面或权衡时才写正文，正文每行不超过 72 个字符，与主题行之间空一行。
- 破坏性变更使用 `!` 并在正文加 `BREAKING CHANGE:` 脚注。
- 不要添加 `Generated with ...` 之类的署名，除非用户明确要求。

## 7. 提交

使用 heredoc 提交，避免引号转义问题：

```bash
git commit -F - <<'EOF'
<主题行>

<正文，可省略>
EOF
```

- 有 pre-commit hook 且失败时：阅读错误信息并修复能修复的问题；修复不了就原样报告并停止。
- 不要使用 `--no-verify` 跳过钩子，除非用户明确要求。

## 8. 校验并汇报

- `git log -1 --stat` 与 `git status --short`
- 汇报：短提交哈希、完整提交信息、涉及文件数、仍未提交的剩余内容。
- 说明本次只做了本地提交，尚未推送；下一步可执行 `/create-pr`。
