---
title: "Hermes 底层核心机制剖析（面向 AI 开发者）"
date: 2026-08-31
tags: ["Hermes", "AI Agent", "LLM", "架构设计"]
author: "EddieSJ"
---

# Hermes 底层核心机制剖析（面向 AI 开发者）

> 一个生产级 agent 框架内部到底是怎么转起来的？本文不评测"好不好用"，而是把 Hermes Agent 拆开，看它的循环、系统提示组装、工具、记忆、技能、编排与安全各自是怎么设计、又为什么这么设计。

## 引言：先看一张全景图

如果你用过 Claude Code 或 Codex，会熟悉"对话式编程 agent"的外在体验：你说一句话，它调用工具、看结果、再调用工具，直到把活干完。Hermes Agent（Nous Research 开源的个人 AI Agent 框架）属于同一类，但它的特别之处在于：同一套内核同时驱动 CLI、Ink TUI、Electron 桌面端、Web 仪表盘、ACP（IDE 集成）与 20+ 消息平台网关（Telegram/Discord/Slack/WhatsApp/iMessage 等），并自带跨会话记忆、技能（程序性记忆）、多代理编排（委派/Cron/Kanban/Webhook）与原生 MCP 客户端。

全文的叙事顺序即拆解顺序：**循环 → 系统提示 → 工具 → 状态 → 记忆/技能 → 编排 → 安全 → 多面**。先记住一句结论，后文反复印证它：

> Hermes 的底层没有魔法——一个被严格执行的 agent 循环（工具结果即上下文），加上把"缓存与角色交替"当架构硬不变量、把"持久化/记忆/技能/编排/安全"做成独立子系统的设计。

## 1. Agent 循环：一切能力的底座

一切从 `run_agent.py` 里的 `AIAgent.run_conversation()` 开始。整个框架的核心逻辑，用伪代码表达只有十几行：

```python
def run_conversation(self):
    system_prompt = build_system_prompt()          # 见 §2
    messages = [{"role": "system", "content": system_prompt}, ...]
    while len(self.iterations) < max_turns:        # agent.max_turns，默认 90
        response = self.llm.chat(messages, tool_schemas)
        if response.tool_calls:
            for call in response.tool_calls:
                result = handle_function_call(call)          # 分派工具
                messages.append(tool_result_message(result))  # 结果回灌
            continue
        return response.text                          # 纯文本 → 结束
```

**为什么这个循环是"底座"**：agent 的一切能力——读文件、跑命令、搜网页、写代码——都不是一次性完成的，而是"调用工具 → 把结果放回对话 → 让模型基于新信息决定下一步"的反复。这里有三个容易被忽略、却决定框架成色的设计。

**第一，工具结果不是"副作用"，是上下文。** 每次工具调用的输出都会被序列化成一条消息，追加回对话历史，供模型下一轮读取。这正是 agent 能"看到"自己执行结果的机制。也因此，所有工具 handler 必须返回 JSON 字符串——统一这条回灌管道的格式。这条约束被写成硬性要求（"All handlers must return JSON strings"），因为一旦某个工具返回了非序列化内容，整条循环就断了。

**第二，循环必须有上限。** `agent.max_turns`（默认 90）兜底，防止失控的工具循环把会话无限拖下去。接近 token 上限时，还会自动触发上下文压缩（见 §2）。

**第三，也是最重要的——两个硬不变量。** 它们贯穿整个框架：

1. **绝不破坏 prompt caching。** 会话中途不得改动历史上下文、工具集或系统提示，唯一例外是上下文压缩。现代 LLM 的 prompt caching 能让长对话成本降一个数量级；一旦中途改动历史，缓存全部失效。缓存是成本与延迟的生命线，不是可选的优化项。
2. **消息角色严格交替。** 不能出现连续两条 assistant 或两条 user 消息，只有 tool 结果可以连续重复。这是 OpenAI 兼容 API 的硬约束，也决定了所有"往会话里塞消息"的路径——out-of-band 用户消息、cron 投递、委派回灌——都必须遵守这条协议纪律，否则 API 直接报错。

## 2. 系统提示组装：每一轮开始前发生了什么

