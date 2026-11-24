# Obsidian 作为个人知识库与 Agent 长期记忆

研究与设计 · 2026-09-17 · 面向 Windows 本地工作流

> **推荐：Markdown 原始资料与共享知识笔记 + 少量稳定记忆 + 明确的更新协议 + 按任务检索。**
> 检索用 frontmatter 与路径收窄后跑 `rg`，不建索引（2026-11-24 裁决）；QMD / Basic Memory 截至本次未引入，是否升级需另立决定。不要先把所有知识拆成原子事实，也不要让所有更新都等待人工批准。

这是一份设计与实施规格，附可复制的示例和验收用例。2026-09-17 初稿生成时尚未连接本机核实、也未修改 vault；2026-11-24 已按 `D:\ObsidianRepo` 的落地情况回写修订（见下方实施状态）。文中目录、字段与验收阈值是设计建议；第三方能力附原始来源。

> **实施状态（2026-11-24）**：vault 已于 2026-09-17 部署，2026-10 迭代出 OpenCode 自动注入插件与园丁巡检。本设计不再单独定义现行机制：Codex 侧接入已撤销（§7.1）、operation executor 与单写入器不采用（§5.2、阶段 B）、索引与上下文预算不做（§6.4）、工具面收敛为「基础读写 + 结构化搜索规格」（§7.3）。阶段 C 的检索对比尚未执行，是否执行或收缩为失败台账仍待决定。现行口径以 `D:\ObsidianRepo\10-Projects\memory-system\decisions\` 与 `90-System/specs/` 为准；本文保留 2026-09-17 的研究与推理，修订处就地标注。

## 1. 研究结论：原方案方向可行，但有六处需要调整

| 原判断 | 核查与修正 | 设计影响 |
|---|---|---|
| Markdown 是唯一事实源，所以不需要数据库 | Markdown 可以是可编辑内容的权威存储；数据库可以只是可重建索引，两者并不冲突。存储权威也不代表内容一定正确 | 保留文件所有权；2026-11-24 裁决不建索引（§6.4），向量库另见 vault 路线图 |
| 一事实一文件是最佳实践 | 是 `obsidian-memory-for-ai` v3 的特定协议；Basic Memory 用实体笔记、段落观察和关系组织知识 | 默认按知识对象或问题组织，只有独立生命周期的记录才拆分 |
| AI 不应修改原始笔记 | 应区分原始证据、人类自写内容和授权 AI 维护的结论页；禁止所有修改会妨碍知识更新 | 原始证据留存；结论页允许受控修订；人工段落保留 |
| AI 写入都进入 Inbox，由人确认 | v3 协议本身也允许可信自动化应用操作；人工审核不是每次写入的必经步骤 | 按来源、冲突和维护权限分流，低风险更新直接执行 |
| 几千上万条笔记才需要向量检索 | 文件数不是可靠阈值；小库也可能存在跨语言、术语不一致和多来源综合问题 | 先补 `aliases` 与标题；检索组件升级未执行 |
| 根目录放 AGENTS.md 就有跨会话记忆 | 还需要 agent 能发现入口、实际检索、正确写入并在新会话验证 | 增加跨仓库接入与端到端验收 |

Eric J. Ma 的文章确实报告维护时间占比从 30–40% 降到不足 10%，也描述了月度日志、会议、人物、项目以及 `AGENTS.md`。但这是一位实践者的自述，不能当作普适收益或独立对照实验。他也会基于源材料更新人物与项目笔记，要求能追溯证据。[原始文章](https://ericmjl.github.io/blog/2026/3/6/mastering-personal-knowledge-management-with-obsidian-and-ai/)

`obsidian-memory-for-ai` 的 v3.1 确有原子事实、事件、操作提案、回执和协作声明；其规范明确把并发定位为少量 agent 的协作，声明文件不是事务锁。它是可借鉴的参考实现，现有资料不足以支持“最工程化”这种全市场排序。[项目说明](https://github.com/jrcruciani/obsidian-memory-for-ai) · [v3 规范](https://github.com/jrcruciani/obsidian-memory-for-ai/blob/main/SPEC-v3.md)

原调研中 MemPalace 的介绍需要降权：当前项目 README 主要描述原文保存、语义检索与可替换后端，默认使用 ChromaDB。宣传中的检索 R@5 不等于最终回答正确率，更不能证明不存在错误记忆。设计不依赖博客中的极短压缩指针或“近乎完美”的说法。[项目原始说明](https://github.com/MemPalace/mempalace)

## 2. 值得借鉴的路线与选型

本次围绕四条主线研究：Obsidian 原生能力；Markdown 记忆与知识编纂；检索和时间更新；agent 接入与验证。优先采用官方文档、原作者文章和项目仓库。搜索返回的候选、仓库自报指标与本地实测严格分开；本报告没有复现第三方性能榜单。

| 路线 / 项目 | 已核实的机制 | 适合你的部分 | 暂不采用的部分 |
|---|---|---|---|
| Obsidian 原生文件、CLI、Bases | 本地文件；CLI 控制应用；Bases 基于文件属性提供视图 | 主存储、人类编辑与浏览、链接维护、状态面板 | 不把视图当额外事实库 |
| Karpathy LLM Wiki | 原始资料、持续维护的 wiki、组织规则；摄入、查询、检查 | 把多次研究积累为可更新的主题结论 | 不假设 AI 维护近乎无成本、永不漏改 |
| Basic Memory | Markdown 实体、observations、relations；稳定 permalink；MCP 与本地索引 | 曾作为「少写基础设施」的候选 | 截至 2026-11-24 未安装；与自有 vault 会争夺同一内容的写入权 |
| Atomic Markdown Memory v3 | 一事实一文件、schema、操作信封与回执 | 来源、更新前置条件、可审计操作 | 初期不整体引入原子事实库与全部工具 |
| QMD | 本地 BM25、向量召回、RRF、重排；CLI/MCP | 曾作为「只缺搜索」的候选 | 截至 2026-11-24 未安装：实测小库 `rg` 足够快，索引只带来新鲜度成本 |
| Letta MemFS | Git 支持的文件记忆；常驻内容与按需读取分层 | 少量常驻信息 + 任务相关读取 | 无需为此迁移整个 agent runtime |
| Graphiti | 带来源、有效时间的动态知识图谱 | 时间语义与失效处理 | 暂不引入图数据库与图谱抽取流水线 |
| MemPalace | 原文会话留存与检索 | 若今后需要海量历史会话搜索，可独立试验 | 不作为知识结论的唯一权威 |
| Claudian | 在 Obsidian 内嵌 coding agents，以 vault 为工作目录 | 喜欢在 Obsidian 内对话时作为界面选项 | 不误认为装插件就获得记忆生命周期管理 |

来源：[CLI](https://help.obsidian.md/cli)、[Bases](https://help.obsidian.md/bases)、[LLM Wiki 原文](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)、[Basic Memory](https://github.com/basicmachines-co/basic-memory)、[QMD](https://github.com/tobi/qmd)、[Letta MemFS](https://docs.letta.com/concepts/memfs/index.md)、[Graphiti](https://github.com/getzep/graphiti)、[Claudian](https://github.com/YishenTu/claudian)。

**具体选择（2026-11-24 修订）：先实现可工作的文件协议；不建索引（用户裁决）。检索按 `90-System/specs/search.md` 用 frontmatter 与路径收窄后跑 `rg`，失败先靠标题与别名改进。QMD / Basic Memory 未安装，是否引入留待单独决定。**

这种选择基于你的本地 agent 工作方式和当前小库规模。它不是对所有用户的通用排名。尤其是“能存”“能搜”“能正确更新”属于不同能力，不能用同一个产品标签代替。

## 3. 内容模型：一套共享知识，明确不同内容的用途

### 3.1 知识库与记忆管理需要同时成立

知识库回答“某个概念是什么、有什么证据、研究进展如何”。长期记忆回答“我们之前做了什么、为什么、当前约束是什么、下次怎么继续”。原始会话保存不等于两者都完成。

| 内容 | 典型例子 | 权威位置 | 更新方式 |
|---|---|---|---|
| 原始证据 | 官方文档摘录、论文、实验结果、关键会话片段 | Sources 与代码/实验仓库 | 保留已捕获版本；新内容新建版本或补充 |
| 共享知识 | OTFS 某结构的分析、agent memory 对比、技术解释 | Knowledge | 带来源地修订，保留未解决分歧 |
| 项目状态 | 当前方案、约束、下一步、阻塞项 | 每个项目的 INDEX | 随事实变化更新；链接决策和执行证据 |
| 情景记忆 | 一次调试、一次实验、一次讨论的结果 | 项目 sessions | 一次事件一个独立记录，纠错另附记录 |
| 稳定个人记忆 | 输出偏好、工作环境、长期约束 | Memory | 来源明确；按作用域更新 |
| 程序性经验 | 故障识别条件、修复步骤、验证方法 | lessons；成熟后由既有 skill/repo 管理 | 经过验证再提升为可复用流程 |
| 导航和上下文包 | HOME、主题 MOC、当次检索结果 | 导航页与派生视图 | 可再生成；不是新的证据来源 |

**不要再建一份与 Knowledge 完整重复的 `facts/`。** 例如“Flow matching 的目标函数”属于知识页；“我研究该目标函数时采用过哪个实验设定”属于项目与事件；“技术解释应使用公式”属于偏好。三者互相链接，但不重复维护正文。

代码、依赖版本、实验配置、可执行 skill 的权威位置通常仍是相应 Git 仓库。Obsidian 保存解释、决策入口和明确日期/commit 的状态快照；不复制一套实时配置并期待永远一致。

### 3.2 建议目录

以下表格是目标组织方式，目录按内容需要逐步创建，不要求第一天填满。

> 2026-10-28 起，目录职责与边界的唯一权威文本是 vault `90-System/specs/vault-layout.md`（七桶职责、工具硬路径、30-Sources 触发条件、80-Archive 与 superseded 分界、Inbox 提案生命周期）。本表保留为设计原稿。

| 路径 | 内容与边界 |
|---|---|
| `AGENTS.md` | vault 使用协议；不放大量个人事实 |
| `HOME.md` | 少量主题与活跃项目入口，不列举全库每个文件 |
| `00-Inbox/` | 待归类资料、不确定推断、冲突提案 |
| `10-Projects/<project-id>/INDEX.md` | 项目目标、当前结论、约束、下一步、证据入口 |
| `10-Projects/<project-id>/decisions/` | 独立决策记录及替代关系 |
| `10-Projects/<project-id>/sessions/` | 按会话/事件独立保存的执行结果 |
| `20-Knowledge/<topic>/` | 概念、比较、研究综合、主题 MOC |
| `30-Sources/` | 外部资料与关键原始记录；保留 URL、版本和捕获时间 |
| `40-Memory/` | `profile.md`、`preferences.md`、`environments.md` 与按需新增的已验证经验（未单设 lessons 目录） |
| `80-Archive/` | 不再活跃但仍有历史价值的内容 |
| `90-System/` | 模板、schema、派生视图、操作回执；默认不参与普通知识检索 |
| `_assets/` | 图片、PDF 等附件；保留正文引用关系 |

检索数据库、模型缓存和临时锁放在 vault 外的本机目录，例如 `%LOCALAPPDATA%\ObsidianAgentMemory\<vault-id>\`；不通过文件同步工具共享正在写入的 SQLite 数据库。2026-11-24 裁决后这些目录均未使用。

人物页只有在确有持续协作需求时再增加。你以研究与开发为主，不必照抄管理十二人团队的 dossier 结构。PARA 可用于项目活跃状态，MOC 用于导航，Zettelkasten 的“独立可理解”用于知识页质量；无需同时强制执行三个完整体系。

### 3.3 何时拆成独立记录

默认“一份笔记回答一个稳定问题”，而不是“一句话一个文件”。出现以下情况才拆：有独立来源与有效时间；需要独立替代或审核；被许多页面单独引用；同页反复发生编辑冲突。

适合独立成页：一次架构决策、一条有作用域的长期约束、一种已验证故障模式。不适合机械拆分：同一个公式的每个符号、会议中每一句话、论文摘要的每个句子。

ID 与路径分离：笔记创建时获得不可变 UUID，标题与路径允许改变；正文链接仍使用 Obsidian wikilink。维护 ID 到路径的映射可由扫描重建。文件名建议清晰可读，避免 Windows 不允许的字符；不要把完整主张文本编码成路径主键。

## 4. Schema、来源与时间：让记录可验证

### 4.1 最小公共字段

不是每篇笔记都要填二十个字段。普通知识与项目页只要求以下六项：

| 字段 | 作用 |
|---|---|
| `id` | 不随移动、重命名变化的唯一标识 |
| `type` | `source / knowledge / project / decision / session / memory / moc` |
| `status` | `draft / active / conflicted / superseded / archived` |
| `created_at` | 创建时间，ISO 8601，带明确时区 |
| `updated_at` | 最近一次实质性更新；不是内容有效时间 |
| `sources` | 来源链接列表；可为空，但不能把空来源的推断当已确认事实 |

可选公共字段：`aliases`、`tags`、`scope`、`maintenance`。其中 `maintenance` 可为 `human`、`shared`、`agent`、`generated`。这表示约定的维护方式，不是操作系统权限或技术隔离。

**`scope` 语义（2026-10-28 裁决）**：只表示**内容归属**——产出/维护该记录的项目或归属实体（`memory-system`、`user-global`、`repo:<路径>` 等），**不表示适用范围**；适用范围由 `tags` 表达。

对 memory/decision 等需要判断真值与生效时间的记录，再增加：`evidence`、`valid_from`、`valid_to`、`recorded_at`、`supersedes`、`superseded_by`。`evidence` 采用 `explicit / observed / inferred / unverified`；不使用没有校准依据的“置信度 0.93”。

**`review_after`（2026-11-24）：保留在 schema 但生产记录零使用，实际复核由 `review.base` 的 `days_stale`（`updated_at` 超过 14 天）驱动。** `status` 枚举只有 `draft / active / conflicted / superseded / archived`，没有 `resolved`：冲突解决后保持 `active`，在正文补解决记录。

Obsidian 的 Properties 面板不完整支持嵌套属性；本设计因此用扁平字段与字符串列表，把复杂证据放正文。属性中的 wikilink 要加引号，同名属性在全库保持相同类型。[官方 Properties 文档](https://help.obsidian.md/properties)

### 4.2 每条重要结论都能回到具体证据

一篇研究页应包含：问题、适用条件、结论、推导或解释、证据、局限、相关知识、最近验证日期。混合多个来源时，在对应段落附来源，不能只在页末放五个 URL。

来源页至少记录：原 URL/仓库路径、标题、作者或机构、捕获时间、版本/commit（可得时）、关键段落定位、原文与摘录边界。论文保留 PDF 或稳定版本链接；公式与图表不能因文本提取而丢失其成立条件。

对于会话产生的记忆，要区分：用户明确表达、工具观察、assistant 建议、用户最终接受。assistant 曾经建议一个方案，不等于用户采用了这个方案。模型多次复述同一结论，也不会变成多个独立证据。

摘要和派生视图只能转引其上游证据，不能因为 AI 在下一轮又读到自己的摘要，就提高该结论的可信等级。这是防止错误在循环整理中逐渐“被坐实”的关键约束。

### 4.3 当前事实与历史事实分开处理

用两个问题理解时间：**什么时候生效？什么时候系统得知？** `valid_from/valid_to` 表示前者，`recorded_at` 和操作历史表示后者。Graphiti 的时间与来源设计值得借鉴，但本地文件只加几个字段并不会自动拥有完整双时态查询能力。[Graphiti](https://github.com/getzep/graphiti)

本方案约定有效区间为左闭右开 `[valid_from, valid_to)`，未知起始时间留空，不能用捕获日冒充真实生效日。替代记录保留旧版本；若需要回答“当时系统相信什么”，必须同时保存修订记录或可访问的版本历史，不能只看当前 YAML。

冲突检测先比较 `subject + predicate + scope`，再比较有效区间与基数约束。例如：

- “工作站有 RTX 5070”和“服务器有两张 RTX Pro 6000”不是冲突。
- 同一设备同时有两块 GPU 也不一定冲突，`installed_gpu` 是多值属性。
- “项目当前选用模型 A”与“同一项目当前唯一选用模型 B”才可能构成单值冲突。
- 后写入的旧资料不能仅因时间戳较新而覆盖较新事实。

发现候选冲突后，保留两条证据并设为 `conflicted`；明确的用户更正或可验证的新状态可以解决冲突。文档过了 `review_after` 只表示需要复核，不自动变成错误或删除。

## 5. 写入协议：把自动化用于确定的工作

### 5.1 写入权限按内容区分

| 操作 | 默认处理 | 条件 |
|---|---|---|
| 新增有来源的 session / 执行结果 | 自动写入 | 记录实际完成与未完成；不需要 operation ID |
| 更新 agent 维护的项目“当前状态” | 自动应用并留 diff | 来源可核实、无冲突、写入前目标文件 mtime 未被他方改动 |
| 保存用户明确要求记住的偏好 | 自动应用并留来源 | 不扩大原表达的作用域 |
| 新建有来源的知识草稿 | 自动创建 | 推断与原文分开，默认 draft |
| 在已授权知识页补充已核实信息 | 自动小范围修改 | 只改相关段落，不覆盖人工论证 |
| 从一次行为推断长期偏好 | 进入 Inbox | 标注 inferred，默认不用于个性化决策 |
| 相互矛盾的事实或大范围重构 | 生成差异与处理提案 | 先完成可审阅结果，再集中处理不确定项 |
| 改写原始证据 | 保存新版本/纠错记录 | 不伪造原始表达或静默覆写原记录 |
| 生成 MOC、任务上下文包、Bases 视图 | 自动重建 | 内容来自已有权威文件 |

这里的自动化是实施目标。落地时把它固化为 vault 协议（`AGENTS.md`）与校验脚本；`AGENTS.md` 单独存在不能构成强制访问控制，也不提供并发事务。

### 5.2 单次更新的执行流程

1. 判断是否值得保存：以后能复用、与项目连续性有关，或用户明确要求。临时寒暄、重复摘要、未验证猜测不升级为稳定记忆。
2. 找已有实体、笔记和证据，优先补充已有对象，避免同义文件分裂。
3. 确定目标 `id/path`、来源、修改理由与影响范围。
4. 校验结构、路径、来源、作用域、有效时间；可机械检查的部分交给 `validate_memory.py` / `vault-health.mjs`。
5. 写入前查看目标文件 mtime：30 分钟内被其他会话写过就避让，不代改；人工或同步修改后重新读取并调整差异，不强行覆盖。
6. 单文件用 Read/Write/Edit 完整改写；涉及多文件时逐个落盘，中断后按实际文件状态续做，不假定整批已生效。
7. 写回执（项目事件或 `global-events`）、更新受影响索引、报告实际完成与未完成。

**2026-11-24 裁决：不实现 operation executor、单写入器、operation ID 与 hash 前置条件。** 单 agent 场景下先读全文、完整重写再加 `git` 回滚已覆盖主要风险；并行会话用「mtime 避让」作为正式替代方案（见 vault `AGENTS.md`「写入」）。代价是不提供原子事务：多个写者同时改写同一文件仍可能竞态，出现时以 git 历史恢复并保留双方内容。需要严格并发事务时再引入写入服务或数据库，Markdown 继续作为内容层。

不让多个 agent 同时 append 一个总日志：项目事件用单个 `logs/project-events.md`，非项目事件用 `90-System/logs/global-events/<date>-<slug>.md` 独立文件。

“事件只追加”是普通编辑约定，不妨碍修正错误或执行明确的删除需求。撤回错误事件时追加纠错记录；需要删除内容时同时修正引用它的 wikilink 与 INDEX 指针。无索引与缓存（§6.4），不存在「已删记忆从向量库复活」的路径。

### 5.3 捕获、巩固与遗忘

| 时机 | 动作 |
|---|---|
| 用户做出决定、给出纠正、实验完成 | 当场保存关键增量，不等到会话关闭 |
| 一个阶段结束 | 保存进度、失败尝试、证据和下一步 |
| 有可用会话结束/压缩前 hook | 补充保存；作为增强，不作为唯一保障 |
| 待处理项累积或定期检查 | 去重、合并、解决冲突、更新链接和过期提示 |
| 项目结束 | 项目归档；可复用结论保留在 Knowledge，旧操作细节降权 |

压缩前钩子不存在或触发失败时，系统仍应通过阶段性保存恢复。保存失败必须可见，不能在回复里说“记住了”却没有成功回执。

默认先保留关键片段和定位信息，按需求选择是否存完整 transcript。完整会话归档是独立能力，不应挤占日常检索。保留期限依据资料用途设置；不要用“永久保留一切”替代筛选。

## 6. 检索设计：按任务选择证据，而不是总装载全库

Anthropic 的 context engineering 文档支持逐步获取相关上下文、外部笔记与有限上下文预算；Letta MemFS 则展示常驻文件与按需读取文件的分层。这些是可借鉴机制，不意味着某个固定 token 数对所有模型都最佳。[Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Letta](https://docs.letta.com/concepts/memfs/index.md)

### 6.1 建议读取顺序

1. **判断是否需要个人/项目历史。** 普通知识问答不必先读全量个人记忆。
2. **定位作用域。** 从用户指定项目、当前 repo 和 HOME 得到候选主题；不确定时扩大搜索。
3. **精确读取入口。** 当前项目 INDEX、明确相关的偏好、对应主题 MOC。
4. **检索候选。** 先按 `type`/`status`/`scope`/`tag` 与目录收窄范围，再跑 `rg`（规格见 vault `90-System/specs/search.md`）；Obsidian search 作为人工兜底，不引入 BM25/向量召回。
5. **读取匹配段落与证据。** 对需要解释原因的问题，跟随决策和 source 链接；图扩展默认限制一至两跳。
6. **判断时效与冲突。** 当前问题优先 active；历史问题按询问时间读取旧记录；冲突不能因排名低而隐藏。
7. **组成当次上下文包。** 保留结论、来源、作用域、有效时间和未解决项，去掉重复摘要。

MOC 是加速入口，不是唯一通路。搜索不到时要跨主题回退；不能因为索引漏了一页就认定知识不存在。全文搜索命中也只是候选，不能跳过内容阅读。

### 6.2 对不同问题采用不同路径

| 用户问题 | 优先内容 | 必须避免 |
|---|---|---|
| “继续上次项目” | 项目 INDEX + 最近 session + 活跃 decision | 只读最后一条聊天摘要 |
| “为什么当时没用方案 A？” | decision + 原始讨论/实验记录 | 用现在的结论倒推过去理由 |
| “我现在的设备配置是什么？” | 有 scope 的环境记录 + 最近有效证据 | 把不同设备合并成一个实体 |
| “我们以前怎么修复过这个错误？” | `40-Memory` 已验证经验 + 症状/版本 + 验证结果 | 套用已失效版本的 workaround |
| “这个研究方向目前怎么看？” | 主题综合 + 新来源 + 未解决分歧 | 返回旧摘要却不说明日期 |
| “上个月采用什么方案？” | 事件、旧 decision、有效区间 | 无条件过滤所有 superseded 页面 |
| “这是我之前说过的吗？” | 原会话/引文定位 | 仅凭 assistant 总结确认用户原话 |

### 6.3 中文、英文和符号混合要专门测试

你的语料同时包含中文、`KV cache`、`键值缓存`、OTFS、代码标识符和公式，检索必须覆盖三种情况：中文子串、跨语言同义表达、精确版本/错误码。

`rg` 对中文按字面子串匹配，跨语言同义（“键值缓存” ↔ `KV cache`）只能靠 `aliases` 与标题补足——这是当前最主要的短板，实测 151 篇 Markdown（不含模板/specs）中仅 2 篇有 `aliases`。因此检索改进的优先级是：先补 `aliases` 与可读标题，再补失败台账；分词与索引不讨论（2026-11-24 裁决：索引不做）。

用固定三条自检即可：中文子串（“键值缓存”）、跨语言（别名命中 `KV cache`）、精确错误码/版本号（不能返回相似错误）。任何一次真实搜索失败都记入失败台账，见阶段 C。

### 6.4 索引与上下文预算（2026-11-24 裁决：不做）

本节整体不实施：不建可重建索引（`document_id / content_hash / indexed_at / section_anchor`），不做混合检索与 RRF 重排，不做 LLM 摘要缓存，不设 token 预算数值。理由：vault 实测 162 篇 Markdown / 1.7 MB，全库 `rg` 单关键词约 33 ms，索引换不来速度，却引入新鲜度、删除残留与 hash 失效问题（对应验收用例 25/26）。

唯一保留的尺寸契约是插件对项目 INDEX 的 **6000 字符截断**（`vault-health.mjs` 同时按 5400 / 6000 告警）。这是运行时的真实限制，不属于本节的预算建议。没有索引后，「索引过期、删除后从缓存复活」整类问题消失；`vault-health` 的源漂移检查是 mtime 比对，不是索引，保留。

### 6.5 什么时候升级（2026-11-24 现状：未执行，条件保留）

| 实际问题 | 下一步 |
|---|---|
| 精确名称就能找到，读取几页足够 | 保持文件搜索，改进标题、别名和入口 |
| 明明存过，换个说法或语言就搜不到 | 先补 `aliases` / 标题并记录失败；补完仍反复失败再评估检索组件 |
| 多个客户端需要一致的 search/read/write 接口 | 读写继续用基础文件工具；若确需统一契约，另立决定（§7.3 不加工具） |
| 大量依赖关系和时间关联问题人工遍历困难 | 先改索引页与事件拆分；仍不够再评估图索引或 Graphiti |
| 多机器同时写、事务一致性成为主要故障 | 改为协调写入服务/数据库；Markdown 继续作为内容或导出层 |

截至 2026-11-24，QMD / Basic Memory 未安装、未对比，本节升级路径保持待定。既有 vault 决策「记忆系统工具面路线图」把向量检索列为不做（重访条件「无」），但未逐项评估 QMD 的词法检索部分；若将来引入，需先明确该不做清单是否只覆盖向量部分。当前可先改标题、`aliases`、标签或 `search.md` 的收窄规则。

## 7. Windows 与本地 Agent 接入

### 7.1 两种工作场景都要覆盖

**在 vault 中整理资料：** agent 的工作目录直接设为 `D:\ObsidianRepo`，让它读取本地协议与 HOME。

**在代码仓库中工作：** 当前目录仍是项目 repo；全局指令只放简短的记忆入口和触发条件，按需读取 vault 的项目笔记。不要为了记忆而把所有 coding 会话都改到 vault 根运行。

> **2026-11-24 修订**：实际只接入 OpenCode。Codex 侧接入（全局 AGENTS 记忆块 + SessionStart hook）已于 2026-10-10 由用户裁决撤销，Codex 使用原生 memories；`C:\Users\g00943418\.codex\hooks.json` 与 hook 脚本均已改名 `.bak`。OpenCode 侧则从「全局入口 + 按需读取」进一步演化为插件自动注入：会话开始与每次压缩后注入 vault `AGENTS.md` + `HOME.md` + 匹配项目的 INDEX（截断 6000 字符），并注册 `memory.status` / `memory.inject` 两个只读诊断工具。机制与已知限制见 vault `90-System/integrations/opencode/obsidian-memory/README.md`。

Codex 官方指令发现规则区分全局和当前项目路径；任意旁边目录的 `AGENTS.md` 不会仅因存在就成为全局记忆入口。[Codex 官方说明](https://developers.openai.com/codex/guides/agents-md/)

OpenCode 同样有作用域规则，而且 V1 与 V2 文档存在差异：本次读取到的 V2 文档明确称 `instructions` 配置虽被 schema 接受，但尚不会解析引用文件。部署时必须确认版本；不要复制 V1 配置后只看“不报错”就认定加载成功。[V1 规则](https://opencode.ai/docs/rules/) · [V2 指令](https://opencode.ai/v2/docs/instructions)

交付包中的 `global-entry.example.md` 是入口文字示例；把它合并进实际 harness 支持的全局指令，不覆盖已有规则。它要求的读取动作需在一个全新会话中实际观察验证。

### 7.2 接口选择

| 方式 | 使用位置 | 约束 |
|---|---|---|
| 直接文件访问 + rg | 批量整理、通用 agent 读写、无人值守任务 | 自行处理 YAML、链接、编码、冲突 |
| Obsidian CLI | 原生链接操作、Bases 查询、vault 检查 | 应用需运行；首次调用可启动应用 |
| MCP | 多个 agent 使用统一工具契约 | 工具协议不自动提供记忆治理或事务 |
| Claudian | 在 Obsidian 内交互 | 是工作界面；底层权限、模型、协议仍需配置 |

> 2026-11-24 裁决：实际只用「直接文件访问 + rg」；Obsidian CLI 仅作人工兜底，MCP 与 Claudian 均未采用。

官方 CLI 支持显式 vault 与 path。自动化不要依赖“当前打开的笔记”，避免写错目标。以下是只读诊断示例；vault 名需以本机注册名称为准：[CLI 参考](https://help.obsidian.md/cli)

```powershell
obsidian version
obsidian vault="ObsidianRepo" read path="HOME.md"
obsidian vault="ObsidianRepo" search query="OTFS" format=json
obsidian vault="ObsidianRepo" unresolved format=json
```

移动/重命名优先用 Obsidian 的 move/rename，并启用自动更新内部链接。直接文件重命名时不能假定其他笔记引用已经修复。官方 CLI skill 可以帮助 agent 使用 Obsidian，但它不包含你这套记忆策略。[Obsidian skills 仓库](https://github.com/kepano/obsidian-skills)

编码统一 UTF-8，自动生成文件使用一致换行；备份与同步分别设计。Git 保存可审查版本，独立备份负责磁盘或误删恢复；云同步冲突文件不能交给 LLM 静默挑一份。若未来使用 WSL，先验证 Windows 路径映射、文件监听、锁和换行再共享写入，第一阶段不强制引入 WSL。

### 7.3 对 agent 暴露的最小能力（2026-11-24 裁决）

**读写记忆不新增工具**：用基础 Read/Write/Edit（vault 内即文件读写）+ vault 协议完成。**只有搜索值得设计**，且交付物是规格而不是服务/CLI/MCP。

| 能力 | 实现方式 | 不做什么 |
|---|---|---|
| 读记忆 | Read 直接读文件；先入口与 INDEX，再正文 | 不做 `recall` 工具、不做上下文包生成器 |
| 写记忆 | Write/Edit + 协议（`AGENTS.md`）+ 提交 | 不做 `record` / `update` 工具、不做 operation executor |
| 搜索 | 按 `90-System/specs/search.md`：frontmatter + 路径收窄，再 `rg`；结果附路径与行号 | 不做守护进程、索引、MCP 搜索服务、向量库 |
| 整理 | 园丁会话（Layer 2）按巡检结果判断性修复 | 不做 `consolidate` 工具 |
| 体检 | `vault-health.mjs` + `validate_memory.py` 只读 CLI | 不做 `doctor` 工具 |

搜索规格的要素（详见 vault `90-System/specs/search.md`）：

- 六个过滤条件：`scope`、`type`、`status`、`tag`、`updated_after`、`text`，全部映射到目录或 frontmatter，不建索引。
- 固定回退顺序：项目 INDEX / HOME → 结构收窄后的 `rg` → 全库 `rg` → Obsidian CLI search（人工兜底）。
- 固定返回：`绝对路径 + 行号 + type/status/scope/updated_at + 命中行`；命中只是候选，结论必须读正文。
- 失败台账：每次真实未命中记录原问题、期望文件、实际候选与失败类型，用于决定补 `aliases` 还是改规格。

现状：规格已落盘至 vault `90-System/specs/search.md`（2026-11-24），由 `AGENTS.md`「读取」节指向；`memory.status` / `memory.inject` 是 OpenCode 插件诊断用的既有工具，不属于通用记忆 API。

### 7.4 数据与指令的边界

抓取网页、论文、源代码注释和历史聊天都是资料，不具有修改 agent 行为规则的权限。只有你维护的明确规则入口提供行为协议；来源页中的“忽略之前指令”等文字应作为引文处理。

本地文件存储不意味着模型推理也在本地。使用远程 provider 时，实际读取并注入上下文的内容可能进入请求；因此按任务选取最小相关资料。不同业务权限的资料最好从读取范围上隔离，而不是只依靠一个标签期待模型遵守。

## 8. 研究与工程工作流示例

### 8.1 论文或技术调研进入知识库

一次摄入形成三种产物：source 页保留版本与证据位置；knowledge 页记录自己的理解、推导、适用边界和对照；若影响当前项目，再更新 decision 或提出变更草稿。不要把外部作者主张直接写成“已经证实的工程结论”。

例如读取一篇 OTFS 信道预测论文，应保留信道模型、速度范围、输入/输出形状、训练和评测设置、对照基线。模型结构相似但实验设置不同，不应直接合并成一个性能结论。知识页的“我们是否采用”只链接项目决策。

### 8.2 故障修复形成可复用经验

一次 provider/tool-calling 故障记录：出现版本、错误信息、最小触发条件、排除过的原因、确认根因、修复动作、验证命令、结果与剩余限制。该记录进入 session；只有复用条件清晰、验证成立，才生成 lesson。

lesson 不是“以后永远这样改”的全局规则。它要声明版本与适用条件。若最后沉淀为 skill，Obsidian 链接该 skill 的权威仓库与版本；不复制一份相互漂移的执行脚本。

### 8.3 “继续上次工作”的新会话

入口定位项目 → 读取当前状态 → 读取上次的未完成项 → 检查 repo 实际状态是否已变化 → 按相关决策继续执行。若笔记说测试通过，但工作树或 commit 已变化，应重新验证相关条件；不能把记忆当成现场状态。

会话收尾保存：本次新增证据、完成项、失败/未完成项、下一步、相关文件或 commit。仅“讨论了 X，进行了深入分析”这类摘要不满足恢复工作的需要。

## 9. 分阶段实施与验收

### 阶段 A：跑通最小闭环

**状态（2026-11-24）：已完成。** 最小目录、HOME、vault 协议、模板、schema、项目 INDEX 于 2026-09-17 部署；跨目录读取、决定保存与历史查询已验收（`10-Projects/memory-system/sessions/2026-09-17 memory-system 验收.md`）。

保留现有两篇笔记，先归属或建立链接，不批量重写。增加 HOME、vault 规则、一个真实项目 INDEX、一份偏好页、Sources 与 session 模板。配置从任意代码目录可发现的入口，显式写下可自动更新的范围。

用三个新会话完成测试：记录一个明确决定；关闭后询问该决定与原因；再更正决定并查询旧时间的状态。验收通过后才扩大摄入量。

### 阶段 B：把可靠性做实（2026-11-24 落地状态）

目标：让「反复写入」出错时能被机械发现、能被回退，而不是靠模型自觉。设计列了七项，落地取舍如下。

| 设计项 | 处置 | 落地位置 |
|---|---|---|
| lint | 已做 | `90-System/ops/validate_memory.py`（schema、uuid、时间、替代关系、证据约束）+ `vault-health.mjs`（13 项巡检） |
| 稳定 ID | 已做 | 每条记录创建时分配 UUID，改名保留 ID |
| Bases 面板 | 已做 | `90-System/views/review.base`（待处理、活跃项目、陈旧复核、INDEX 超限） |
| operation ID | **不采用** | 单 agent 场景无重试执行器，收益为零 |
| hash 前置条件 | **不采用**，改 mtime | 写入前查目标文件 mtime，30 分钟内被他人改过就避让 |
| 单写入器 | **不采用**，改 mtime 避让 | 并行会话各自写不同文件；同文件冲突用 git 恢复 |
| 操作回执 | 降级为协议 | 项目事件 `logs/project-events.md`、非项目 `global-events/`；机器不可解析 |

结论：B 的可靠性由「lint + UUID + Bases + mtime 避让 + git」承担；不实现事务型写入执行器。代价是同一文件被两个会话同时改写时可能丢失一方修改，需要靠 git 历史恢复并手工合并——出现频率上升到影响使用，再引入写入服务。

### 阶段 C：检索验证（2026-11-24：未执行，待定形态）

原设计：积累 30 个真实问题，与 QMD / Basic Memory 对比中文改写、英文别名、代码标识符、多会话、历史更新、无答案、归档与索引失效。

**状态：2026-11-24 未执行**（vault 无 30 题集、无 Recall@5 或任何对照记录）。可先不预造题，用下面的失败台账积累证据；是否保留完整 30 题对比由用户决定。§6.4 已明确不建索引，若做对比只能评估 QMD / Basic Memory 这类外部组件。

失败台账的最小字段：

| 字段 | 内容 |
|---|---|
| 原问题 | 用户当时的问法（保留原话） |
| 期望文件 | 事后确认应该命中的笔记 |
| 实际候选 | 当时 `rg` 返回的前几条 |
| 失败类型 | 换说法 / 跨语言 / 只记得结论 / 多跳 / 精确标识符 / 时间过滤错误 |
| 处置 | 补 `aliases`、改标题、补标签、改 `search.md` 规则 |

台账条目写入对应项目的 `logs/project-events.md` 或 `global-events`，不新建专门目录。判据：同一失败类型出现 3 次以上再改规则，单次失败只补别名。

### 阶段 D：扩大自动维护

对已证明可靠的目录自动巩固和整理；只把有冲突、缺证据、影响面大的变更交由你决定。维护触发可以先手动，稳定后接入实际 harness 支持的 hook 或任务调度；本次设计没有创建任何定时任务。

**状态（2026-11-24）：已落地，但形态与本文不同。** 实现在 OpenCode 插件内：`gardener.ts` 双触发器（定时 30 min + 会话事件防抖 5 min）→ `vault-health` 门控（error>0 / warn≥5 / 距上次全量≥7 天）→ LLM 冷却 6h → 调度 `vault-gardener` 会话；2026-10-08 由 dry-run 翻闸实跑，2026-10-24 并回 obsidian-memory 主插件；spawn 窗口限定本地 [1,8)。插件只触发 agent 会话、绝不直接写 vault，「不做后台写入 daemon」仍成立。机制见 vault `10-Projects/memory-system/decisions/园丁自动化决策.md`。

### 验收不只测“能否搜到”（2026-11-24 收敛）

LongMemEval 把信息提取、多会话推理、知识更新、时间推理和无法回答时的克制列为核心能力，适合借鉴到小型个人验收集；不必直接把整个研究 benchmark 当上线门槛。[原项目](https://github.com/xiaowu0162/LongMemEval)

| 维度 | 测量方式 | 首轮建议目标，非已测结果 |
|---|---|---|
| 证据召回 | 真实问题是否命中期望文件；多证据题另看证据集是否完整 | 不设 Recall@5 目标；按失败台账逐条判断 |
| 最终回答 | 正确性、时间、作用域与来源是否同时正确 | 关键决定/偏好/否定题零错误，其余逐项报告 |
| 更新与冲突 | 多条矛盾证据、明确更正、并行会话写同一文件 | 不静默覆盖冲突；同文件竞态可用 git 恢复 |
| 来源 | 有效主张能否回到原文或观测 | 稳定记忆 100% 可追溯；缺来源保留标识 |
| 索引一致性 | 不适用（无索引）；改名后检查 wikilink 与 INDEX 指针 | 无失效指针冒充现行内容 |
| 成本与延迟 | 记录检索耗时与读取量 | 已测：1.7 MB / 162 篇下单关键词约 33 ms；不设 token 预算 |
| 维护负担 | Inbox 积压、错误修正次数、每周人工时间 | 连续两周观察是否实际减少重复说明 |

不预设题集，也不把一次验收当统计结论。表现差时分别检查捕获、知识组织、检索、时间判断和回答生成，避免一律归因于模型不够强。

## 10. 交给本地 agent 的实施边界

第一批交付应只有：现状盘点与备份方案、最小目录与入口、公共 schema、少量模板、一个真实项目的迁移示例、跨目录读取验证，以及验收记录。**2026-11-24 追加边界：不写记忆 CLI、不装搜索服务、不建索引；搜索只交付 `90-System/specs/search.md` 规格（已交付）。**

设计包本身仍是协议示例和验收材料，不安装插件、不带后台任务、不导入个人资料、不假装实现自动锁、索引或 hooks；真实部署在 vault 侧独立完成（OpenCode 注入插件与园丁已上线）。示例中的新路径、ID 和字段需要本地 agent 按实际文件适配。

**最终推荐的核心是：让 Obsidian 保持一个你也愿意阅读的知识系统，同时让 agent 对“来源、作用域、时间、更新和检索”负起明确责任。** 目录只是承载它的形式；判断成功的标准，是你换会话、换模型、换项目之后，仍能得到有证据且没有过期的上下文。

## 来源与证据等级

查阅日期均为 2026-09-17。在线页面会变化；本地部署应固定工具版本并复验。以下列出实际用于结论的来源，而非候选搜索结果数量。

| 来源 | 等级与用途 |
|---|---|
| [Eric J. Ma 实践](https://ericmjl.github.io/blog/2026/3/6/mastering-personal-knowledge-management-with-obsidian-and-ai/) | 原作者实践；收益为自报 |
| [Obsidian CLI](https://help.obsidian.md/cli) | 官方能力与命令 |
| [Properties](https://help.obsidian.md/properties) | 官方字段与 YAML 限制 |
| [Bases](https://help.obsidian.md/bases) | 官方视图与存储关系 |
| [Bases syntax](https://obsidian.md/help/bases/syntax) | 官方视图格式 |
| [Obsidian skills](https://github.com/kepano/obsidian-skills) | 原始工具使用技能仓库 |
| [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) | 原作者设计理念，不是受控效果实验 |
| [Atomic Memory README](https://github.com/jrcruciani/obsidian-memory-for-ai) | 原项目结构与定位 |
| [Atomic Memory SPEC](https://github.com/jrcruciani/obsidian-memory-for-ai/blob/main/SPEC-v3.md) | 协议边界与并发限制 |
| [Basic Memory](https://github.com/basicmachines-co/basic-memory) | 原项目能力；未在本机安装复现 |
| [Basic Memory format](https://docs.basicmemory.com/concepts/knowledge-format) | 官方内容、observations、permalink 设计 |
| [QMD](https://github.com/tobi/qmd) | 原项目检索架构；未复现性能 |
| [QMD CJK issue](https://github.com/tobi/qmd/issues/617) | 具体历史问题，当前已关闭，不推断仍未修复 |
| [Letta MemFS](https://docs.letta.com/concepts/memfs/index.md) | 官方记忆分层与版本管理 |
| [Graphiti](https://github.com/getzep/graphiti) | 原项目时间、来源与图谱设计 |
| [MemPalace](https://github.com/MemPalace/mempalace) | 原项目机制；自报指标未独立复现 |
| [Claudian](https://github.com/YishenTu/claudian) | 原项目支持范围，避免混淆同名 fork |
| [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md/) | 官方指令发现规则 |
| [OpenCode V1](https://opencode.ai/docs/rules/) | 官方 V1 规则，不混用 V2 |
| [OpenCode V2](https://opencode.ai/v2/docs/instructions) | 官方 V2 版本差异 |
| [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 官方工程经验 |
| [LongMemEval](https://github.com/xiaowu0162/LongMemEval) | 原评测项目的能力划分 |
