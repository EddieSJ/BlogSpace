---
title: Hermes Agent 归档机制深度解析：技能归档、会话归档与任务归档的完整生命周期
date: 2026-09-11
slug: hermes-agent-archival-lifecycle
tags: [Hermes, AI Agent, Archive, Curator, Kanban, Session, 归档]
summary: 拆解 Hermes Agent 在技能、任务、会话、运行时四个层面的归档设计——Curator 的 active/stale/archived 三态生命周期、kanban 的 archived 唯一不可逆终态、state.db 的 archived/hidden 双标记、上下文压缩的 [SKILL_PRUNED] 占位，以及贯穿始终的「归档优先于删除」哲学。
---

# Hermes Agent 归档机制深度解析：技能归档、会话归档与任务归档的完整生命周期

凌晨两点，你负责的那个长期运行的 Hermes 实例磁盘告警了。你打开 `state.db`，里面躺着十几万条消息，体积早就过了 8GB。你第一反应是「清一清」，但删哪条？删错了还找得回来吗？再往前翻，你可能还踩过另外两个坑：技能目录被某个自动任务动过手脚，一个常用的 skill 突然不见了；或者一个任务明明标了 done，后来被 reopen，但订阅它的人从此再没收到过通知。

这三件事表面各不相干，底层其实是同一个问题：一个会自己写状态、自己长记忆、自己派生任务的 Agent，它的状态到底该怎么「过期」。Hermes 给出的答案是一句话——**归档优先于删除（archive before delete）**。全文按技能、任务、会话、运行时四条线拆开，最后落到设计哲学上。

## 一、技能归档：Curator 的三态生命周期与纳管边界

Curator 不是常驻守护进程，而是会话启动时由空闲探测触发的后台维护器。它把技能在 `active → stale → archived` 三个状态之间迁移，阈值全部有默认值（`agent/curator.py:28-32`）：

```yaml
curator:
  enabled: true
  interval_hours: 168      # 两次 pass 的最小间隔，7 天
  min_idle_hours: 2        # 空闲这么久才跑
  stale_after_days: 30     # 闲置 30 天 → 标记 stale
  archive_after_days: 90   # 闲置 90 天 → 移入 .archive/
  consolidate: false       # LLM 合并 pass，默认关闭
```

判定链在 `apply_automatic_transitions()`（`agent/curator.py:191-237`）里：`stale_cutoff = now - 30d`、`archive_cutoff = now - 90d`，锚点 `anchor` 是「最近一次活动时间」。比 `archive_cutoff` 老就归档，比 `stale_cutoff` 老且当前 active 就标 stale，比 `stale_cutoff` 新且当前是 stale 就重新回迁 active。`consolidate` 之所以默认关闭，是因为这个 LLM 合并 pass 每轮要消耗 aux-model token、且对技能库做结构性变更（umbrella 合并、近重复合并），官方文档明确它 off by default（`website/docs/user-guide/features/curator.md:44`）。

这套阈值里藏着一个反直觉的细节：**`use_count == 0` 不等于「可删」**。源码注释原文是 `use_count == 0 is absence of evidence, not staleness`（`agent/curator.py:224-226`）。一个刚创建、从没用过的技能，证据缺失不代表它过期，所以年轻技能一律跳过，不会进到归档判定。这套遥测的载体是 sidecar 文件 `~/.hermes/skills/.usage.json`——按技能名 keyed，`use_count` 与 `created_by` 都落在这里（`tools/skill_usage.py:1,50`）。

**为什么重要：** 这一条挡住了最常见的误删场景——脚本刚铺好一套新技能、还没被调用过，如果按「没用过 = 垃圾」来清，第二天就得重新装一遍。

状态迁移本身有三条硬约束，前两条写在模块 docstring（`agent/curator.py:1-7`），第三条落在 `apply_automatic_transitions()` 的实现里：

1. 永不删除，最大破坏性动作是 archive（移到 `.archive/`，可恢复）；
2. `pinned` 技能跳过所有自动迁移；
3. 被任何 cron job 引用的技能当作 pinned 处理（`agent/curator.py:202,212`：`protected = _cron_referenced_skills()` 后 `if row.get("pinned") or name in protected: continue`）。

纳管边界由 `created_by` 字段决定，而它是个**显式的纳管开关，不是作者归属推断**（`tools/skill_usage.py:271-278`）。本地技能要求 `created_by == "agent"` 才入选；bundled 内置技能在 `prune_builtins=true`（默认）时也算候选；hub 安装的技能**永不**入选，归档时直接拒绝。本机 researcher profile 的实测能直观看到这条边界：

