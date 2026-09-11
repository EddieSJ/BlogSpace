---
title: "Hermes Agent 记忆机制深度解析：Memory、Skills 与上下文压缩的全链路设计"
date: 2026-09-11
tags: ["Hermes", "AI Agent", "Memory", "Skills", "上下文压缩"]
summary: "拆解 Hermes Agent 的四层记忆体系——MEMORY/USER PROFILE 双层持久记忆、Skills 程序性记忆、会话历史 FTS5 检索，以及 frozen snapshot、字符预算治理、渐进式加载与 [SKILL_PRUNED] 幽灵技能防御背后的设计取舍。"
author: "EddieSJ"
---

# Hermes Agent 记忆机制深度解析：Memory、Skills 与上下文压缩的全链路设计

你让一个 Agent 修了三天的 bug，第四天你问它「上次我们是怎么解决那个并发问题的」，它却一脸茫然地从头开始猜。这不是它笨，而是它根本没把那段经历存下来——或者说，存下来了，却不知道什么时候该取出来。

记忆之于 Agent，从来不是「存不存得下」的问题，而是「存什么、存哪、什么时候读、读进来要花多少 token」的权衡。Hermes Agent 把这套权衡拆成了一个显式的四层体系：常驻的事实（MEMORY）、用户的画像（USER PROFILE）、按需加载的技能（Skills）、按需检索的历史（session search）。这篇文章只拆一件事——**这四层各自怎么设计、为什么这么设计、怎么配合成一条完整链路**。

## 一、四层记忆体系：一张总览表

单个上下文窗口装不下「我是谁、用户是谁、我会什么、我们聊过什么」这四件事。Hermes 的取舍一句话就能说清：**常驻的东西必须小，大的东西必须按需加载**。由此分出四层：

| 层 | 载体 | 存什么 | 什么时候进上下文 | 成本特征 |
|---|---|---|---|---|
| 第一层 | MEMORY.md | Agent 自身笔记：环境事实、约定、踩过的坑 | 会话启动即注入 | 每轮都付 token，硬上限 2200 字符 |
| 第二层 | USER.md | 用户画像：姓名/角色/时区、偏好、雷区 | 会话启动即注入 | 硬上限 1375 字符 |
| 第三层 | Skills | 程序性记忆：怎么做某类任务 | 按需 `skill_view` 加载 | 默认只载入索引，正文用多少付多少 |
| 第四层 | 会话历史 | 全部消息（SQLite + FTS5） | 按需 `session_search` 检索 | 无 LLM 调用，几乎零 token |

这张表是全文的骨架，后面每一章展开其中一行。

## 二、双层持久记忆：MEMORY.md 与 USER.md

内置记忆是两层文件，职责边界写得明明白白。`MEMORY.md` 是 Agent 自己的笔记，`USER.md` 是它对你这个用户的画像。源码里连提示块标题都是写死的：

```python
MEMORY_BLOCK_HEADERS = {
    "memory": "MEMORY (your personal notes)", "user": "USER PROFILE (who the user is)"}
```

**为什么重要：** 分两层而不是揉成一个文件，是因为这两类信息的生命周期和隐私边界不同。用户画像属于「关于用户的事实」，Agent 不该随意覆盖；自身笔记属于「Agent 的经验」，随任务积累。分开才能各自设预算、各自门控。

预算才是这层记忆真正的设计核心。默认值在 `config_defaults.py` 里写死：

```python
"memory_char_limit": 2200,   # ~800 tokens at 2.75 chars/token
"user_char_limit": 1375,     # ~500 tokens at 2.75 chars/token
```

而且计量口径很「抠」：计数是 `len(ENTRY_DELIMITER.join(entries))`，`ENTRY_DELIMITER = "\n§\n"`——连条目之间的 `§` 分隔符本身都要算进预算。这 2200 / 1375 字符约合 800 / 500 token，**记忆这一层每轮都稳定占用约 1300 token**。