"底层机制"的第一站其实不是循环，而是**每一轮开始前，系统提示里到底塞了什么**。这由 `agent/prompt_builder.py` 负责，按固定顺序组装：

1. **身份**：`SOUL.md`（在 `$HERMES_HOME` 下）定义 agent 人格，独立于项目规则，始终加载——系统提示的 slot #1。
2. **记忆注入**：跨会话记忆 + 用户画像（见 §5）。
3. **技能注入**：与当前任务相关的技能正文按需加载进上下文（见 §6）。
4. **项目上下文文件**：见下。
5. **环境提示**：`build_environment_hints()` 注入 OS、`$HOME`、cwd、终端后端等事实。

项目上下文文件这块藏了最多工程判断。Hermes 支持 `.hermes.md`/`HERMES.md`、`AGENTS.md`、`CLAUDE.md`、`.cursorrules` 等几种约定文件，但加载优先级是 **first match wins**（每会话只加载一个来源）：

- `.hermes.md` / `HERMES.md`：向上遍历父目录直到 **git root**——root 就是边界。这样家目录里的 `.hermes.md` 不会泄漏进你所有项目。
- `AGENTS.md` / `CLAUDE.md` / `.cursorrules`：**仅当前目录（cwd）**。为什么？因为它们本意是跨 agent 工具（Claude Code、Cursor 等）可移植的约定，不应被向上继承污染。

两个保护性设计值得单独说：每个上下文文件上限 **20,000 字符**，超出做 head+tail 截断（中间丢弃，留 `[...truncated...]` 标记）；所有上下文文件过**威胁模式扫描器**，命中 prompt injection / promptware 的内容被替换成 `[BLOCKED: ...]` 占位符——**阻断内容，而不是阻断文件**，其余内容照常加载。

环境提示里还有个反直觉的细节：**当终端是远程后端（docker/ssh/modal 等）时，抑制主机信息**。原则是"提示词绝不能描述 agent 够不到的主机"——否则模型会基于一个不存在的环境做出错误判断。排查"是配置问题还是框架问题"时，`hermes --ignore-rules` 能一次性跳过上述所有注入，作为隔离手段。

最后是**上下文压缩**：`compression.enabled` 配合 `threshold`（默认 0.50，上下文用到 50% 触发）与 `target_ratio`（默认 0.20，压到 20%）。压缩由 auxiliary 模型（独立的小模型体系，见 §8）执行，是**唯一**允许改动历史上下文的例外——呼应 §1 的缓存不变量：压缩是两害相权取其轻。

## 3. 工具系统：注册表、工具集与 MCP

工具系统的设计可以浓缩成一句话：**注册 ≠ 暴露**。

**注册表**：`tools/registry.py`。任何 `tools/*.py` 在顶层调用 `registry.register(...)` 就会被自动发现导入。每个工具四个要素——`name`、`toolset`、`schema`（OpenAI 工具 schema）、`handler`——外加两个门控字段 `check_fn` 和 `requires_env`。

**check_fn 门控**：工具只有在"需求满足"（比如 API key 存在）时才出现在模型面前。为什么重要？否则每个会话都要背着一堆不可用的工具 schema，既费 token，又诱导模型调用一个注定失败的工具。

**工具集（toolsets）**：工具注册了还不行——**只有名字出现在某个工具集里，agent 才看得到它**。`TOOLSETS` 字典定义了全部工具集（web/browser/terminal/file/code_execution/coding/vision/image_gen/tts/skills/memory/session_search/delegation/cronjob/kanban 等），`_HERMES_CORE_TOOLS` 是多数平台继承的默认包。工具变更 `/reset` 后生效——**绝不在会话中途改工具集**，这仍是缓存不变量的体现。

新增一个工具是两文件式流程：在 `tools/your_tool.py` 里 `registry.register(...)`，再在 `toolsets.py` 里把它加进目标工具集。路径一律用 `get_hermes_home()`，禁止硬编码 `~/.hermes`——这是 Profile 隔离的安全要求。

**原生 MCP 客户端**是工具系统里最值得借鉴的一块。它不是把 MCP 桥接成 CLI，而是 agent 内置的一等公民：启动时 `discover_mcp_tools()` 连接、`list_tools()` 发现并注册，且幂等（重复调用只连未连的服务器）。支持 stdio（`command`+`args`）和 HTTP/StreamableHTTP（`url`+`headers`）两种传输，配置在 `config.yaml` 的 `mcp_servers` 下：

