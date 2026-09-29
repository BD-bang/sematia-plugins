# SemaTIA Official Plugins

SemaTIA 官方插件市场。在 SemaTIA 设置 → 插件 → 添加市场输入 `BD-bang/sematia-plugins`,
或对 agent 说「安装 st-toolkit 插件」。

## 插件

| 插件 | 内容 |
|---|---|
| st-toolkit | st-review 技能(ST 评审清单)+ /review-st、/gen-watch-table 命令 |

## 市场格式

根目录 `marketplace.json` 声明插件清单;每个插件目录可含 `skills/<名>/SKILL.md`、
`commands/*.md`、`agents/*.md`、`.mcp.json`。