```
$ hermes curator status
curator: ENABLED
  interval:       every 7d
  stale after:    30d unused
  archive after:  90d unused
  consolidate:    off (prune-only; LLM merge pass opt-in)

curator-managed skills: 59 total  (agent-created=1  bundled=58)
  active     59
  stale      0
  archived   0
```

三态在磁盘上的区别，就是下面这张表：

| 状态 | 触发条件 | 磁盘位置 | 还能被加载吗 | 退出方式 |
|---|---|---|---|---|
| `active` | 默认；原为 stale 但重新活跃时回迁 | `skills/<name>/` | 是 | 闲置超期 → `stale` |
| `stale` | 闲置超过 `stale_after_days`(30) | `skills/<name>/`，位置不变 | 是 | 再次活跃 → active；继续闲置 → archived |
| `archived` | 闲置超过 `archive_after_days`(90) | `skills/.archive/<name>` | 否，需 restore | `hermes curator restore <name>` |
| `pinned`（豁免） | `hermes curator pin` | 不变 | 是 | `unpin` |

**为什么重要：** `stale` 阶段技能还在原位、还能被正常加载，它只是「该看看还活没活着」的提醒；真正搬走要等到 `archived`。这种分级给了「淡出」一个缓冲期，而不是从「可用」直接跳「消失」。

## 二、可恢复性工程：快照、账本与两级回滚

每次真实（非 dry-run）curator pass 之前，`~/.hermes/skills/` 先被打成 `skills.tar.gz`，放进 `~/.hermes/skills/.curator_backups/<UTC-ISO>/`，配一份 `manifest.json`，保留份数由 `curator.backup.keep`（默认 5）控制。默认开关是 ON（`agent/curator_backup.py:97-104`，注释 "safety by default"）。

```bash
hermes curator backup --reason "before-refactor"   # 手动快照，--reason 落进 manifest.json
hermes curator rollback                            # 恢复最新快照（带确认）
hermes curator rollback --list                     # 列出快照（reason + size）
hermes curator rollback <entry-id>                 # 只撤销账本里那一次变更
hermes curator ledger --skill <name> --limit 50
```

与备份回滚并列的还有三个归档管理子命令：`list-archived` 列出可恢复的已归档技能（`hermes_cli/curator.py:539-542`）、`restore` 把归档技能移回原布局（不重建嵌套目录，`tools/skill_usage.py:611-633`）、`prune` 批量归档闲置 ≥N 天的纳管技能（pinned 豁免、已归档跳过，`hermes_cli/curator.py:333-348`）。

这其实是**两级**恢复：整树 tar.gz 快照负责「大回滚」，逐次变更的审计账本 `~/.hermes/skills/.curator_ledger.jsonl` 负责「精回滚」。账本每行记一次变更，含 before/after 的 `{path, sha256}`，blob 按内容寻址去重。回滚本身也可逆：执行前会先对当前树再拍一张 pre-rollback 快照，失败则整体放弃（`agent/curator_backup.py:358-382`）。

**为什么重要：** 这套快照只解决「技能目录被 curator 改坏了」，**不是**整机 state 快照，也不含 `.hub/` 与 `.git/`（后者是因为曾把 38MB 的技能目录膨胀成 24GB，见 `agent/curator_backup.py:32-37` 注释）。所以它是技能层面的安全网，别指望它救回被误改的会话数据——那要交给第八节的 state-snapshots。

## 三、任务归档：为什么 archived 是唯一不可逆终态

Kanban 的合法状态有 9 个（`hermes_cli/kanban_db.py:89`），`archived` 是其中的终态。归档动作本身很小：一条 `UPDATE tasks SET status='archived' ...`，顺带清掉认领锁与 worker 进程字段（`kanban_db.py:3490-3509`）。

真正值得说的是 **`done` 是可逆的**。review 打回、dashboard 拖拽、控制器流转，都会重开一个 done 任务，并递归把它的 `ready/review/running/done` 后代降级为 `todo` 重新门控（`kanban_db.py:3338-3343`，docstring 直接标为 "THE done-reopen invalidation"）。而 kanban 内核里**不存在 unarchive**（`grep -rni unarchive hermes_cli/kanban*.py` 零命中；会话侧的 unarchive 见第五节），`hermes kanban archive --help` 只有归档和 `--rm` 删除已归档任务两种模式。

