# zcode-plugins

个人 ZCode 插件集合。所有插件放在同一个仓库里统一维护、一起提交；每个插件独立版本、独立安装。

## 仓库结构

```
marketplace.json        # 市场清单：列出仓库内所有插件
plugins/
  git-pr/               # 插件 1：提交代码与创建 PR
  <future-plugin>/      # 以后新增的插件，与 git-pr 平级
README.md
CONTRIBUTING.md         # 新增插件的步骤与校验清单
```

`marketplace.json` 里的 `plugins` 是数组，新增插件只需要：

1. 在 `plugins/` 下创建插件目录；
2. 在 `marketplace.json` 里追加一条 `source: "./plugins/<name>"`；
3. 提交并推送。

ZCode 刷新市场后即可看到新插件。具体步骤见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 当前插件

### git-pr

| 命令 | 作用 |
| --- | --- |
| `/commit` | 审查改动、安全扫描、暂存，并按仓库既有风格提交（不推送） |
| `/create-pr` | 推送当前分支并创建 PR/MR；需要时先建特性分支并提交 |
| `/commit-pr` | 一次完成提交、推送和创建 PR |

PR 创建会自动降级：`gh` → GitHub Token API → 预填网页链接；GitLab 使用 `glab` 或 MR 链接。提交前会检查凭据文件、真实密钥字面量和超大二进制文件；不使用 `--no-verify`，不执行 force push。

详细说明见 [plugins/git-pr/README.md](plugins/git-pr/README.md)。

## 安装

在 ZCode 中打开 **Plugin Marketplace → Add → Add Plugin Marketplace**，添加本仓库根目录（包含 `marketplace.json` 的目录），然后安装需要的插件。

本地开发时直接添加工作区路径即可，修改插件后在市场里刷新/更新。

## 反馈

有问题或想新增插件，欢迎在 [Issues](https://github.com/ffffk2/zcode-plugins/issues) 中提出。