**为什么重要：** 因为记忆每一轮都在上下文里，它的体积直接是每轮 prompt 的固定开销。预算硬性有界，是为了让「每轮固定开销」可控、可预期——这是把记忆做成常驻能力的前提。

## 三、记忆如何进入上下文：frozen snapshot 与 prefix cache

记忆不是每轮重新渲染的，而是**会话启动时渲染成一份冻结快照**，注入系统提示。模块 docstring 说得很直白：

```python
Both enter the system prompt as a FROZEN snapshot at session start;
mid-session writes hit disk but never change the prompt (prefix cache intact).
```

渲染格式固定：46 个 `═` 分隔线 + 标题 + 使用率 + 条目，你在上下文里会看到类似 `MEMORY (your personal notes) [11% — 258/2,200 chars]` 的块头。

**为什么重要：** 冻结快照的理由是保护 LLM 的 **prefix cache**。如果每轮记忆内容一变，系统提示就跟着变，那整段前缀的 KV cache 全部失效，每一轮都要重新计算。冻结之后，会话中途你 `add` 一条记忆，它**立刻写盘**，但**整个会话期间系统提示不变**，直到下次启动才生效——性能优先于「即时可见」。

这套注入发生在系统提示组装管线的 volatile 分区：`volatile_parts = [skills_prompt, *_memory_parts(agent)]`，记忆块与 skills 索引同属 volatile 层。理解这点，后面讲 Skills 和压缩时就能串起来了。

> 一点措辞澄清：工具描述里写「Memory is injected into every future turn」，指的其实是「每轮都在上下文里可见」，代码事实是**会话启动组装一次**。

## 四、写不进去的时候：预算、整合与防死循环

预算满了会怎样？**报错，而不是静默丢弃。** 超限时工具返回一段明确指令，并附上当前全部条目和用量：

```python
if len(ENTRY_DELIMITER.join(entries + [content])) > limit:
    return self._failure_with_entries(target, (
        f"Memory at {self._char_count(target):,}/{limit:,} chars. Adding this entry "
        f"({len(content)} chars) would exceed the limit. Consolidate now: use 'replace' to merge "
        f"overlapping entries into shorter ones or 'remove' stale or less important entries ..."))
```

**为什么重要：** 设计者把「满预算」当成了一个**强制整合的信号**，而不是一个可以跳过的错误。官方文档明确：Memory 不会自动压缩，满时返回 error 让你做「提高信息密度」的决策——合并重叠条目、删掉过期条目——而不是放弃写入。因为一旦允许静默丢弃，Agent 就会退化成无状态。

配套有两个工程细节值得单独说：

第一，**批量 `operations` 数组只在最终状态检查预算**。源码注释原文是 `budget check against the FINAL state only`。所以一次调用可以先 `remove`/`replace` 腾出空间、再 `add` 新条目——单独 `add` 会溢出，放进同一个批量就不会。批次是全有或全无的。

第二，**防死循环保护**：每轮最多允许 3 次合并失败（`_MAX_CONSOLIDATION_FAILURES_PER_TURN = 3`），超过就返回终态结果，让模型停止重试。设计意图注释写得很清楚：`a failed memory side effect must never block the turn's reply`——一个失败的记忆副作用，绝不能卡住这一轮的回复。

## 五、什么时候写、什么时候不写

记忆不是垃圾桶。四条写入通道的分工，是这套体系里最容易踩坑的地方：

- **跨会话恒真的事实** → 写 memory（`MEMORY.md` / `USER.md`）
- **一周内就过期的事实** → 留会话历史（session search）
- **流程 / 工作流** → 写进 skill，用到才加载
- **无条件的规则 / 人格** → 放 SOUL.md（身份层，每会话无条件）

系统提示里有一句可直接引用的规则原文：