```bash
hermes kanban archive <task_id...>            # 归档：唯一终态
hermes kanban archive --rm <archived_id...>   # 永久删除：只接受「已归档」的 id
```

要彻底抹掉一个任务，必须**先归档、再 `--rm` 删除**，两步刻意操作。`delete_archived_task()` 对非 archived 直接返回 False，注释原文：`anything else must be archived first so data loss takes two deliberate actions`（`kanban_db.py:3519-3527`）。

| 维度 | `done` | `archived` |
|---|---|---|
| 是否终态 | 否，可逆 | 是，唯一不可逆终态 |
| 能否回退 | 能，后代递归降级 todo | 否，无 unarchive |
| 通知订阅 | 保留（靠游标去重） | 退订 |
| 默认视图/统计 | 显示 | 隐藏（`WHERE status != 'archived'`） |
| 删除路径 | 需先 archive | `archive --rm` |

**为什么重要：** 终态设计决定了「任务完成」和「任务了结」是两回事。done 表示「工作做完了，但可能还会被翻出来改」，archived 才是「这件事彻底结束」。把可逆性交还给 done，是后面订阅语义成立的前提。

## 四、订阅与 GC：当「可逆」遇上「有界」

订阅只在任务到达 `archived` 时才退订，`done` 退订被刻意禁止。理由直接写在源码注释里：done 可逆，提前退订会让之后的 reopen 静默丢通知；去重交给游标而不是退订（`gateway/kanban_watchers_notifier.py:40-51`）。

这带来一个必然的代价：永不归档的看板，订阅行会无限累积，而且每行每 tick 都要扫一遍。于是 notifier 加了一个有界 GC，`purge_stale_done_notify_subs()`（`hermes_cli/kanban_db_notify.py:259-296`）：

```sql
DELETE FROM kanban_notify_subs WHERE task_id IN (
  SELECT t.id FROM tasks t
  WHERE t.status IN ('done', 'blocked')
    AND COALESCE(
      (SELECT MAX(e.created_at) FROM task_events e WHERE e.task_id = t.id),
      t.completed_at, t.created_at, 0
    ) < :cutoff
);
```

窗口来自 `kanban.done_sub_retention_days`（默认 30，0 = 关闭）。注意集合里只有 `done` 和 `blocked`：`backlog`/`ready` 这类「只是在等派发」的任务不算废弃，不参与清理。源码注释把区别说得很清楚：`blocked` 是被抛弃而非闲置——原文 "they are abandoned, not idle"（`kanban_db_notify.py:274`）。

**为什么重要：** 这个 GC 的边界一旦划错，就会误删「还在排队」的任务订阅。区分「被抛弃」和「在等待」，是它把「可逆」和「有界」同时保住的关键。

## 五、会话归档：archived / hidden 双标记与 state.db 的分工

会话持久化在**每个 profile 各自**的 `$HERMES_HOME/state.db`（SQLite + FTS5）。sessions 表上有两个互不相同的可见性标记：`archived`（归档/软隐藏）和 `hidden`（隐藏）。两者都让会话从 `hermes sessions list`、`/resume`、桌面侧边栏消失，但都**不删数据**，而且都沿「压缩血缘」整条链一起翻转（`hermes_state_sessions.py:831-833`、`895-897`）。

```sql
-- 默认列表：archived 与 hidden 各是一道独立的可见性闸门
SELECT ... FROM sessions s
WHERE s.archived = 0
  AND s.hidden = 0;
```

两个标记语义不同。`archived` 是用户主动的软隐藏（`hermes sessions archive`，幂等、可 unarchive）；`hidden` 是 Bot Mode 的既有契约——canonical Bot Chat 靠「hidden + 精确标题」这一对判别来阻止被改名（`hermes_state_titles.py:81-90`）。pin 会顺带清除 hidden（`hermes_state_sessions.py:880` 原文 `a pin means "keep this visible"`），但 canonical Bot Chat 例外。

真正的删除走另一条路：`hermes sessions prune`，只删**已结束**的会话，pinned 与进行中的永不删（`config_defaults.py:2006-2018`）。

```bash
hermes sessions archive --title "dry run" --dry-run   # 软隐藏，含 --older-than 等过滤集
hermes sessions prune  --older-than 30                # 真删，默认只删已结束会话
```

至于 `*.jsonl`，它已经被 `state.db` 取代。历史逐会话文件方案不再使用，`.jsonl` 现在的角色是异常兜底：当 state.db 在活进程下被替换时，把待写消息追加到 `<id>.jsonl`（`hermes_state.py:298-312`）。本机 `~/.hermes/sessions/` 实测内容也确实是 `request_dump_*.json` 之类的诊断转储，不是逐会话 transcript。

