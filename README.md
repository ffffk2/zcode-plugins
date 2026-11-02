# zcode-git-pr

一个 ZCode 插件：用斜杠命令完成「提交代码」和「创建 Pull Request / Merge Request」。

## 仓库结构

```
dev-market/
  marketplace.json      # 本地开发市场清单，供 ZCode 添加
  git-pr/               # 插件源码
    .zcode-plugin/plugin.json
    commands/           # /commit、/create-pr、/commit-pr
    README.md           # 插件详细说明
```

## 命令

| 命令 | 作用 |
| --- | --- |
| `/commit` | 审查改动、安全扫描、暂存，并按仓库既有风格提交（不推送） |
| `/create-pr` | 推送当前分支并创建 PR/MR；需要时先建特性分支并提交 |
| `/commit-pr` | 一次完成提交、推送和创建 PR |

PR 创建会自动降级：`gh` → GitHub Token API → 预填网页链接；GitLab 使用 `glab` 或 MR 链接。提交前会检查凭据文件、真实密钥字面量和超大二进制文件；不使用 `--no-verify`，不执行 force push。

## 安装

在 ZCode 中打开 **Plugin Marketplace → Add → Add Plugin Marketplace**，添加本仓库的 `dev-market` 目录（或克隆后的对应路径），然后安装 `git-pr` 插件。

## 更多说明

见 [dev-market/git-pr/README.md](dev-market/git-pr/README.md)。