> Anything learned while doing a task (procedures, pitfalls, and the user's preferences and corrections for that kind of work) belongs in the task's skill via skill_manage, where it loads only when relevant; memory is injected into every turn and must stay small.

还有一条写法规则特别有意思：**写成陈述句，不要写成命令句**。`'User prefers concise responses'`（对）vs `'Always respond concisely'`（错）——因为命令式措辞在之后的会话里会被模型当成一条指令重读，可能反过来覆盖用户当前的真实请求。这是把「记忆会被当成上下文读」这件事想透了之后的产物。

**为什么重要：** 放错层级的结果不是丢失，而是污染——一条本该按需加载的工作流被塞进每轮必读的 MEMORY.md，平白吃掉 token 预算。

## 六、Skills：把「怎么做」下移为程序性记忆

Skills 是第三层，也是这套体系里最 token-efficient 的设计。一个 skill 就是一个 `SKILL.md`（YAML frontmatter + 正文），可挂 `references/`、`templates/`、`assets/`、`scripts/` 支撑文件。关键是它的**渐进式加载**：

- **Level 0**：`skills_list()` 只返回 `{name, description, category}` 的索引（约 3k token 量级）
- **Level 1**：`skill_view(name)` 载入完整正文
- **Level 2**：`skill_view(name, path)` 只载入某个具体 reference 文件

本机实测印证了这点：researcher profile 的 skills 索引快照（`.skills_prompt_snapshot.json`）里，87 个 skill 条目只有 name / category / description 等元数据，**没有正文**。也就是说，系统提示里常驻的永远是「索引」，正文要等模型判断「现在用得上」才 `skill_view` 拉进来。

索引里那条 description 是唯一的触发器，而它被硬性截断成 57 个字符：

```python
SKILL_PROMPT_DESC_LIMIT = 60
def extract_skill_description(frontmatter: Dict[str, Any]) -> str:
    desc = _normalize_skill_description(frontmatter)
    return desc[:SKILL_PROMPT_DESC_LIMIT - 3] + "..." if len(desc) > SKILL_PROMPT_DESC_LIMIT else desc
```

`60` 减去 3 个省略号，正好是 57 + `"..."`。官方规范要求 description 的首 57 字符必须是一个自足的触发器：`Use when <trigger>. <one-line behavior>.`

**为什么重要：** 这是整个体系 token 哲学的集中体现。大量「怎么做」的知识被下移为按需加载，常驻上下文里只留一行 57 字符的触发器。写 skill 的人如果没把触发条件放进前 57 字符，这个 skill 在模型眼里就等于不存在。

## 七、上下文压缩与幽灵技能防御

长会话会把上下文撑爆，压缩是必须的。Hermes 有双压缩层：Gateway 层的会话卫生在 85% 阈值触发（安全网），Agent 内的 ContextCompressor 在 50% 阈值触发（主力，按真实 API token 算）。压缩默认开启，`threshold: 0.50`，`tail_mode: lean` 会保留约 2.5% 窗口的尾部（10K–25K token 夹逼），连续性改由摘要 + 机械提取的锚点索引 + `session_search` 恢复指针来承接。

压缩对记忆体系的直接影响，是一个叫「幽灵技能」（ghost-skill）的陷阱：压缩把旧的 `skill_view` 结果降级成一行摘要时，模型**还以为技能已经加载**，但正文其实已经不在上下文里了，于是凭对技能的幻觉记忆行动。解决办法是给每个被裁剪的技能留一个标准标记：

```python
SKILL_PRUNED_MARKER_PREFIX = "[SKILL_PRUNED:"
def _skill_pruned_marker(skill_name: str) -> str:
    return (f"{SKILL_PRUNED_MARKER_PREFIX} content lost in compression; "
            f"reload with skill_view(name='{skill_name}')]")
```

系统提示里还有一条写死的契约：看到 `[SKILL_PRUNED]` 就要重新 `skill_view` 加载，且重新加载后，同一技能的旧 marker 是历史残留、应忽略。如果压缩摘要把 marker 也弄丢了，系统会**确定性地回注**一个 `## Pruned Skills` 块补齐，确保模型一定能看到重载提示。

**为什么重要：** 记忆体系不是「存一次就永远有效」的。压缩会主动删减已加载的东西，`[SKILL_PRUNED]` 是删减与「模型以为自己还知道」之间的安全网。诚实说明：压缩阈值与 marker 机制来自源码与官方文档，属「代码可证实、运行时未触发」——本机 `state.db` 未观测到压缩事件。

## 八、会话历史：FTS5 检索及其边界

第四层是「我们聊过什么」。Hermes 把全部消息落进每个 profile 自己的 `state.db`，用 SQLite 的 FTS5 全文检索索引，`session_search` 直接查 DB、返回真实消息——**无 LLM 调用、无摘要**；只有超长的单条消息会在展示层截断（read 视图 1200 字符、scroll 视图 4000 字符），并附带 `content_truncated` 与 `original_content_chars` 标记。discovery 查询数十毫秒级。

但它有一道明确的覆盖边界，源码里写得很直白：

```python
# Hidden from browsing/searching — integrations (HERMES_SESSION_SOURCE=tool), delegate
# subagent runs, kanban workers are not the user's history.
_HIDDEN_SESSION_SOURCES = ("kanban", "subagent", "tool")
_DEMOTED_SESSION_SOURCES = ("cron",)
```

kanban worker、subagent、tool 集成产生的会话**根本不会被搜索返回**——因为它们不是「用户的历史」；cron 会话可搜但被降权排在交互会话之后，防止 cron 的词汇靠 BM25 霸榜。本机实证：researcher profile 的 `state.db` 里确实存在 3 条 `source='kanban'` 的会话，而 `session_search` 不会返回它们。

**为什么重要：** 会话检索带着「谁的历史」判断，不是无差别全文搜索。一个 kanban worker 搜自己的历史命中是 0，这是设计而非 bug。容量「无限」，但时效受 `auto_prune` 约束：默认开，**已结束**且超过 90 天无活动的会话会在启动时被清理（活跃、置顶、mid-turn 的会话永不删除）。

## 九、落盘位置与数据模型

记忆和会话最终都落在磁盘上。目录结构是 `$HERMES_HOME/memories/{MEMORY.md, USER.md}`，每个 profile 一份独立的 `state.db`。真实 schema 里 `messages` 表的关键列如下（本机 schema 节选，`...` 处为省略列）：

```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,
    tool_name TEXT,
    effect_disposition TEXT,
    timestamp REAL NOT NULL,
    token_count INTEGER,
    ...                          -- finish_reason / reasoning* / codex_* / display_* 等列从略
    active INTEGER NOT NULL DEFAULT 1,
    compacted INTEGER NOT NULL DEFAULT 0,
    api_content TEXT,
    display_kind TEXT,
    "_compressed_summary" INTEGER NOT NULL DEFAULT 0,
    ...);
```

FTS 侧也不是单一索引：`messages_fts` 全量索引（但 `role='tool'` 的内容截断到 8192 字符），`messages_fts_trigram` 只索引「非 tool、非 cron」的消息，另有可选的 CJK 扩展索引。

**为什么重要：** 数据模型暴露了工程取舍——`tool` 结果体积大、密度低，全量索引也要截断；cron 会话会污染 trigram 索引，直接排除。诚实标注：本机 `sessions.system_prompt` 列在 97 条会话中**全部为空**，因此「记忆块写进了历史会话系统提示」只能靠代码路径 + 当前会话实际含 MEMORY 块论证，DB 无法直接佐证。

## 十、多 profile 隔离：每个 Agent 自己的记忆

Hermes 用 profile 来做记忆隔离——每个 profile 是独立的 Hermes home，有独立的 `config.yaml`、`.env`、memories、sessions、skills、cron jobs、`state.db`。官方文档里有一条硬性警告，几乎可以当作部署铁律：

> Never point two agent processes at the same profile (the same Hermes home). Both write memory automatically, and each loads the other's writes into its system prompt at session start — so two writers on one home compound each other's state until it stops being anything you configured.

**为什么重要：** 记忆是自动写入的，没有「这个文件归我、那个文件归你」的仲裁。两个进程共用同一个 home，等于两个人往同一本笔记本里无意识地互写。profile 存在的目的就是防止这种污染；而需要共享记忆的多个 Agent，官方给的答案是改用 external memory provider（内置支持 Honcho、Mem0 等 8 个插件，叠加不替换内置记忆）。

本机实测能直观感受到这种隔离：default 的 `MEMORY.md` 有 2907 字节的微信网关、博客流水线等 6 条大条目，researcher 只有 1 条 358 字节的清单——同一套 Hermes，六个 profile 六种记忆。

## 十一、记忆 × 自动化：cron 与 Kanban

这套记忆体系还和两个自动化场景深度耦合。

cron 是 **per-profile** 设计：一个 job 属于某个 profile，执行时用那个 profile 的环境、`.env`、skills 和记忆。但 cron 会话之间**没有跨次记忆**——`Cron jobs run in isolated sessions with no memory of previous runs`，需要上一个 job 的输出时用 `context_from` 前置注入。注意区分：这里说的是会话历史不继承；profile 级的 `MEMORY.md` 仍属于同一个 profile 的其他运行。

kanban swarm 的 worker 则以 **assignee profile 的身份**启动（dispatcher 执行 `hermes -p <assignee>`），因此 worker 自动携带的是**那个 profile** 的记忆与 skills，而不是 orchestrator 的。这解释了为什么一个 writer 角色的 worker，其记忆里装的是「写作」相关的约定——因为它是带着 writer profile 的记忆被拉起来的。

**为什么重要：** 多 Agent 编排里，「谁的记忆被加载」由「进程挂载在哪个 profile」决定。这一步错配会让 worker 带着完全错误的上下文干活。

## 十二、设计哲学与失败模式

把全文串起来，这套记忆体系其实是同一条设计主线的三处体现：**常驻的必须小（预算 2200/1375）→ 大块知识下移为按需加载（Skills 的 Level 0/1/2）→ 更旧的历史下移为按需检索（session_search）**，而冻结快照优先服务于 prefix cache。

它同时也把「记忆膨胀会怎样失败」想得很透，对应的治理机制可列成一张表：

| 失败模式 | 治理机制 |
|---|---|
| 无界增长撑爆 prompt | 硬字符上限，超限报错不静默丢 |
| replace 换更长内容再次溢出 | replace 同受上限约束，需缩短或先删 |
| 模型在满预算上反复重试耗尽 turn | 每轮最多 3 次合并失败，之后返回终态 |
| 合并批次误删最后一条导致清空 | batch 拒绝把非空 store 删空 |
| 外部手工编辑造成文件漂移被覆盖 | 漂移检测 + `.bak` 快照 + 拒写 |
| 记忆条目被注入污染并跨会话存活 | 写入严格扫描 + 加载时 `[BLOCKED: ...]` 占位 |
| 无人值守 review 擅自删记忆 | background review 禁用 replace/remove |
| 压缩后模型以为技能还在 | `[SKILL_PRUNED: ...]` marker + 确定性回注 |

这张表是全文最有「工程感」的部分——不是「理想情况下应该怎样」，而是「每一类失败都有人预想过，并写好了对应机制」。对生产环境工程师，最后的评估清单：记忆预算 2200/1375 是否够用、是否需要 `write_approval` 防错误假设写入画像、是否需要 external provider（内置是单机策展式记忆，不是知识图谱）、retention 90 天是否符合可追溯性要求、多 Agent 是否做到「一 profile 一进程」、长任务是否依赖 skill 正文常驻（那会遇到 `[SKILL_PRUNED]`）。

记忆系统做得好不好，不看你存了多少，而看你能不能回答那三个问题：**存什么、什么时候读、读进来花多少 token**。Hermes 的答案，就是这一套有界的、分层的、为缓存和成本精打细算过的四层设计。