**为什么重要：** 双标记的设计说明「不可见」和「删除」在 Hermes 里始终是两件事。你把它藏起来，随时能翻出来；真要物理抹掉，得单独走 prune。

## 六、运行时压缩：[SKILL_PRUNED] 与 prompt cache 的取舍

运行时的「归档」由 `context_compressor` 承担：上下文占用超过 `compression.threshold`（默认 0.50，窗口小于 512K 的模型抬到下限 0.75）时触发，把中间回合交给廉价辅助模型摘要，头尾受保护（`protect_first_n=3`、`protect_last_n=20`）。

被压缩掉的 `skill_view` 结果会留下唯一规范标记：

```
[SKILL_PRUNED: content lost in compression; reload with skill_view(name='hermes-agent')]
```

这个占位符会在摘要里被**确定性重新注入**，目的是防 ghost-skill 幻觉——防止模型以为技能还在上下文里、凭空调用一个已经不在的技能。源码还记录过一个历史 bug：曾出现「emit 了 `[SKILL_PRUNED:` 却检查 `[SKILL_PRUNED]`，导致 marker 明明存活还被重复注入」的问题（`agent/context_compressor.py:686-707`）。

压缩的另一面是代价：它重写已发送历史、**破坏 prompt-cache 前缀**。这正是多个默认值被设为关闭的原因：

```yaml
compression:
  enabled: true
  threshold: 0.50
  micro_compact: false       # 每回合都破坏 prompt-cache 前缀，默认关
  proactive_prune_tokens: 0  # 每次 prune 都重写历史，用 min_reclaim 门控让它「偶发」
  in_place: true             # 旧回合 active=0, compacted=1 软归档，仍可 session_search
```

**为什么重要：** 压缩是 Hermes 系统提示硬不变式里「唯一允许破坏 prompt cache 的例外」（这条原句不在任何 `.py:line`，是系统提示模板文本）。它用 prompt cache 的连续性，换来了上下文的有界性——这个交易只有在你理解了前缀缓存的成本之后，才明白为什么 `micro_compact` 默认必须关。

## 七、更新与恢复：state-snapshots 与 update receipts

`hermes update` 默认（`updates.pre_update_backup: quick`）会为**根 home 以及每一个同级 profile** 各拍一份关键状态小快照到 `<HERMES_HOME>/state-snapshots/<ts>-pre-update/`，只保留最近 1 份（`_PRE_UPDATE_SNAPSHOT_KEEP = 1`，`hermes_cli/update_cmd_maint.py:43`）。单个文件超过 1 GiB 会被跳过（`_PRE_UPDATE_SNAPSHOT_MAX_FILE_SIZE = 1 << 30`，`update_cmd_maint.py:47`），以免更新被膨胀的 `state.db` 拖停；执行体是 `hermes_cli/backup.py:1152-1207` 的 `_create_quick_snapshot_locked`。

它的定位被官方文档一句话盖章：**"Quick snapshots are file-loss recovery, not code-rollback insurance"**（`updating.md:37`）。要拿到一致的时间点回滚点，得用 `--backup` 的完整 zip（保留 5 份）。每次 `hermes update` 还会写一份机器可读回执到 `~/.hermes/logs/update_receipts/`（保留 20 份），本机真回执节选：

```json
{
  "outcome": "failed",
  "pre_update":  { "short_sha": "8dc3da96", "version": "0.21.1", "source": "git" },
  "post_update": { "short_sha": "8dc3da96", "version": "0.21.1", "source": "git" },
  "steps": [ { "name": "pre_update_backup", "ok": false, "detail": "disabled or failed" } ]
}
```

注意这条回执里 `pre_update_backup` 是 `ok:false`，而配置默认是 `quick`——说明该 profile 当次被显式关掉或失败了。**以 receipt 为准，别拿默认值当事实。** 另外素材已核实：`hermes snapshot` 不是顶层命令，CLI 侧对应物是 `hermes backup --quick`。

**为什么重要：** 快照解决「文件丢了怎么办」，回执解决「昨晚那次更新到底动了什么」。两者都是事后审计，不是事前保险——这跟第二节的技能快照是同一套思路。

## 八、工作区与垃圾回收：scratch 的生命周期

