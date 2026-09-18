# Obsidian 记忆与知识库设计包

这是设计规格与可复制示例，不是已部署的软件。研究日期：2026-09-17。

先阅读 `Obsidian-memory-design.md`，再由本地 agent 根据当前 vault 与 harness 版本增量实施。不要将本包整体解压覆盖已有 vault。

文件说明：

- `AGENTS.example.md`：vault 内的规则示例，部署时与已有规则合并。
- `global-entry.example.md`：在其他代码仓库工作时使用的全局记忆入口示例。
- `templates/`：frontmatter 和正文模板；双大括号字段必须替换。
- `specs/note.schema.json`：公共 frontmatter 的 JSON Schema；只覆盖可机械校验的部分。
- `specs/semantic-checks.md`：来源、链接、时间、基数和写入约束。
- `specs/acceptance-cases.md`：30 个建议验收用例。
- `examples/review.base`：Bases 视图示例；只展示结构化字段，不等同自动审核。
- `examples/update-operation.yaml`：操作信封示例，未附执行器。

本包不包含自动部署脚本、MCP server、记忆写入器、索引服务或定时任务；也未复制用户历史个人信息。

交付检查已核对 Markdown/HTML 结构、模板必填字段以及 JSON/YAML 可解析性。未在你的 Obsidian、Windows 或 agent 运行时执行；Bases 表达式与自动化行为需要本地验收。HTML 不依赖外部资源。

推荐实施顺序：备份与盘点 → 一个项目试点 → 全局入口 → 三会话恢复/更正测试 → 确定性校验 → 按实际遗漏试点检索增强。

实现时另外交付本机环境信息、实际修改 diff、通过/失败的验收记录。模板字段或能力未实现时应明确说明，不要宣称已经自动执行。
