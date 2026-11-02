# 新增插件

本仓库是一个多插件市场：一个 `marketplace.json` 加多个 `plugins/<name>/`。新增插件不需要动仓库结构，只做下面几步。

## 1. 创建插件目录

```text
plugins/<name>/
  .zcode-plugin/plugin.json
  commands/            # 可选：斜杠命令，文件名即命令名
  skills/              # 可选：技能
  hooks/               # 可选：钩子
  README.md
```

`plugin.json` 最小内容：

```json
{
  "name": "<name>",
  "version": "0.1.0",
  "description": "一句话说明",
  "author": { "name": "ffffk2" },
  "commands": "./commands"
}
```

约束：

- 目录名、`plugin.json` 的 `name`、`marketplace.json` 条目的 `name` 三者必须一致。
- 只声明真实存在的资源；路径不能越出插件目录。
- 命令文件名只能是小写字母、数字、`_`、`-`、`:`，且不超过 64 个字符；子目录会成为 `/父:子` 形式的命令名。
- 命令 Markdown 的 frontmatter 是扁平解析，每个值写在一行；`$ARGUMENTS` / `$1` 用于取参数。

## 2. 注册到市场

在 `marketplace.json` 的 `plugins` 数组里追加条目：

```json
{
  "name": "<name>",
  "source": "./plugins/<name>",
  "version": "0.1.0",
  "description": "一句话说明",
  "displayName": "展示名",
  "displayName_i18n": { "zh-CN": "中文展示名" },
  "description_i18n": { "zh-CN": "中文说明" },
  "category": "developer-tools"
}
```

- `source` 必须是相对于仓库根目录、且位于仓库内部的路径。
- 版本号与 `plugin.json` 保持一致；更新插件时同步递增两者。
- 不要有重复的 `name` 或重复的 `source`。

## 3. 提交前自检

- [ ] `plugin.json` 与 `marketplace.json` 都是合法 JSON。
- [ ] 目录名 = 插件 manifest 的 `name` = 市场条目的 `name`。
- [ ] 两处 `version` 一致。
- [ ] `source` 指向真实存在的插件目录，且没有越出仓库。
- [ ] 代码和文档里没有遗留的 `TODO`、`FIXME`、`YOUR_API_KEY` 这类占位符（ZCode 校验会拒绝）。
- [ ] 没有提交凭据文件或真实密钥。

## 4. 提交与发布

```bash
git add plugins/<name> marketplace.json
git commit -m "feat(<name>): 新增插件"
git push
```

多个插件可以放在同一个提交里一起提交，也可以在提交信息里分别说明。

## 5. 本地安装与更新

1. ZCode 的 **Plugin Marketplace** 里刷新 `zcode-plugins` 市场。
2. 新插件会出现在列表中，直接 Install。
3. 已安装插件的版本更新后，使用更新操作即可；插件 ID 是 `<name>@zcode-plugins`。

## 版本约定

- 每个插件独立语义化版本（`0.1.0` 起步）。
- 只改文档或内部实现：`patch`；新增能力：`minor`；不兼容变更：`major`。
- 更新插件时同时修改 `plugin.json` 和 `marketplace.json` 里的版本号。
