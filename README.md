# omz (Oh My ZCode) — ZCode 插件版

> 为 AI 编程代理提供多代理编排能力。本插件是 [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) 的 ZCode 适配发行版。
>
> **版本：** 5.4.0 · **插件名：** `omz` · **市场源：** `omz-plugins`

本插件把 omz 的 skills、commands、subagents、hooks 打包成 ZCode 可直接加载的格式。**不包含 omz 的 MCP 服务器** —— 其含义与替代方案见 [限制与迁移指南](#限制与迁移指南)。

---

## 插件内容

| 组件 | 数量 | 来源位置 |
|------|-----:|---------|
| Skills | 39 | `skills/<name>/SKILL.md` |
| Slash commands | 21 | `commands/*.md` |
| Subagents | 19 | `agents/*.md` |
| Hooks | 7 个事件、16 条命令 | `hooks/hooks.json`（事件列表见下） |
| MCP 服务器 | — | **已排除**（见迁移指南） |

### 触发的事件与脚本

| 事件 | 匹配规则 | 脚本 |
|------|---------|------|
| `UserPromptSubmit` | `*` | `keyword-detector`、`skill-injector` |
| `SessionStart` | `*` | `session-start`、`project-memory-session`、`wiki-session-start` |
| `SessionStart` | `init` | `setup-init` |
| `PreToolUse` | `*` | `pre-tool-enforcer` |
| `PermissionRequest` | `Bash` | `permission-handler` |
| `PostToolUse` | `*` | `post-tool-verifier`、`project-memory-posttool`、`post-tool-rules-injector` |
| `PostToolUseFailure` | `*` | `post-tool-use-failure` |
| `Stop` | `*` | `context-guard-stop`、`workflow-drift-guard`、`persistent-mode`、`code-simplifier` |

---

## 功能对照表：ZCode omz vs Claude Code OMC

用本表判断这个 ZCode 版本是否够用，或你需要完整的 Claude Code 发行版。

### 主要功能对齐

| 维度 | Claude Code OMC | ZCode omz（本发行版） | 说明 |
|------|---------------:|----------------------:|------|
| 通过 `$name` 加载的 Skills | 39 / 39 | 39 / 39 | 内容完全一致；frontmatter schema 相同 |
| 通过 `/name` 加载的 Commands | 21 / 21 | 21 / 21 | 内容完全一致 |
| 通过 `@name` 加载的 Subagents | 19 / 19 | 19 / 19 | 同一份 `agents/*.md`；ZCode 支持 `model: opus` 等 |
| `marketplace.json` 格式 | Claude Code schema | 同 schema | 两个运行时都接受同一 marketplace 格式 |
| `plugin.json` 位置 | `.claude-plugin/` | `.zcode-plugin/` | ZCode 先查 `.zcode-plugin/`，再查 `.claude-plugin/` |
| Hook stdin/stdout 协议 | JSON 入参，JSON 出参 | 同 | 遵循同一份 hook 契约 |
| `$CLAUDE_PLUGIN_ROOT` 替换 | ✅ | ✅ | ZCode 同时注入 `ZCODE_PLUGIN_ROOT`，两者都可用 |
| `.omc/` 磁盘状态文件 | ✅ | ✅ | 同路径、同 JSON 结构；skill 直接读取 |
| `package.json` / `npm`（hook 脚本） | 必需 | 无要求（¹） | 本发行版 hook 脚本全部用纯 Node + stdin/stdout；只有一个可选原生模块（`better-sqlite3`）在已安装时被兼容 |

### Hook 事件

| 事件 | Claude Code OMC | ZCode omz | 功能差异 |
|------|:--------------:|:---------:|---------|
| `UserPromptSubmit` | ✅ 触发 | ✅ 触发 | 无 |
| `SessionStart`（`startup` / `clear` / `compact`） | ✅ 触发 | ✅ 触发 | 无 |
| `SessionStart`（`init`）（²） | ✅ 触发 | ⚠️ 几乎不会触发 | ZCode 标准 source 是 `startup` / `clear` / `compact`；`init` 匹配规则（调用 `setup-init.mjs`）为对齐上游而保留，但实际不会自动触发。详见限制章节 §6 |
| `PreToolUse` | ✅ 触发 | ✅ 触发 | 无 |
| `PermissionRequest` | ✅ 触发 | ✅ 触发 | 无 |
| `PostToolUse` | ✅ 触发 | ✅ 触发 | 无 |
| `PostToolUseFailure` | ✅ 触发 | ✅ 触发 | 无 |
| `Stop` | ✅ 触发 | ✅ 触发 | 无 |
| `SubagentStart` | ✅ 触发 | ❌ 已移除 | 失去 `subagent-tracker.mjs` 启动日志 |
| `SubagentStop` | ✅ 触发 | ❌ 已移除 | **失去 `verify-deliverables.mjs`**（任务完成度校验） |
| `PreCompact` | ✅ 触发 | ❌ 已移除 | 失去 `pre-compact.mjs`、`project-memory-precompact.mjs`、`wiki-pre-compact.mjs`（可由更细粒度的 `PostToolUse` / `Stop` flush 缓解） |
| `SessionEnd` | ✅ 触发 | ❌ 已移除 | 失去 `session-end.mjs`、`wiki-session-end.mjs`（无直接替代；用 `$remember` 或将 `.omc/` 提交到 git） |

> （¹）ZCode omz **零必需 npm 依赖**。唯一的可选原生模块是 `better-sqlite3`：装上后可启用并发工作流状态锁（由 `scripts/lib/state-lock.mjs` 使用）；未装时由防御性 `try/catch` 捕获，插件优雅降级。详见下文"优雅降级"。
>
> （²）`init` 匹配规则不是 ZCode 的标准 source。ZCode 文档化的 `SessionStart` source 是 `startup`、`clear`、`compact`。该 hook 仅为与 Claude Code 版 oh-my-claudecode 对齐而保留，在 ZCode 下实际不会自动触发。如需运行 `setup-init.mjs`，请通过 `$omz-setup` skill 调用，或手动执行。

### MCP 服务器工具（约 50 个）

omz 的 MCP 服务器暴露以下工具；**本 ZCode 发行版一概没有**。

| 分组 | 工具数 | 示例 | ZCode omz 内？ | 替代方案 |
|------|------:|------|:-------------:|---------|
| 状态 (State) | 6 | `state_read`、`state_write`、`state_list_active` | ❌ | 直接用 `Read` / `Edit` / `Write` 读写 `.omc/state/*.json` |
| 项目记忆 | 4 | `project_memory_read`、`project_memory_write` | ❌ | 同上，或调用 `$remember` skill |
| 记事本 | 6 | `notepad_write_working`、`notepad_read` | ❌ | 同上，或 `$remember` |
| 共享记忆 | 5 | `shared_memory_read`、`shared_memory_write` | ❌ | 直接读写 `.omc/shared-memory/` |
| Wiki | 7 | `wiki_ingest`、`wiki_read`、`wiki_query` | ❌ | 调用 `$wiki` skill（在文件层面完成） |
| LSP | 11 | `lsp_hover`、`lsp_goto_definition`、`lsp_rename` | ❌ | ZCode 内置 `Read` / `Grep` / `Glob` / `Edit` 已覆盖大部分需求 |
| AST grep | 2 | `ast_grep_search`、`ast_grep_replace` | ❌ | ZCode `Grep` + 正则 |
| Python REPL | 1 | `python_repl` | ❌ | ZCode `Bash` + `python3 -c` |
| 追踪 (Trace) | 2 | `trace_summary`、`trace_timeline` | ❌ | 用 `Read` 读 `.omc/traces/` |
| 会话检索 | 1 | `session_search` | ❌ | `Bash` + `grep` 检索 transcript |
| oh-my-claudecode skills 加载器 | 3 | `list_omc_skills`、`load_omc_skills_global` | ❌ | skill 通过 `$name` 自动加载 |
| Deepinit | 1 | `deepinit_manifest` | ❌ | `$deepinit` skill |
| Merge readiness | 5 | `merge_readiness_start`、`merge_readiness_record_answer` | ❌ | `$review` skill |

### 持久化文件（`.omc/` 目录）

两个发行版写入相同的磁盘位置。区别在于**写入方式**：在 Claude Code oh-my-claudecode 中，hook 和 MCP 工具完成写入；在 ZCode omz 中，hook 仍写入，但 MCP 触发的写入需要替换为 skill 调用或直接文件编辑。

| 路径 | 用途 | ZCode omz 下的读写方式 |
|------|------|----------------------|
| `.omc/state/*.json` | 每个项目的工作流状态 | `Read` / `Edit` / `Write` 工具，或由 hook 驱动 |
| `.omc/project-memory.json` | 持久化的项目上下文 | 同上 |
| `.omc/notepad.md` | 工作中的会话笔记 | 同上；`persistent-mode.mjs` Stop hook 会自动持久化 |
| `.omc/wiki/` | 结构化知识图谱 | 任何变更都通过 `$wiki` 调用 |
| `.omc/shared-memory/` | 跨会话知识库 | `Read` / `Write` 工具 |
| `.omc/traces/` | 执行追踪日志 | `Read` 工具 |
| `.omc/plans/*.md` | `$plan` / `$ralplan` 产出的计划文件 | `Read` / `Edit` 工具 |
| `.omc/handoffs/` | Team 模式的 handoff 文档 | 同上 |

### 配置位置

| 配置项 | Claude Code OMC | ZCode omz |
|--------|----------------|-----------|
| 插件清单 | `.claude-plugin/plugin.json` | `.zcode-plugin/plugin.json` |
| Marketplace 目录 | `.claude-plugin/marketplace.json` | 根目录 `marketplace.json` |
| 用户 hooks / MCP | `~/.claude/settings.json` | `~/.zcode/cli/config.json`（需 `hooks.enabled: true`） |
| 用户 agents | `~/.claude/agents/*.md` | `~/.zcode/agents/*.md` |
| 用户 skills | `~/.claude/skills/<name>/SKILL.md` | `~/.zcode/skills/<name>/SKILL.md` |
| 用户 commands | `~/.claude/commands/*.md` | `~/.zcode/commands/*.md` |
| 项目级 hooks | `settings.json` 或 `.claude/settings.json`（只读展示，必须 import） | **项目级不支持** —— 必须由插件提供 |
| 项目级 MCP | `.mcp.json` | `.mcp.json` 或 `.zcode/config.json` |

### 快速判断

| 使用场景 | 建议 |
|---------|------|
| 主要使用 omz 的 **skills / commands / subagents** 做计划、评审、执行任务 | ✅ **用 ZCode omz** —— 主要入口完全对齐 |
| 跨多会话依赖 omz 的**状态机**（项目记忆 + 记事本 + wiki） | ✅ **用 ZCode omz** —— 文件持久化等价；`$remember` / `$wiki` skill 替代 MCP 写入 |
| 经常用 omz 的 **MCP LSP 工具**做重构 | ⚠️ **用 Claude Code oh-my-claudecode** —— `lsp_rename`、`lsp_code_actions` 在 ZCode 中无直接对应 |
| 依赖 **`verify-deliverables.mjs`** 防止空完成 | ⚠️ **用 Claude Code oh-my-claudecode** —— 或本地加一个 `PostToolUse` hook 匹配 `Agent|Task` |
| 依赖 **PreCompact** 给超长会话做快照 | ⚠️ **用 Claude Code oh-my-claudecode** —— 或把 `.omc/` 提交到 git，并定期跑 `$remember` |
| 需要 **SessionEnd 异步清理**做 wiki 同步 | ⚠️ **用 Claude Code oh-my-claudecode** —— 或在结束会话前手动跑 `$wiki` |

---

## 安装

### 前置条件

ZCode 原生支持 Node.js hook —— 本插件的 hook 脚本以 `node` 运行并从 stdin 读取 JSON。本插件 **零必需 npm 依赖**：唯一曾经被引用的 Node 模块 `bridge/mcp-server.cjs` 已随 MCP 排除一起移除。

**唯一可选的原生模块**：`better-sqlite3`。它对插件加载/运行并非必需。装上后启用并发工作流状态锁；不装时部分工作流状态写入（autopilot / ralph / team）会静默降级，但插件仍能工作。完整行为矩阵见 "优雅降级"。

```bash
node --version    # 任意现代 Node（18+）即可

# 可选：仅当依赖 autopilot/ralph/team 工作流时安装。
# 不装也能跑，但状态写入是尽力而为。
npm install -g better-sqlite3
```

如果你主要使用 skills / commands / subagents（plan、execute、review、verify 等），完全不需要 `better-sqlite3`。

### 安装步骤

#### 方式 A：从 GitHub 安装（推荐）

1. **添加 GitHub marketplace 源**
   打开 ZCode → **Settings → Plugins → Create → Add Marketplace → GitHub
   Repository**，填入：

   ```
   ooptsd/oh-my-zcode
   ```

   也可使用完整 URL：

   ```
   https://github.com/ooptsd/oh-my-zcode
   ```

   ZCode 会自动克隆该仓库，从仓库根目录读取 `marketplace.json`。在
   **Personal** marketplace 下面应该能看到 `omz-plugins`。

   > 仓库当前为 public，clone 不需要凭证。

2. **安装插件**
   在 marketplace 中点击 `omz` 的 **Install**，然后在插件列表里启用它。

3. **验证**
   - Settings → Hooks 应显示该插件的 hooks（只读）。
   - Settings → MCP servers **不应** 列出任何 `plugin:omz:*` 服务（本发行版没有）。
   - 在聊天会话中输入 `/`，能看到 omz 的命令（`/omz-setup`、`/plan`、`/ask` 等）；输入 `$`，能看到 skills（`$plan`、`$execute`、`$review` 等）。
   - 开启新会话，`SessionStart` hook 应触发（可在 `~/.zcode/cli/log/zcode-<date>.jsonl` 中查找 `hook.run` 记录）。

#### 方式 B：从本地源码安装

如果你 clone 了仓库做修改、调试或贡献代码，可以把本地目录直接挂为 marketplace 源，省去 GitHub 拉取这一步。

1. **添加本地 marketplace 源**
   打开 ZCode → **Settings → Plugins → Create → Add Marketplace → Local
   Directory**，指向包含 `marketplace.json` 的目录（即本仓库根目录）：

   ```
   /path/to/oh-my-zcode
   ```

   你应该能在 **Personal** marketplace 下面看到 `omz-plugins`。

2. **安装插件并验证** —— 同方式 A 的步骤 2 与 3。

---

## 优雅降级

本插件的设计目标是 **在没有任何安装任何 npm 包（含原生模块）的情况下也能加载和运行**。只有一个可选原生模块，并且插件能容忍它的缺失：

| 可选模块 | 使用方 | 缺失时的行为 | 推荐安装时机 |
|---------|--------|--------------|------------|
| `better-sqlite3` | `scripts/lib/state-lock.mjs`（在 `try { require('better-sqlite3') } catch {}` 内加载） | `Database` 保持 `null`；`isStateFileLockingSupported()` 返回 `false`；锁获取变为尽力而为。工作流状态写入（autopilot / ralph / team）非竞争场景下仍能成功，但高并发编辑下可能出现竞态 | 大量使用 `autopilot`、`ralph` 或 `team` 工作流，并发访问共享 `.omc/state/*.json` 时 |

`verify.sh` 强制要求 **没有其他** 非 stdlib 模块出现在任何 hook 脚本中（除非包在防御性 `try/catch` 内）。这意味着未来任何引入 `ajv`、`zod` 或其他 npm 包的回归都会在 `bash verify.sh` 时构建失败。`better-sqlite3` 是唯一的例外。

**检查机制：**

1. 扫描 `scripts/` 与 `scripts/lib/` 下所有 `.mjs` / `.cjs` / `.js`。
2. 对每条 `require('X')` 与 `from 'X'` 进行分类：
   - 本地相对路径（`./foo`、`../bar`）：始终允许。
   - Node stdlib（`fs`、`path`、`crypto`、`os`、`child_process`、`util`、`url`、`stream`、`module`、`worker_threads`、`events`、`buffer`、`process`，有无 `node:` 前缀均可）：始终允许。
   - `ajv`、`zod`：**始终拒绝**，即便包在 try/catch 内。
   - 其他：仅当该 require 出现在同一源文件 `try {` 之后 200 字符以内时才允许。
3. 任何违规都会让 `verify.sh` 退出非零，并打印精确的 `file:line: <reason>`。

如需新增可选依赖，把 `require()` / `import` 包在 `try/catch` 中，并保证周边代码能优雅处理 `null` / `undefined` —— 验证检查就会放行。

---

## 限制与迁移指南

omz 的完整发行版依赖 ZCode 不支持的 hook 与一个暴露约 50 个工具的 MCP 服务器。本 ZCode 发行版把它们全部排除。下面是完整的丢失清单及具体迁移路径。

### 1. MCP 服务器（`bridge/mcp-server.cjs`）—— 已排除

MCP 服务器按用途分组暴露以下工具，全部在本插件中不可用。

| 分类 | 工具 | 用途 | 迁移 |
|------|------|------|------|
| **State** | `state_write`、`state_read`、`state_list_active`、`state_get_status`、`state_clear`、`state_migrate_non_git` | 每个项目的工作流状态（`.omc/state/`） | 状态文件仍在磁盘上。读取它们的 skill（如 `plan`、`execute`）回退到直接用 `Read` 工具读文件，不需要 MCP |
| **项目记忆** | `project_memory_write`、`project_memory_read`、`project_memory_add_note`、`project_memory_add_directive` | 持久化项目上下文（`.omc/project-memory.json`） | 直接用 `Read` / `Edit` / `Write` 操作 `.omc/project-memory.json`，或调用 **`remember`** skill（封装相同逻辑） |
| **记事本** | `notepad_write_working`、`notepad_write_priority`、`notepad_write_manual`、`notepad_read`、`notepad_stats`、`notepad_prune` | 工作中的会话笔记（`.omc/notepad.md`） | 调用 **`remember`** skill 或直接读写 `.omc/notepad.md`。`persistent-mode.mjs` Stop hook 已在会话结束时自动持久化记事本 |
| **共享记忆** | `shared_memory_write`、`shared_memory_read`、`shared_memory_list`、`shared_memory_delete`、`shared_memory_cleanup` | 跨会话、跨项目的知识库 | 直接读写 `.omc/shared-memory/` 下的文件。`wiki` skill 提供类似能力，结构更严谨 |
| **Wiki** | `wiki_ingest`、`wiki_read`、`wiki_query`、`wiki_add`、`wiki_delete`、`wiki_list`、`wiki_lint` | 结构化知识图谱 | 直接调用 **`wiki`** skill（聊天中 `$wiki`），所有 wiki 操作都以文件编辑形式封装 |
| **LSP** | `lsp_servers`、`lsp_diagnostics`、`lsp_diagnostics_directory`、`lsp_hover`、`lsp_goto_definition`、`lsp_find_references`、`lsp_document_symbols`、`lsp_workspace_symbols`、`lsp_code_actions`、`lsp_code_action_resolve`、`lsp_prepare_rename`、`lsp_rename` | 代码智能 | **使用 ZCode 内置的 `Read` / `Grep` / `Glob` 工具** 已覆盖大部分导航需求。重构 / 重命名依靠编辑器原生 LSP 或 ZCode 的 `Edit` 工具 |
| **AST grep** | `ast_grep_search`、`ast_grep_replace` | 结构化代码搜索/替换 | 用 **`Grep`** + 正则即可。编排工作流中很少需要 AST 级精度 |
| **Python REPL** | `python_repl` | 进程内执行 Python | 用 **`Bash`** + `python3 -c '...'` 或写入临时文件 |
| **追踪 (Trace)** | `trace_summary`、`trace_timeline` | 查看历史会话的执行追踪 | 直接用 `Read` 读追踪文件（`.omc/traces/`） |
| **会话检索** | `session_search` | 跨会话搜索 | `Bash` + `grep -r` 检索 `~/.zcode/cli/transcripts/`（或等价路径） |
| **oh-my-claudecode skills 加载器** | `list_omc_skills`、`load_omc_skills_global`、`load_omc_skills_local` | 发现可用的 oh-my-claudecode skills | ZCode 中通过 `$skill-name` 直接调用 skill，无需枚举工具 |
| **Deepinit manifest** | `deepinit_manifest` | 生成项目初始化清单 | 调用 **`deepinit`** skill（`$deepinit`），效果相同 |
| **Merge readiness** | `merge_readiness_start`、`merge_readiness_report`、`merge_readiness_set_content`、`merge_readiness_record_answer`、`merge_readiness_cancel` | 多方合并 Q&A 工作流 | 使用 **`review`** skill，它以手动编排方式覆盖相同工作流 |

**经验法则**：如果某个 Claude Code 调用本来会调用 oh-my-claudecode MCP 工具，在 ZCode 中改为调用对应的 `skills/<name>/SKILL.md` 或直接读写底层文件。Skills（39 个）才是主要入口；MCP 服务器只是一层便利封装。

### 2. `SubagentStart` / `SubagentStop` hooks —— 不支持

ZCode 不触发这两个事件。两个脚本受影响。

#### `scripts/subagent-tracker.mjs`

**在 Claude Code 上会做的事：**
把每次 `Task` / sub-agent 调用的开始与结束记录到追踪文件（通常是 `.omc/state/subagents.jsonl` 或类似位置），以便 omz 审计哪些 sub-agent 运行过、运行多久、结果如何。

**迁移：** 该追踪文件是辅助信息。如果你在 ZCode 里需要审计日志，可使用一个匹配 `Agent|Task` 的 `PostToolUse` hook（本地添加；本插件不自带，因为 omz 的追踪脚本是 Claude Code 专用）。或者事后查阅 `~/.zcode/cli/transcripts/`。

#### `scripts/verify-deliverables.mjs`

**它做的事：** 这是 **影响最大的丢失**。在每次 `SubagentStop` 时，它会校验正在完成的 sub-agent 是否真的产出了约定的交付物（文件存在 + 最小内容）。它能抓出 "任务标记完成但零输出文件" 这种已知故障模式 —— 自主代理的常见弱点。

**迁移选项（按有效性排序）：**

1. **添加本地 `PostToolUse` hook**：在你的 ZCode 用户配置（`~/.zcode/cli/config.json`）中，添加一个匹配 `Agent|Task` 的 hook，里面挂你自己的交付物校验脚本。具体 schema 见 ZCode hooks 文档。
2. **多步工作流完成后手动校验。** 当 `execute` 或 `team` skill 结束后，先核对计划文件（`.omc/plans/*.md`）中列出的预期输出路径，再认定任务完成。
3. **使用 `verify` skill（`$verify`）** 在任何多交付物任务结束时调用，它会走一遍计划的验收标准并人工核对证据。

### 3. `PreCompact` hook —— 不支持

ZCode 在上下文窗口压缩前不触发此事件。三个脚本受影响。

#### `scripts/pre-compact.mjs`

**它做的事：** 把 omz 进行中的工作流状态（当前计划步骤、todo 列表、agent 分配、最近工具调用）快照到 `.omc/state/snapshot-<timestamp>.json`，以便压缩后恢复。

**迁移：**
- `persistent-mode.mjs` 这个 `Stop` hook（**本插件里有**）已经在每次 `Stop` 事件时写快照。快照频率比 PreCompact 高，覆盖范围类似。
- 长会话中可周期性调用 **`remember`** skill（`$remember`），比如每 ~10 个 prompt 一次，把状态写入 `.omc/notepad.md` 与 `.omc/project-memory.json`。

#### `scripts/project-memory-precompact.mjs`

**它做的事：** 在压缩前把内存里的项目记忆缓冲区刷到 `.omc/project-memory.json`，防止丢失。

**迁移：**
- `project-memory-posttool.mjs` 这个 `PostToolUse` hook（**本插件里有**）在每次工具调用后都刷盘，粒度比 PreCompact 更细。
- 为保险起见，在任何可能触发压缩的长任务前先跑 `$remember`。

#### `scripts/wiki-pre-compact.mjs`

**它做的事：** 把进行中的 wiki 图保存到 `.omc/wiki/state.json`。

**迁移：** `wiki-session-start.mjs` 与 `wiki-session-end.mjs`（等等 —— `wiki-session-end.mjs` 也丢了，见下文）两个 hook 负责加载和保存 wiki。没有 `SessionEnd` 时，只有 `wiki-session-start.mjs` 会在会话开始时跑。要补偿这一点，定期调用 `$wiki` 触发保存，或者把 `.omc/wiki/` 提交到 git，由 VCS 保存状态。

### 4. `SessionEnd` hook —— 不支持

ZCode 不触发此事件。两个脚本受影响。

#### `scripts/session-end.mjs`（异步）

**它做的事：** 在会话结束时把最终状态持久化到磁盘：`.omc/state/`、`.omc/notepad.md`、`.omc/project-memory.json`。以异步方式运行（fire-and-forget），不阻塞会话。

**迁移：**
- `Stop` hook 是 **最接近的替代**，但它按模型回合触发，而非按会话触发。如果需要会话结束语义，在 ZCode 配置中添加本地 hook 或依赖手动 `$remember` 调用。
- 把 `.omc/state/` 与 `.omc/notepad.md` 频繁提交到 git，让它们活在会话边界之外。

#### `scripts/wiki-session-end.mjs`（异步）

**它做的事：** 在会话结束时保存 wiki 图。

**迁移：** 与上文 `wiki-pre-compact.mjs` 相同。`wiki-session-start.mjs` 会在下次会话开始时加载回来；要保持最新，需要手动调用 `$wiki` 或提交 git。

### 5. `SessionStart` 的 `maintenance` 匹配规则 —— 死代码

oh-my-claudecode 定义了一个匹配 `maintenance` 的 `SessionStart` hook，调用 `setup-maintenance.mjs`。ZCode 与 Claude Code 都没有把 `maintenance` 列为标准会话 source（标准是 `startup`、`clear`、`compact`）。这条匹配规则 **在两个运行时里都从未触发过**。本发行版已排除。

**迁移：** 如果你以前用过 omz 的 `/omz-setup maintenance` 工作流，改用 `$omz-setup` skill 配以相应子命令手动调用。

### 6. `SessionStart` 的 `init` 匹配规则 —— 非标准 ZCode source

插件 **确实** 提供一个匹配 `init` 的 `SessionStart` hook，调用 `setup-init.mjs`。该 hook 是为与 Claude Code 版 oh-my-claudecode 对齐而保留。但 ZCode 文档化的 `SessionStart` source 是 `startup`、`clear`、`compact` —— `init` **不是** 标准 ZCode source，所以这条匹配规则在 ZCode 下实际不会自动触发。匹配规则仍保留在 `hooks/hooks.json` 中，因为删掉它会让本发行版与上游 omz 的 hook 契约出现分歧。

**迁移：** 若想运行 `setup-init.mjs`，手动通过 `$omz-setup` skill（`$omz-setup init`）调用，或直接执行：

```bash
node "$ZCODE_PLUGIN_ROOT"/scripts/setup-init.mjs < /dev/null
```

这是一个已知限制而非 bug —— 该匹配规则留在清单里，未来 ZCode 一旦把 `init` 加为会话 source，无需改插件就会自动接管。

---

## 与 Claude Code 行为一致的部分

在受支持的事件集合中（11 个里的 7 个），行为与 omz 的 Claude Code 版完全相同：

- 每个 prompt 上的关键词检测与 skill 注入。
- 跨工具调用持久化的项目记忆。
- 工具使用强制（PreToolUse）。
- Bash 权限门禁。
- 工具结果校验（PostToolUse）。
- 失败恢复提示（PostToolUseFailure）。
- Stop 时刻的上下文守卫、漂移守卫、persistent mode、code simplifier。

变量 `$CLAUDE_PLUGIN_ROOT`（出现在每条 hook 命令中）由 ZCode 在 hook 执行时替换为插件根目录。包装脚本 `scripts/run.cjs` 按 ZCode hook 契约从 stdin 读取、向 stdout 写入 JSON。

---

## 本发行版 **没有** 的内容（汇总）

供对比上游 oh-my-claudecode 的用户参考：

- ❌ MCP 服务器（`bridge/mcp-server.cjs`）及其约 50 个工具
- ❌ Subagent 追踪 hooks（`SubagentStart` / `SubagentStop`）
- ❌ 交付物校验 hook（`verify-deliverables.mjs`）
- ❌ PreCompact 状态快照（3 个脚本）
- ❌ SessionEnd 异步清理（2 个脚本）
- ❌ 死代码 `maintenance` 匹配规则

**净效果：** omz 的 *skills、commands、subagents*（真正的工作流入口）行为完全一致。omz 的 *运行时簿记*（MCP 状态、subagent 追踪、压缩快照、会话结束清理）会出现降级，需要手动或基于 git 的补偿。

如果你需要完整的 Claude Code 功能集，请通过其原生安装器（`.claude-plugin/marketplace.json`）安装 oh-my-claudecode。

---

## 开发

### 与上游同步

本发行版是 oh-my-claudecode 5.4.0 的快照。如需从 `/path/to/oh-my-claudecode/` 重新同步：

```bash
# 手动重新拷贝组件
rsync -av --delete \
  /path/to/oh-my-claudecode/commands/ \
  /path/to/oh-my-claudecode/commands/

rsync -av --delete \
  /path/to/oh-my-claudecode/skills/ \
  /path/to/oh-my-claudecode/skills/

rsync -av --delete \
  /path/to/oh-my-claudecode/agents/ \
  /path/to/oh-my-claudecode/agents/

# 通过剥离不支持的事件来重新生成 hooks/hooks.json
# 通过排除不支持事件的脚本来重新生成 scripts/
# （无脚本化等价物 —— 人工对照上文 hook / 脚本列表）
```

### 文件数量（安装后）

```bash
find /path/to/oh-my-claudecode \
  \( -path '*/skills/*/SKILL.md' -o -name '*.md' \) | wc -l
# 预期：~80（39 个 skills + 21 个 commands + 19 个 agents）
```

---

## 许可证

MIT —— 见 `LICENSE`（继承自上游 oh-my-claudecode）。