```yaml
mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
  my-api:
    url: "https://mcp.example.com/mcp"
    headers:
      Authorization: "Bearer ${API_KEY}"
```

命名约定 `mcp_{server}_{tool}`（如 `mcp_filesystem_read_file`），命名空间隔离防冲突。每个服务器一个长驻 asyncio 任务；断线时**指数退避自动重连**（最多 5 次，序列 1s/2s/4s/8s/16s，cap 60s）。

MCP 的安全三件套尤其值得做 agent 的人照抄：

- **子进程环境白名单**：stdio 子进程只继承白名单环境变量（PATH/HOME/USER/LANG/LC_ALL/TERM/SHELL/TMPDIR/XDG_*），API key 默认不透传，除非显式 `env:` 声明——防凭据泄漏给不可信 MCP 服务。
- **错误消息凭据剥离**：工具错误消息里的凭据模式（`ghp_`、`sk-`、`token=`、`key=` 等）自动剥离后再给 LLM。
- **采样（sampling/createMessage）**：MCP 服务可反向请求 LLM 补全，默认开启；但 `max_tool_rounds`（默认 5）防止工具死循环。不可信服务可 `sampling: {enabled: false}` 整体关闭。

## 4. 状态与会话持久化

agent 的价值一半在"下次还记得"。Hermes 的持久化是**一切皆可审计**：

- **SQLite + FTS5**：`~/.hermes/state.db` 是规范会话存储，`hermes sessions list|browse|export|prune` 直接读它；`~/.hermes/sessions/` 存网关路由索引、请求 dump 与 `*.jsonl` 转写。
- **三种续接方式**：resume（`--resume ID` / `--continue`）、branch（`/branch` 会话分支）、handoff（`/handoff` 把 CLI 会话交接给消息平台继续）。
- **文件系统检查点**：`checkpoints.enabled` + `max_snapshots`（默认 50），`/rollback` 列出并恢复文件改动——这是 agent 写坏代码前的"后悔药"。
- **配置布局纪律**：设置进 `config.yaml`、密钥只进 `.env`（`hermes config set` 自动路由到正确文件）；`$HERMES_HOME` / Profile 使多实例完全隔离。

## 5. 记忆系统：陈述性记忆的分工

Hermes 把"记忆"这个模糊概念拆成了三类，各司其职（这是最值得 AI 开发者借鉴的设计之一）：

| 类型 | 内容 | 载体 | 注入方式 |
|------|------|------|----------|
| 程序性记忆 | 可执行步骤/流程 | 技能（SKILL.md） | 按需加载 |
| 陈述性记忆 | 事实/偏好/教训 | `user` + `memory` 双 store | 每轮注入 |
| 会话历史 | 原文对话 | SQLite+FTS5 | 按需检索（session_search） |

其中陈述性记忆（`memory` 工具）有两个 store：`user`（用户是谁、偏好、纠正）与 `memory`（环境事实、工具怪癖、经验教训）。写操作支持**原子批处理**（operations 数组一次提交）与字符预算检查（超限拒绝，需先删旧再加新）。

两条纪律值得写进任何 agent 的文档：一是**高信号过滤**——只存"能减少未来用户重复纠正"的事实，任务进度、PR 号、提交 SHA 这类临时状态不存（该走 session_search）；二是**声明式而非指令式**——写"用户偏好简洁"而不是"你必须简洁"，因为指令式措辞会在后续会话被当成命令重复执行。写入可配 `memory.write_approval` 人工审批。

## 6. 技能系统：让 agent 自改进

技能 = 一个 `SKILL.md`（YAML frontmatter + 正文），可带 `references/` `templates/` `scripts/` 附属文件。它是 agent 的"程序性记忆"，也是自改进的机制：

```yaml
---
name: python-debugpy
description: 调试 Python：pdb REPL + debugpy 远程 (DAP)。
version: 1.0.0
---
```

一个常被忽略的细节：**description 前 57 字符就是触发索引**。技能的 description 会被截断进系统提示的技能清单，触发条件必须自包含在那 57 字符内——否则"相关技能"扫描可能漏加载，技能就白写了。