Kanban worker 的隔离目录有三种 `workspace_kind`（`hermes_cli/kanban_db.py:98` 的 `VALID_WORKSPACE_KINDS = {"scratch", "worktree", "dir"}`）：`scratch`（托管临时目录）与 `worktree`（git 工作树）在可回收集合 `_REMOVABLE_KINDS` 内（`kanban_db_workspace.py:22`），`dir`（共享目录）**故意保留**——它不在 `_REMOVABLE_KINDS`，即不可回收。`scratch` 在任务 `complete` 和 `archive` 两条路径上都会回收，但若还有未终结的子任务则**推迟**——子任务可能仍需读父工作区里的交接产物（`kanban_db_workspace.py:131-139`）。

回收带严格的「托管路径包含性」守卫：绝不对托管根之外的路径 `rmtree`。docstring 记录了它的来由——一个 `default_workdir` 落在真实源码树上、又配了 `workspace_kind='scratch'` 的看板，会让任务完成时误删用户数据（`kanban_db_workspace.py:97-110`）。`hermes kanban gc` 是兜底清理器，回收已归档任务的遗留工作区、旧事件行、旧 worker 日志：

```bash
hermes kanban gc --event-retention-days 30 --log-retention-days 30
```

事件 GC 里有个细节：`gc_events()` 保留 `decomposed` 事件以维持分解血缘（`kanban_db.py:3863-3871`）。

**为什么重要：** 工作区的「可回收」与否，本质是「这份数据还有没有读者」。子任务没结束，父工作区就是活的交接现场；都结束了，它才是垃圾。这个判断一旦做错，要么泄漏磁盘，要么删掉还在被读的产物。

## 九、设计哲学与运维清单

把四条线并排看，Hermes 在技能、任务、会话、通知订阅、工作区五个层面共享同一套语义：**默认动作是可逆的软隐藏/归档，永久删除必须是一次单独的、显式的第二步**。三根支柱支撑它——可追溯（账本、回执，先记录后处置）、可恢复（restore/rollback/unarchive，删之前总有退路）、订阅语义（只要完成可逆，就不因一次 done 解绑）。

archive 与 delete 的边界，一句话版：**archive 改变可见性与状态，delete 消除行与数据**。三处系统的对照如下：

| 维度 | 技能（Curator） | Kanban 任务 | 会话（state.db） |
|---|---|---|---|
| 归档动作 | 目录移入 `.archive/` | `status='archived'` | `archived = 1`（软隐藏） |
| 归档可逆？ | 是，`curator restore` | 否（无 unarchive） | 是，unarchive |
| 默认自动发生？ | 是（90d，pinned/cron 豁免） | 否，人工触发 | 否（`auto_archive: False`） |
| 真删入口 | `curator purge`（TTL 默认 0=永不） | `archive --rm`（须先归档） | `sessions prune`（只删已结束） |

给长期运行 Hermes 的工程师，落地建议五条：

1. 第一次自动 pass 前先 `hermes curator run --dry-run` 预览，因为首次 pass 会被推迟一个 `interval_hours`（`agent/curator.py:151-158`）；
2. 区分两道护栏：`pinned` 还挡 `skill_manage delete`，cron 引用只挡自动迁移；
3. 磁盘真正的杠杆不在技能归档，而在 `sessions.auto_prune` + `VACUUM` 和 `hermes kanban gc`；归档技能默认永久保留，想收口才设 `archive_ttl_days`；
4. 永不归档的看板必须给订阅 GC 留窗口（`kanban.done_sub_retention_days`），否则每 tick 都要扫累积的订阅行；
5. 更新前确认 `updates.pre_update_backup` 实际生效状态，别拿默认值当保障。

归档机制做得好不好，不看你清了多少数据，而看你能不能回答三个问题：**这条状态过期了是「藏起来」还是「删掉」？藏起来之后还能不能找回来？删掉之前有没有留痕？** Hermes 的答案，就是这一套可逆优先、删除显式、处处留痕的设计。

---

## 参考

- Curator：<https://hermes-agent.nousresearch.com/docs/user-guide/features/curator>
- Kanban 任务板：<https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban>
- Sessions 用户指南：<https://hermes-agent.nousresearch.com/docs/user-guide/sessions>
- Session Storage（开发者）：<https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage>
- Context Compression & Caching：<https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching>
- 更新指南（state-snapshots 与回执）：<https://hermes-agent.nousresearch.com/docs/getting-started/updating>

取证基线：Hermes Agent v0.21.1（2026.9.7），源码根 `/Users/sunjian/.hermes/hermes-agent`；文中 `file:line` 均相对此根。