生命周期由 `skill_manage` 工具管理，一条纪律贯穿始终：**用后即补**——任务与某技能相关就 `skill_view` 加载并遵循，用后发现问题就 `skill_manage(action='patch')` 立即修补。技能不维护就变成负债。

后台的 **Curator** 负责归档：`~/.hermes/skills/.usage.json` 记录每个技能 use/view/patch 次数与最后活跃时间，闲置标 stale 后归档。三条策略很克制：**只动 `created_by: "agent"` 的技能**（内置/商店/pin 技能豁免）；**永不删除**（最大破坏动作是 archive）；归档前自动 tar.gz 备份。合并整理（把重叠技能并成伞技能）默认关闭，需显式开 `curator.consolidate: true`。

## 7. 多代理与后台系统：agent 之上的协作层

单个 agent 循环之外，Hermes 还提供一套"让 agent 之间协作、让工作活过进程"的后台系统。理解它们的关键是抓住一条主线：**进程内委派 vs 持久调度**。

**delegate_task（进程内，轻量）**：子代理分 leaf（默认，不能再委派）和 orchestrator（可再派生，受 `max_spawn_depth` 约束）两种，批量并行上限 `max_concurrent_children`（默认 3）。两个心智模型最重要：一是子代理**完全隔离**——独立上下文、独立终端，对父会话一无所知，context 必须自包含；二是**委派是进程内的、不持久的**——父进程退出，后台子代理即失。要活过进程的工作，必须用下面三套持久化工具。另外记住：子代理的"自报"≠事实，对外副作用（上传、发布）必须拿到可验证句柄（URL/ID/路径）并亲自核验。

**Cron（持久调度器）**：schedule 四形态——时长（`30m`）、自然语言（`every monday 9am`）、5 段 cron（`0 9 * * *`）、ISO 一次性时间戳。几个机制很精巧：`no_agent=True` 时脚本即任务，stdout 原样投递、**空输出=静默**（看门狗模式）；`monitor_script`/`monitor_url` 输出哈希不变则整轮跳过（零 token）；每轮 **3 分钟硬中断**兜底；`.tick.lock` 跨进程防重复 tick。投递时带 header/footer 帧而非镜像进目标会话——**为保持角色交替协议**（呼应 §1）。

**Kanban（跨 profile 的 SQLite 工作队列）**：这是 Hermes 最"重"的协作层。worker 进程被 `HERMES_KANBAN_TASK` 钉在具体任务上，只看到 `kanban_*` 子集工具——硬隔离边界。Dispatcher（默认跑在网关内）原子 claim、promote ready 任务、回收超时 claim、按 assignee 派生对应 Profile；连续 spawn 失败 `failure_limit`（默认 2）次自动 block。核心思想是**依赖图即执行模型**：`parents=[...]` 门控，全部 parent 完成才 ready；orchestrator 只做分解路由，不亲自实现。

**Webhook（外部事件 → agent 运行）**：订阅即一条 prompt 模板，支持 `{dot.notation}` 取嵌套字段，HMAC-SHA256 签名校验（每订阅独立 secret）。窄化事件流靠两层机制：声明式 `filters`（equals/contains/regex 等运算符 + all/any/not 分组，不匹配返回 200 忽略）和 `--script` 路由脚本（stdin 收 JSON、stdout 替换 payload）。`--deliver-only` 让模板渲染结果**直接投递、零 LLM 成本**——外部服务推送、告警转发、agent 间 ping 的正确姿势。

## 8. 提供商无关与凭据工程

Hermes 用 **35+ provider 插件**（`plugins/model-providers/`，用户同名插件覆盖内置）+ OAuth/API key 双认证实现提供商无关。三层工程：

- **凭据池**：同一 provider 多个凭据自动轮换，跳过已耗尽/失效的 key（`hermes auth` 管理）。
- **fallback 链**：主 provider 失败时的降级链（`hermes fallback add`）。
- **模型别名**：`resolve_alias()` 先查用户 `model_aliases` 再查内置表（sonnet/opus/gpt5/codex/gemini/deepseek/grok/llama/qwen 等 21 个），用户表优先。

还有个**auxiliary 模型**体系：vision、压缩、session_search 等辅助任务用独立小模型，`auto` 回退到 OpenRouter 或 Google——把昂贵的主模型留给对话本体。

## 9. 安全与隐私：默认安全

安全设计的哲学是**默认安全，且防自关**：

- **密钥脱敏**（`security.redact_secrets`，默认开）：所有工具输出进上下文/日志前扫描 API key/token 模式。关键是**导入时快照**——会话中途改配置不生效，这是故意设计，防止模型在任务中自己把开关翻回去。
- **PII 脱敏**（`privacy.redact_pii`，默认关）：网关层哈希用户 ID、剥离手机号。
- **危险命令审批**（`approvals.mode` 三档）：`smart`（默认，aux LLM 评估：低风险自动放行、高风险拒绝、不确定才问人）/ `manual`（全问）/ `off`（全跳）。`--yolo` 只跳过审批，**不影响脱敏**（两者独立）。
- **注入防线**：项目文件 [BLOCKED] 扫描（§2）+ MCP 子进程环境白名单 + 错误凭据剥离（§3）。

一个常见误区：要"重置权限"时，正确姿势是清 `command_allowlist` 与 shell-hooks allowlist，**不是**开 yolo——脱敏和审批是两套独立机制。

## 10. 多面架构：一套内核，多个表面

```
   CLI ──────┐
   TUI ──────┤                              ┌──────────────┐
   Desktop ──┤        agent 核心            │   gateway    │
  Dashboard ─┤    （循环/工具/记忆/技能）    │  → 20+ 平台   │
    ACP ─────┤     run_agent + 各子系统     │    适配器     │
  hermes proxy┘                             └──────────────┘
```

同一核心驱动所有表面，消息平台网关适配器在 `plugins/platforms/`，消息进来走同一 agent 核心、带全部工具（不只是聊天）。两个亮点：所有斜杠命令从单一注册表（`hermes_cli/commands.py` 的 `COMMAND_REGISTRY`）派生，CLI/TUI/Telegram 菜单/自动补全同源——加一个命令处处生效；`hermes proxy` 是 OAuth 背书的本地 OpenAI 兼容 API，Codex/Aider/Cline 免 key 直连。

## 11. 给 AI 开发者的设计模式清单

如果只带走八条，是这些：

1. **循环即核心，工具结果即上下文**：一切能力围绕"结果回灌"设计（handler 返回 JSON 字符串）。
2. **注册表 + 工具集双层门控**：注册≠暴露；check_fn 按环境裁剪工具面，省 token 且防误用。
3. **一切皆可审计**：SQLite+FTS5 会话、JSONL 转写、`*.lock` 防重入、usage.json 遥测。
4. **三类记忆分工**：程序性（技能）/ 陈述性（memory）/ 可检索历史（session_search），各司其职。
5. **进程内委派 vs 持久调度分层**：轻量委派随进程消亡，必须活下来的工作用 Cron/Kanban/Webhook。
6. **缓存不变量优先**：性能与成本是架构约束，不是可选的优化项。
7. **默认安全 + 防自关**：脱敏默认开且导入时快照、注入扫描、子进程环境白名单。
8. **配置/密钥分离 + Profile 隔离**：设置与凭据分文件，多实例完全隔离，路径统一走 `get_hermes_home()`。

## 12. 结语

Hermes 的底层没有魔法：一个被严格执行的 agent 循环，配上把"持久化、记忆、技能、编排、安全"做实、做进系统提示组装管线和独立子系统的设计。对想自建 agent 框架的开发者，最有价值的三件事是：把工具结果当上下文的第一性循环、把缓存与角色交替当架构不变量、把记忆与技能做进系统提示组装管线。剩下的，都是这三件事的展开。

---

参考资料：

- Hermes Agent 官方文档：https://hermes-agent.nousresearch.com/docs/
- 配置指南：https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- CLI 命令参考：https://hermes-agent.nousresearch.com/docs/reference/cli-commands
- 提供商集成：https://hermes-agent.nousresearch.com/docs/integrations/providers
- 开发者指南：https://hermes-agent.nousresearch.com/docs/developer-guide/
- 仓库源码布局：https://github.com/NousResearch/hermes-agent
