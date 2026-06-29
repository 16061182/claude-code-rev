# Claude Code 源码深度技术分析

> **版本**：基于 `@anthropic-ai/claude-code 999.0.0-restored` 还原源码树  
> **代码规模**：2,015 个 TypeScript/TSX 源文件，约 52 万行代码  
> **运行时**：Bun ≥ 1.3.5 / Node.js ≥ 24.0.0  
> **文档目标**：面向资深工程师的全栈架构分析，覆盖全部子系统

---

## 目录

- [Chapter 1 — 项目全景与 Harness 工程哲学](#chapter-1)
- [Chapter 2 — 启动与引导链路](#chapter-2)
- **PART I — Agent Loop 核心**
  - [Chapter 3 — Agent Loop：AsyncGenerator + 状态机](#chapter-3)
  - [Chapter 4 — 工具执行系统：StreamingToolExecutor + 智能并发](#chapter-4)
  - [Chapter 5 — 权限与安全模型](#chapter-5)
  - [Chapter 6 — Hook 系统](#chapter-6)
- **PART II — 复杂任务处理**
  - [Chapter 7 — 任务规划与命令系统](#chapter-7)
  - [Chapter 8 — 子 Agent 与上下文隔离](#chapter-8)
  - [Chapter 9 — 上下文管理：五级递进压缩](#chapter-9)
- **PART III — 记忆与恢复**
  - [Chapter 10 — 持久化记忆系统](#chapter-10)
  - [Chapter 11 — System Prompt 运行时组装](#chapter-11)
  - [Chapter 12 — 错误恢复策略](#chapter-12)
- **PART IV — 长任务与后台执行**
  - [Chapter 13 — Task 系统](#chapter-13)
  - [Chapter 14 — 后台任务与 Cron 调度](#chapter-14)
- **PART V — 多 Agent 团队协作**
  - [Chapter 15 — Agent 团队架构](#chapter-15)
  - [Chapter 16 — Worktree 隔离与远程模式](#chapter-16)
- **PART VI — 扩展能力**
  - [Chapter 17 — Skill 与 Plugin 系统](#chapter-17)
  - [Chapter 18 — MCP 协议与外部工具池](#chapter-18)
  - [Chapter 19 — 多 LLM Provider 支持](#chapter-19)
- **PART VII — KV-Cache 感知设计（核心中的核心）**
  - [Chapter 20 — KV-Cache 感知：贯穿全系统的优化哲学](#chapter-20)
- **PART VIII — 基础设施层**
  - [Chapter 21 — 终端 UI 层](#chapter-21)
  - [Chapter 22 — 服务层横切关注点](#chapter-22)
  - [Chapter 23 — 原生模块与垫片层](#chapter-23)
- [Chapter 24 — 附录](#chapter-24)

---

<a name="chapter-1"></a>
# Chapter 1 — 项目全景与 Harness 工程哲学

## 1.1 什么是 Harness：模型是司机，Harness 是车辆

Claude Code 的本质不是一个"AI Agent"，而是一个 **Harness**——让语言模型在代码工程领域高效运作的执行环境。这个区分至关重要：

- **Agency**（自主行动能力）来自模型训练，不来自外部代码编排
- **Harness** 是给模型提供感知、行动、记忆和权限边界的基础设施

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:          file I/O, shell, network, browser, MCP
    Knowledge:      CODEBUDDY.md, memory, skills, system prompt
    Observation:    git diff, error logs, file state, task notifications
    Action:         CLI commands, API calls, file writes, spawning agents
    Permissions:    sandbox isolation, approval workflows, trust boundaries
```

拆解 Claude Code 的本质：

```
Claude Code = one agent loop
            + tools (bash, read, write, edit, glob, grep, browser...)
            + on-demand skill loading
            + context compaction (5-level)
            + subagent spawning
            + task system with dependency graphs
            + async mailbox team coordination
            + worktree-isolated parallel execution
            + permission governance
            + hooks extension system
            + memory persistence
            + MCP external capability routing
            + KV-Cache-aware design (first citizen)
```

## 1.2 技术栈与运行环境

| 层次 | 技术 |
|------|------|
| **运行时** | Bun 1.3.5+（主），Node.js 24+（兼容） |
| **语言** | TypeScript / ESM，react-jsx |
| **UI 框架** | React 19 + 自研 Ink 渲染引擎 |
| **CLI 框架** | Commander.js + Zod 参数校验 |
| **AI/LLM** | @anthropic-ai/sdk，Bedrock，Vertex，GitHub Models |
| **协议** | MCP（Model Context Protocol），OpenTelemetry，JSON-RPC |
| **代码工具** | Sharp（图像），diff，Highlight.js，marked |
| **数据格式** | JSONC，YAML，cacache，proper-lockfile |
| **布局引擎** | Facebook Yoga（TypeScript 纯 TS 移植版） |

**为什么选 Bun？**

- 原生 TypeScript 执行，无需编译步骤
- `bun:bundle` 提供编译期特性门控（`feature()` → 死代码消除）
- 更快的启动时间（与 Node.js 相比约快 4-5 倍）
- 内置包管理器（`bun install`）

## 1.3 代码规模与模块分布

```
src/
├── 39 个子目录（功能模块）
├── 22 个根级 TypeScript 文件（核心入口与工具）
└── 总计约 2,015 个源文件

关键文件行数：
  src/main.tsx          4,690 行  — 主 REPL 循环
  src/query.ts          1,730 行  — Agent Loop 核心
  src/QueryEngine.ts    1,295 行  — 多轮会话管理
  src/Tool.ts             792 行  — 工具框架
  src/commands.ts         410 行  — 命令注册
  src/services/mcp/client.ts  3,000+ 行  — MCP 客户端
  src/skills/loadSkillsDir.ts 1,087 行  — 技能加载
```

## 1.4 顶层目录结构

```
claude-code-rev/
├── src/
│   ├── entrypoints/      CLI 入口与 SDK 类型
│   ├── commands/         80+ 命令实现
│   ├── tools/            40+ 工具实现
│   ├── services/         23 个服务模块
│   ├── components/       30+ React 组件
│   ├── utils/            40+ 工具函数模块
│   ├── tasks/            8 种 Task 类型实现
│   ├── bridge/           远程控制与进程间通信
│   ├── remote/           远程会话管理
│   ├── memdir/           持久化记忆系统
│   ├── skills/           技能系统
│   ├── plugins/          插件系统
│   ├── ink/              自研终端渲染引擎
│   ├── state/            全局状态管理
│   ├── context/          React Context 体系
│   ├── keybindings/      键绑定系统
│   ├── vim/              Vim 模式模拟
│   ├── query/            查询引擎配置子模块
│   └── ...
├── shims/                原生模块垫片（11 个）
├── vendor/               vendored 源码构建产物
├── AGENTS.md             仓库开发规范
└── package.json
```

## 1.5 关键常量（src/constants/）

`src/constants/` 目录集中存放全局常量，避免魔法数字散落各处：

| 文件 | 核心内容 |
|------|---------|
| `toolLimits.ts` | 工具输出预算（50K/200K 字符限制） |
| `systemPromptSections.ts` | System Prompt 各 Section 的 缓存键 |
| `prompts.ts` | 各类提示模板（压缩、恢复、摘要） |
| `product.ts` | 产品名称、版本、发布渠道 |
| `oauth.ts` | OAuth 端点、客户端 ID |
| `errorIds.ts` | 错误码枚举 |
| `apiLimits.ts` | API 速率限制常量 |
| `betas.ts` | Anthropic Beta 功能标志 |
| `messages.ts` | 用户可见消息模板 |
| `keys.ts` | 键盘快捷键名称 |

## 1.6 版本迁移机制（src/migrations/）

系统通过 `migrationVersion` 字段追踪设置格式版本，当前版本为 11。每次启动时在 `runMigrations()` 中顺序执行迁移：

```
迁移历史（当前 11 个）：
  migrateAutoUpdatesToSettings
  migrateBypassPermissionsAcceptedToSettings
  migrateEnableAllProjectMcpServersToSettings
  resetProToOpusDefault
  migrateSonnet1mToSonnet45
  migrateLegacyOpusToCurrent
  migrateSonnet45ToSonnet46
  migrateOpusToOpus1m
  migrateReplBridgeEnabledToRemoteControlAtStartup
  resetAutoModeOptInForDefaultOffer
  migrateFennecToOpus（仅 ANT 内部构建）
```

迁移是**幂等的**——已完成的迁移通过版本号检查跳过，确保多次启动不重复执行。

---

<a name="chapter-2"></a>
# Chapter 2 — 启动与引导链路

## 2.1 Bootstrap 入口与快速路径

Claude Code 的启动链路经过精心设计，核心原则是：**让 `--version` 在零模块加载的情况下立即返回**。

### 入口文件链路

```
bootstrap-entry.ts (6行)
  └── bootstrapMacro.ts → 注入全局 MACRO 对象
  └── entrypoints/cli.tsx → 快速路径分发器
        └── main.tsx → 主 REPL 循环（延迟加载）
```

**bootstrap-entry.ts**（6 行）：

```typescript
import { ensureBootstrapMacro } from './bootstrapMacro'
ensureBootstrapMacro()
await import('./entrypoints/cli.tsx')
```

**bootstrapMacro.ts**（30 行）：

向 `globalThis.MACRO` 注入构建时常量，包括版本号、包 URL、更新日志链接等。这些值在构建时内联，运行时无需读取 `package.json`：

```typescript
type MacroConfig = {
  VERSION: string           // 来自 package.json
  BUILD_TIME: string        // 构建时戳
  PACKAGE_URL: string       // npm 包名
  NATIVE_PACKAGE_URL: string
  VERSION_CHANGELOG: string
  ISSUES_EXPLAINER: string
  FEEDBACK_CHANNEL: string  // 'github'
}
```

**dev-entry.ts**（诊断模式，123 行）：

仅用于还原开发工作流的诊断入口。启动时扫描 `src/` 和 `vendor/` 中所有相对导入，若存在无法解析的导入则输出报告并**不进入主程序**，只有 `missing_relative_imports === 0` 时才路由到 `entrypoints/cli.tsx`。

## 2.2 CLI 初始化序列（entrypoints/cli.tsx，303 行）

`cli.tsx` 是**快速路径分发器**，其设计原则是：不同入口场景仅加载各自所需的模块，最小化冷启动开销。

### 顶层副作用（所有路径共享）

```typescript
// 1. 禁用 corepack 自动 pin（防止意外修改 package.json）
process.env.COREPACK_ENABLE_AUTO_PIN = '0'

// 2. CCR（远程模式）内存调优
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  // 追加 NODE_OPTIONS: --max-old-space-size=8192
}

// 3. Ablation 基准测试（feature 门控）
if (feature('ABLATION_BASELINE')) { /* ... */ }
```

### 快速路径一览

| 参数 | 加载模块 | 说明 |
|------|---------|------|
| `--version` / `-v` | **零** | 直接输出 `MACRO.VERSION`，约 5ms 返回 |
| `--dump-system-prompt` | 最小集 | 输出 System Prompt 后退出（仅 ANT 构建） |
| `--claude-in-chrome-mcp` | Chrome MCP | 运行 Chrome 扩展桥接服务 |
| `--daemon-worker=<kind>` | Daemon Worker | 精简模式，跳过 analytics sink |
| `remote-control` / `bridge` | Bridge 模块 | 验证 OAuth → 策略检查 → `bridgeMain()` |
| `daemon` | Daemon 主程序 | 监督进程 |
| `ps`/`logs`/`attach`/`kill` | Session 管理 | 后台会话操作 |
| **（默认）** | **main.tsx** | 交互式 REPL |

### 默认路径（交互式）

```typescript
// 1. 捕获早期输入（防止 stdin 数据丢失）
startCapturingEarlyInput()

// 2. 性能检查点
profileCheckpoint('cli_before_main_import')

// 3. 动态导入 main.tsx（延迟加载约 135ms 的模块链）
const { main: cliMain } = await import('./main.js')

profileCheckpoint('cli_after_main_import')
await cliMain()
profileCheckpoint('cli_after_main_complete')
```

### Feature Flag 机制

所有条件分支使用 `feature()` 函数：

```typescript
if (feature('BRIDGE_MODE')) {
  // 此分支在非 bridge 构建中被 Bun bundler 死代码消除
}
```

`feature()` 来自 `bun:bundle`，在编译期将未启用的功能完全从 bundle 中移除，实现零运行时开销的功能门控。

## 2.3 CLI I/O 层（src/cli/）

`src/cli/` 提供 CLI 的输入/输出基础设施：

### 核心模块

**exit.ts**（31 行）：

```typescript
export function cliError(msg?: string): never
export function cliOk(msg?: string): never
```
集中式退出辅助函数，返回类型 `never` 使 TypeScript 控制流分析正确工作。

**ndjsonSafeStringify.ts**（33 行）：

转义 U+2028（行分隔符）和 U+2029（段落分隔符），防止 NDJSON 流被意外截断：

```typescript
// 这两个 Unicode 字符在 JSON 字符串中合法，但会被某些接收方视为换行符
str.replace(/\u2028|\u2029/g, c => '\\u' + c.charCodeAt(0).toString(16))
```

**structuredIO.ts**：SDK 模式的双向消息流，实现异步生成器接口 `structuredInput()`，支持 `StdinMessage | SDKMessage` 的流式读取，维护 pending 请求追踪表和已解析 tool_use ID 集合（上限 1000 条）。

**remoteIO.ts**：继承 `StructuredIO`，为 CCR（Cloud Code Runner）模式提供远程双向流，支持 URL 解析和 Transport 选择（WebSocket / SSE / Hybrid）。

**transports/ 目录**（8 个传输层实现）：

| 文件 | 说明 |
|------|------|
| `WebSocketTransport.ts` | WebSocket 连接 |
| `SSETransport.ts` | Server-Sent Events + POST |
| `HybridTransport.ts` | WebSocket 主，SSE 降级 |
| `SerialBatchEventUploader.ts` | 事件批量上传 |
| `WorkerStateUploader.ts` | CCR Worker 状态持久化 |
| `ccrClient.ts` | CCR v2 客户端（心跳、epoch、写入） |
| `transportUtils.ts` | Transport 工厂（按 URL 选择） |

## 2.4 启动并行预取与性能 Profiler

`main.tsx` 的开头是整个启动链路中最关键的性能优化点——在模块导入链（约 135ms）执行的**同时**并行启动外部子进程：

```typescript
// main.tsx 前 20 行（import 语句之前执行）
import { profileCheckpoint } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')           // T=0

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()   // 立即 fork 子进程读取 macOS MDM 策略

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()  // 立即 fork 子进程预取 Keychain OAuth Token
```

**设计原理**：`plutil`（读取 macOS plist）和 Keychain 访问都是外部进程调用，耗时 50-200ms。通过在 import 链执行期间并行启动这些进程，将其延迟完全隐藏在模块加载时间内。

### 启动 Profiler 时间线

```
T=0ms    main_tsx_entry                 ← startMdmRawRead() + startKeychainPrefetch() 同时启动
T=~100ms main_tsx_imports_loaded        ← 所有顶层 import 完成
T=~200ms cli_before_main_import
T=~300ms cli_after_main_import
T=~500ms cli_after_main_complete        ← 交互式 REPL 就绪
```

## 2.5 Feature Flags 与懒加载机制

**Feature Flags**（编译期，`bun:bundle`）：

```typescript
// 死代码消除：DAEMON 未启用时此分支从 bundle 移除
if (feature('DAEMON')) {
  const { daemonMain } = await import('./daemon/main.js')
  await daemonMain(args.slice(1))
  return
}
```

**运行时 GrowthBook 特性门控**：

```typescript
// 运行时动态加载，支持 A/B 测试
const isEnabled = await growthbook.isOn('tengu_some_feature')
```

两种机制组合使用：编译期门控消除不需要的代码体积，运行时门控支持灰度发布。

## 2.6 主循环入口（main.tsx）与 REPL 启动

`main.tsx`（4,690 行）是整个 CLI 的主控制器，其 `main()` 函数执行以下阶段：

### Phase 1：预初始化（并行）

```typescript
// 1. 安全：防止 Windows PATH 劫持
process.env.NoDefaultCurrentDirectoryInExePath = '1'

// 2. 等待 MDM 和 Keychain 预取完成
await Promise.all([awaitMdmRawRead(), awaitKeychainPrefetch()])

// 3. 运行迁移
await runMigrations()

// 4. 并行预取启动遥测
await Promise.all([
  getIsGit(cwd),
  getWorktreeCount(cwd),
  getGhAuthStatus(),
])
```

### Phase 2：Commander 命令树注册

通过 `preAction` 钩子确保 `init()` 在任何子命令执行前完成：

```typescript
program.hook('preAction', async (thisCommand) => {
  await Promise.all([awaitMdmRawRead(), awaitKeychainPrefetch()])
  await init()           // 加载配置、GrowthBook、认证
  await runMigrations()
  loadRemoteManagedSettings()  // 非阻塞
})
```

### Phase 3：交互式 vs 非交互式分支

```typescript
if (isNonInteractiveSession) {
  // --print / -p 模式：调用 ask()，无 TUI
  await ask(prompt, options)
} else {
  // 交互式 REPL：启动 React/Ink TUI
  await launchRepl(options)
}
```

### setup() 函数（src/setup.ts，478 行）

`init()` 内部调用 `setup()`，按严格顺序完成：

1. Node.js 版本检查（≥18）
2. 会话 ID 切换（`--resume` 场景）
3. Unix Domain Socket 创建（Hooks 快照依赖此时序）
4. 工作目录设置 → Hooks 配置快照
5. Worktree 创建（含 git 集成和 tmux 会话）
6. 后台任务初始化（SessionMemory、ContextCollapse）
7. 插件预取（并行，非阻塞）
8. Analytics Sink 绑定（`initSinks()`，阻塞）
9. 首个可靠启动事件 `logEvent('tengu_started')`
10. 权限模式验证（bypass 权限的 root/sudo 检查）

### 延迟预取（startDeferredPrefetches）

**首次渲染后**才执行，避免阻塞 TUI 显示：

```typescript
async function startDeferredPrefetches() {
  // 跳过条件：bare 模式、--print 模式、单次渲染后退出
  if (isBareMode() || isNonInteractiveSession) return
  
  await Promise.all([
    initUser(),
    getUserContext(),
    prefetchSystemContextIfSafe(),   // 仅在用户已同意信任对话框后
    getRelevantTips(),
    countFilesRoundedRg(cwd, AbortSignal.timeout(3000)),
    initializeAnalyticsGates(),
    prefetchOfficialMcpUrls(),
    refreshModelCapabilities(),
    settingsChangeDetector.initialize(),
    skillChangeDetector.initialize(),
  ])
}
```

---

<a name="chapter-3"></a>
# Chapter 3 — Agent Loop：AsyncGenerator + 状态机

> **这是 Harness 工程的核心。** 所有其他机制——工具、压缩、权限、Hooks——都是在这个循环的框架内运作的。

## 3.1 整体架构：Generator-Based Streaming 设计

Claude Code 的 Agent Loop 位于 `src/query.ts`（1,730 行），采用 **AsyncGenerator + while(true) 状态机**模式。

### 为什么选 AsyncGenerator 而非 Promise？

```typescript
// Promise 方式：必须等待整个响应才能处理
const response = await callModel(messages)
processResponse(response)

// AsyncGenerator 方式：边产生边消费，真正的流式
async function* query(params): AsyncGenerator<StreamEvent, Terminal> {
  while (true) {
    for await (const message of callModel(messages)) {
      yield message  // 立即传递给消费者，无需等待完整响应
    }
  }
}
```

AsyncGenerator 使得：
1. UI 可以在 LLM 流式输出期间实时更新
2. 工具执行可以与 LLM 响应流水线化（边接收边执行）
3. 错误可以在发生时立即处理，而非等待完整响应

### 导出签名

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent          // 流式事件（token、工具状态等）
  | RequestStartEvent    // 每轮 API 调用开始
  | Message              // 完整消息（assistant/user/system/tool_use_summary）
  | TombstoneMessage     // 流式回退时作废的旧消息
  | ToolUseSummaryMessage, // 工具调用摘要（用于压缩后展示）
  Terminal               // 最终退出原因
>
```

## 3.2 QueryEngine：多轮会话状态管理

`src/QueryEngine.ts`（1,295 行）是 `query()` 的多轮封装层，面向 SDK 消费者：

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]          // 完整对话历史
  private abortController: AbortController    // 用于中断
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage        // 累计 token 用量
  private readFileState: FileStateCache       // 跨轮文件读取缓存（LRU）

  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage, void, unknown>
}
```

**每轮 `submitMessage()` 的职责**：

1. 将用户消息追加到 `mutableMessages`
2. 重建 `ToolUseContext`（含当前消息、文件缓存、权限上下文）
3. 调用 `query()` AsyncGenerator 消费所有事件
4. 累计 `totalUsage`（跨轮叠加）
5. 将完整 transcript 持久化到磁盘（若启用会话持久化）
6. 追踪权限拒绝次数（用于 auto 模式熔断）
7. 检查预算上限（`maxBudgetUsd`）
8. 生成 SDK 格式的结果消息

**跨轮保留 vs 每轮重建**：

| 状态 | 跨轮保留 | 每轮重建 |
|------|---------|---------|
| `mutableMessages` | ✓ | — |
| `readFileState`（文件 LRU 缓存） | ✓ | — |
| `totalUsage` | ✓ | — |
| `permissionDenials` | ✓ | — |
| `ToolUseContext`（含权限快照） | — | ✓ |
| `ToolPermissionContext` | — | ✓ |
| Skill 发现缓存 | — | ✓ |

## 3.3 query()：单轮执行的完整生命周期

`queryLoop()` 的 `while(true)` 循环由 7 个阶段构成：

### 状态对象

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number      // 最大输出 token 恢复计数
  hasAttemptedReactiveCompact: boolean      // 是否已尝试响应式压缩
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number                          // 当前轮次
  transition: Continue | undefined           // 上一次迭代的转移原因（用于测试断言）
}
```

`transition` 字段记录"为什么要再来一轮"，所有可能的值：

| 转移原因 | 触发场景 |
|---------|---------|
| `next_turn` | 有工具调用结果，正常继续 |
| `reactive_compact_retry` | 响应式压缩后重试 |
| `max_output_tokens_escalate` | 输出 token 上限升级（8k → 64k） |
| `max_output_tokens_recovery` | 多轮恢复注入（最多 3 次） |
| `stop_hook_blocking` | Stop Hook 阻塞 |
| `token_budget_continuation` | Token 预算续写提示 |
| `collapse_drain_retry` | Context Collapse 后重试 |

### Phase 1：Setup

```typescript
yield { type: 'stream_request_start' }

// 初始化 StreamingToolExecutor（若启用）
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(tools, canUseTool, toolUseContext)
  : null

// 重置每轮变量
let assistantMessages: AssistantMessage[] = []
let toolResults: (UserMessage | AttachmentMessage)[] = []
let toolUseBlocks: ToolUseBlock[] = []
let needsFollowUp = false
```

### Phase 2：Context 准备（压缩管线）

```typescript
// 应用工具结果预算（Tool Output Budget）
let messagesForQuery = applyToolResultBudget(messages)

// L1: Snip Compact（若启用 HISTORY_SNIP）
if (feature('HISTORY_SNIP')) {
  messagesForQuery = snipCompact(messagesForQuery)
}

// L2: MicroCompact（每轮都执行）
messagesForQuery = deps.microcompact(messagesForQuery)

// L3: Context Collapse（按需，约 85% 窗口时触发）
if (feature('CONTEXT_COLLAPSE') && shouldCollapse) {
  await contextCollapse.collapse(messagesForQuery)
}

// 检查 token 数量是否触发 L4 AutoCompact
await deps.autocompact(messagesForQuery, autoCompactTracking)
```

### Phase 3：API 流式调用

```typescript
for await (const message of deps.callModel({
  messages: messagesForQuery,
  systemPrompt,
  tools: sortedTools,           // 按名称排序，保护 KV Cache
  maxOutputTokens: maxOutputTokensOverride,
  // ...
})) {
  yield message  // 立即透传给消费者

  if (message.type === 'assistant') {
    for (const toolBlock of message.content.filter(c => c.type === 'tool_use')) {
      // 边收边执行：不等 API 响应结束就开始执行工具！
      streamingToolExecutor?.addTool(toolBlock, message)
    }
  }

  // 错误扣押：发现恢复场景时不立即 yield 错误
  if (isRecoverableError(message)) {
    withholdError(message)
    break
  }
}
```

**"边收边执行"的意义**：LLM 响应是流式的，`tool_use` 块通常在响应前段出现。传统方式需等响应结束再执行工具。`StreamingToolExecutor.addTool()` 在收到工具调用时立即开始执行，从而将工具执行时间与 LLM 生成时间**重叠**，降低体感延迟。

### Phase 4：错误恢复

详见 Chapter 12。

### Phase 5：工具执行

```typescript
// 选择执行路径
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()  // 流式执行器：等待剩余结果
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)  // 同步执行

for await (const update of toolUpdates) {
  yield update.message  // 逐条产出工具执行结果
  toolResults.push(...normalizeMessagesForAPI([update.message], tools))
}
```

### Phase 6：附件与记忆预取

```typescript
// 获取文件变更、任务通知等附件消息
const attachments = await getAttachmentMessages(context)

// 消费记忆预取结果（若已 settle）
if (memoryPrefetch && isSettled(memoryPrefetch)) {
  const memories = await memoryPrefetch
  attachments.push(...memories)
}
```

### Phase 7：转移判断

```typescript
if (!needsFollowUp && !hasStopHookErrors) {
  // 没有工具调用，对话结束
  return { reason: 'completed' }
}

// 构建下一轮状态
state = {
  ...state,
  messages: [...state.messages, ...assistantMessages, ...toolResults, ...attachments],
  turnCount: state.turnCount + 1,
  transition: { reason: 'next_turn' },
}
// continue → 下一次 while(true) 迭代
```

## 3.4 消息归一化与 System Prompt 构建

在每轮 API 调用前，消息历史经过归一化处理：

```typescript
// 归一化：确保消息格式符合 API 要求
const normalizedMessages = normalizeMessagesForAPI(messages, tools)

// System Prompt 按 Section 组装（详见 Chapter 11）
const systemPrompt = await getSystemPrompt(tools, model, additionalDirs, mcpClients)
```

## 3.5 Token 预算控制（src/query/tokenBudget.ts，94 行）

Token 预算系统有两个独立维度：

**维度一：每轮 Token 预算（防止无限续写）**

```typescript
type BudgetTracker = {
  continuationCount: number      // 连续续写次数
  lastDeltaTokens: number        // 上次增量
  lastGlobalTurnTokens: number
  startedAt: number
}

function checkTokenBudget(tracker, agentId, budget, globalTurnTokens): TokenBudgetDecision {
  const pct = Math.round((turnTokens / budget) * 100)
  const isDiminishing = continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD  // <500 tokens 边际递减
  
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    return { action: 'continue', nudgeMessage }
  }
  return { action: 'stop', completionEvent }
}
```

**维度二：任务总预算（API 级成本控制）**

```typescript
// 压缩前记录基准
const preCompactContext = getTotalTokensUsed()

// 每轮扣减：传给 callModel 的实际预算
const remainingBudget = taskBudget - (totalTokens - preCompactContext)
```

## 3.6 Stop Hooks 与 Post-Sampling 扩展点（src/query/stopHooks.ts，474 行）

Stop Hooks 在**模型输出结束后、工具执行前**触发，可以：

1. 产出进度消息（跟踪 Hook 计数）
2. 产出附件消息（`hook_success`、`hook_error_during_execution`）
3. 设置 `preventContinuation = true`（阻止循环继续）
4. 返回 `blockingErrors`（注入阻塞错误 → 触发 `stop_hook_blocking` 转移）
5. 执行 TaskCompleted Hooks（对 `in_progress` 任务）
6. 执行 TeammateIdle Hooks（若当前 Agent 是队友）

```typescript
const stopHookResult = yield* handleStopHooks(
  messagesForQuery, assistantMessages, systemPrompt,
  userContext, systemContext, toolUseContext, querySource, stopHookActive
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  state = { ...state, stopHookActive: true, transition: { reason: 'stop_hook_blocking' } }
  continue
}
```

## 3.7 Cost 追踪（src/cost-tracker.ts，324 行）

```typescript
// 每次 API 调用后累加
addToTotalSessionCost(cost, usage, model)

// 会话结束时持久化（供下次启动展示）
saveCurrentSessionCosts(fpsMetrics)

// 从上次会话恢复（用于 /resume 场景）
restoreCostStateForSession(sessionId)

// 格式化显示
formatTotalCost()  // → "$0.0123"
```

Cost 状态按 `sessionId` 隔离存储，`/resume` 时可恢复历史成本计数。

---

<a name="chapter-4"></a>
# Chapter 4 — 工具执行系统：StreamingToolExecutor + 智能并发

## 4.1 Tool 接口与 buildTool() 工厂模式（src/Tool.ts，792 行）

### 核心 Tool 接口

```typescript
type Tool<Input, Output, P extends ToolProgressData> = {
  name: string
  aliases?: string[]
  searchHint?: string          // 3-10 词的能力描述（用于 ToolSearch）

  // 执行
  call(args, context: ToolUseContext, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>

  // Schema
  inputSchema: Input           // Zod schema
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number   // Infinity = 永不持久化

  // 并发与安全声明
  isConcurrencySafe(input): boolean   // false = 独占访问
  isReadOnly(input): boolean
  isDestructive?(input): boolean

  // 权限生命周期
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>

  // 中断行为
  interruptBehavior?(): 'cancel' | 'block'  // 默认 'block'

  // 结果格式化
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode

  // UI 渲染
  userFacingName(input): string
  renderToolUseMessage(input, options): React.ReactNode
}
```

### ToolResult 类型

```typescript
type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext  // 仅非并发工具可用
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}
```

### buildTool() 工厂：失败安全的默认值

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // 失败安全：默认假设不可并发
  isReadOnly: () => false,           // 失败安全：默认假设有写操作
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',   // 默认跳过安全分类器
  userFacingName: () => '',          // 回退到 tool.name
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def } as BuiltTool<D>
}
```

`TOOL_DEFAULTS` 的设计遵循**失败安全原则**：
- `isConcurrencySafe: false` → 默认串行，防止并发 bug
- `isReadOnly: false` → 默认假设写操作，避免被过于宽松的权限策略通过

`buildTool()` 在类型层面通过 `BuiltTool<D>` 条件类型保证：**每个工具调用后都拥有所有必需方法**，消费方无需 `?.()` 判空。

## 4.2 工具注册总入口（src/tools.ts，30KB）

**getAllBaseTools()** 返回所有内置工具的主列表（50+ 个），通过 `feature()` 门控：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    // ...
    ...(feature('REPL_MODE') ? [REPLTool] : []),
    ...(feature('WORKTREE_MODE') ? [EnterWorktreeTool, ExitWorktreeTool] : []),
  ]
}
```

**assembleToolPool()** 合并内置工具与 MCP 工具，并排序以稳定 KV Cache：

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  // 内置工具和 MCP 工具分别排序，保持内置工具为连续前缀
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

> **KV Cache 关键**：工具列表分区排序确保内置工具构成稳定的 Cache Prefix。详见 Chapter 20。

## 4.3 StreamingToolExecutor：边收边执行（src/services/tools/StreamingToolExecutor.ts，530 行）

### 架构设计

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>          // executeTool() 的 Promise
  results?: Message[]              // 工具执行结果
  pendingProgress: Message[]       // 立即 yield，不排队
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 并发调度器：canExecuteTool()

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||  // 无执行中 → 任何工具均可运行
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
    // 并发安全工具 + 所有执行中工具均并发安全 → 并行
  )
}
```

**调度不变量**：

| 场景 | 行为 |
|------|------|
| 队列为空 | 任何工具立即执行 |
| 并发工具 vs 并发工具 | 并行执行 |
| 非并发工具 | 等待所有执行中工具完成，独占执行 |
| 非并发工具执行中 | 后续工具（包括并发安全工具）全部排队等待 |

## 4.4 智能并发：isConcurrencySafe 双队列调度

### addTool()：进入调度系统

```typescript
addTool(toolBlock: ToolUseBlock, assistantMessage: AssistantMessage): void {
  const toolDef = this.tools.find(t => t.name === toolBlock.name)
  const isConcurrencySafe = toolDef?.isConcurrencySafe(toolBlock.input) ?? false

  const tracked: TrackedTool = {
    id: toolBlock.id,
    block: toolBlock,
    assistantMessage,
    status: 'queued',
    isConcurrencySafe,
    pendingProgress: [],
  }

  this.tools.push(tracked)
  void this.processQueue()  // 异步触发，不阻塞调用方
}
```

### processQueue()：调度循环

```typescript
private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue

    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)  // 启动执行（不等完成）
    } else {
      if (!tool.isConcurrencySafe) break  // 非并发工具阻塞队列
    }
  }
}
```

### 错误级联：Bash 错误中止兄弟工具

```typescript
if (isErrorResult && tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
  // Bash 错误才触发级联，Read/WebFetch 错误不触发
}
```

**设计理由**：Shell 命令通常有隐式依赖（`mkdir` 失败 → 后续命令无意义）；而文件读取、Web 请求通常是独立操作，失败不影响兄弟任务。

### 结果有序输出：getCompletedResults()

```typescript
*getCompletedResults(): Generator<MessageUpdate, void> {
  for (const tool of this.tools) {
    // 进度消息：立即 yield，不排队
    while (tool.pendingProgress.length > 0) {
      yield { message: tool.pendingProgress.shift()!, newContext: this.toolUseContext }
    }

    if (tool.status === 'yielded') continue
    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'
      for (const message of tool.results) yield { message }
    } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
      break  // 停在执行中的非并发工具，维持结果顺序
    }
  }
}
```

**结果顺序保证**：结果按工具**接收顺序**输出，而非完成顺序。这确保了消息历史的确定性，对 KV Cache 稳定性至关重要。

## 4.5 核心工具实现详解

### 工具属性速查表

| 工具 | isConcurrencySafe | isReadOnly | maxResultSizeChars | 关键特性 |
|------|:-----------------:|:----------:|:------------------:|---------|
| Read | ✓ | ✓ | Infinity | mtime 去重；拦截 /dev/zero 等危险设备 |
| Edit | ✗ | ✗ | 100K | 原子性读-改-写；mtime 过期检查 |
| Write | ✗ | ✗ | 100K | 文件历史备份；LSP 通知 |
| Glob | ✓ | ✓ | 100K | 结果上限 100；路径相对化 |
| Grep | ✓ | ✓ | 100K | ripgrep 集成；head_limit 250 |
| Bash | ✗ | 启发式 | 100K | 2s 进度 spinner；15s 自动后台化 |
| WebFetch | ✓ | ✓ | 100K | Deferred 工具；预批准主机 |
| Agent | ✗ | 依赖 | 100K | 派生子 Agent |
| MCPTool | 依赖声明 | 依赖声明 | 100K | 运行时覆盖 |

### FileReadTool 关键设计

```typescript
// maxResultSizeChars: Infinity — 永不持久化
// 原因：持久化会导致 Read→文件→Read 死循环
maxResultSizeChars: Infinity

// 去重逻辑：相同文件/偏移/限制 + mtime 未变 → 返回 FILE_UNCHANGED_STUB
// 减少重复内容在 context 中的占用

// 拦截危险设备（会产生无限输出或阻塞输入）
const BLOCKED_DEVICES = ['/dev/zero', '/dev/urandom', '/dev/stdin', '/dev/tty']
```

支持的文件类型：
- 文本文件（带行号的 `cat -n` 格式）
- 图像（base64 编码，支持 JPEG/PNG/GIF/WebP）
- Jupyter Notebook（`.ipynb`，提取单元格内容）
- PDF（逐页提取文本和视觉内容）
- 大文件（超过阈值时分块输出）

### BashTool 关键设计

```typescript
const PROGRESS_THRESHOLD_MS = 2000    // 2s 后显示 spinner
const ASSISTANT_BLOCKING_BUDGET_MS = 15_000  // 15s 后自动后台化

// 搜索/读取命令启发式识别（用于 UI 折叠显示）
const BASH_SEARCH_COMMANDS = new Set(['find', 'grep', 'rg', 'ag', ...])
const BASH_READ_COMMANDS = new Set(['cat', 'head', 'tail', 'jq', ...])
// 语义中性命令（如 echo, printf）在管道中被忽略
const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set(['echo', 'printf', ...])
```

## 4.6 ToolUseContext：工具执行的运行时上下文

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }

  abortController: AbortController    // 轮级中断信号
  readFileState: FileStateCache       // 跨工具文件读取缓存

  // 状态访问（React Context 风格）
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  // UI 回调
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void

  // 追踪
  updateFileHistoryState: (updater: ...) => void
  updateAttributionState: (updater: ...) => void

  // 子 Agent 上下文
  agentId?: AgentId
  agentType?: string
  messages: Message[]
}
```

---

<a name="chapter-5"></a>
# Chapter 5 — 权限与安全模型

## 5.1 权限系统设计目标与三层模型

Claude Code 的权限系统目标是：**在安全约束下给 Agent 最大行动自由**，同时防止意外的破坏性操作。

三层模型：

```
Layer 1: Input Validation (validateInput)
  └── 格式校验、必填字段、类型检查

Layer 2: Tool-Specific Permission Check (checkPermissions)
  └── 工具自身的业务逻辑检查（如路径白名单）

Layer 3: General Permission System
  └── ToolPermissionContext 规则引擎
      ├── Deny Rules → 直接拒绝
      ├── Ask Rules → 要求用户确认
      ├── Allow Rules → 自动批准
      └── Mode-based → default/auto/bypass
```

## 5.2 ToolPermissionContext：不可变快照

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean    // 后台 Agent 使用
  awaitAutomatedChecksBeforeDialog?: boolean // Coordinator Worker 使用
  prePlanMode?: PermissionMode
}>
```

整个对象深度不可变（`DeepImmutable`），防止 Agent 在运行时篡改权限规则。每轮 `submitMessage()` 创建新快照，确保权限检查基于稳定状态。

## 5.3 Permission Mode：五种模式

```typescript
// 用户可见模式（外部）
type ExternalPermissionMode = 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'

// 内部运行时模式（含 'auto' 和 'bubble'）
type PermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

| 模式 | 行为 | 典型场景 |
|------|------|---------|
| `default` | 所有操作询问用户 | 默认交互模式 |
| `acceptEdits` | 文件编辑自动允许，其他询问 | 信任代码编辑 |
| `dontAsk` | 'ask' 行为转换为 'deny' | 无人值守模式 |
| `auto` | ML 分类器决定 allow/deny | 自动模式（GrowthBook 门控） |
| `plan` | 只允许读操作，写操作锁定 | 计划模式 |
| `bypassPermissions` | 允许所有（含危险操作） | 明确信任场景 |
| `bubble` | 权限请求冒泡到父 Agent | 子 Agent 使用 |

## 5.4 规则系统：ToolPermissionRulesBySource

规则按**来源**分类，优先级从高到低：

```
policySettings > localSettings > userSettings > projectSettings > flagSettings > cliArg > command > session
```

每条规则的结构：

```typescript
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string         // 工具名，支持通配符
    ruleContent?: string     // 可选内容模式（如 "npm publish:*"）
  }
}
```

**匹配模式**：

```
整个工具:        toolName: "Bash"
内容级别:        toolName: "Bash", ruleContent: "npm publish:*"
MCP 服务器级:    toolName: "mcp__server1"  → 匹配 server1 的所有工具
MCP 通配符:      toolName: "mcp__server1__*"
```

## 5.5 权限决策流程

```
工具调用到达
     │
     ▼
validateInput()
     │  ← 语法/格式校验失败 → 拒绝
     ▼
checkPermissions()（工具自身逻辑）
     │
     ▼
通用权限系统
     ├─ alwaysDenyRules 匹配？→ deny
     ├─ alwaysAskRules 匹配？→ ask（除非 sandboxOverride）
     ├─ mode == 'bypassPermissions'？→ allow
     ├─ alwaysAllowRules 匹配？→ allow
     ├─ mode == 'auto'？→ 调用 ML 分类器
     │    ├─ classifier: "safe" → allow
     │    ├─ classifier: "risky" → ask
     │    └─ classifier: error/unavailable → 降级询问
     └─ mode == 'default'？→ ask
```

**PermissionDecision** 的四种结果：

```typescript
type PermissionDecision =
  | { behavior: 'allow'; updatedInput?; decisionReason? }
  | { behavior: 'ask'; message; suggestions?; pendingClassifierCheck? }
  | { behavior: 'deny'; message; decisionReason }
  | { behavior: 'passthrough'; message; decisionReason? }
```

`decisionReason` 是鉴别联合类型，记录决策来源：`'rule' | 'mode' | 'classifier' | 'hook' | 'workingDir' | 'safetyCheck' | ...`

## 5.6 ML 安全分类器（src/jobs/classifier.ts）

在 `auto` 模式下，权限系统调用 `classifyYoloAction()` 进行两阶段 ML 分类：

```typescript
type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean    // 上下文太长，降级到手动
  model: string
  stage?: 'fast' | 'thinking'   // 两阶段管道
  // Stage 1 指标
  stage1Usage?, stage1DurationMs?, stage1RequestId?
  // Stage 2 指标（thinking 阶段）
  stage2Usage?, stage2DurationMs?, stage2RequestId?
}
```

**熔断器**：连续 3 次分类器失败后停止重试，防止 API 调用浪费。全局统计显示熔断机制每天可节约约 25 万次 API 调用。

## 5.7 Denial Tracking 与升级机制

```typescript
// 连续拒绝上限：3 次
// 总拒绝上限：20 次
// 触发条件：超出阈值后提示用户检查权限配置
```

**本地可变状态**（子 Agent 无法修改父 Agent 的拒绝计数器）：

```typescript
// ToolPermissionContext 顶层不可变
// 但 localDenialTracking 允许在该层次进行局部追踪
localDenialTracking?: DenialTrackingState
```

## 5.8 Sandbox 模式与 Policy Limits

**Sandbox 激活条件**：

```
IS_SANDBOX env var = '1'
OR CLAUDE_CODE_BUBBLEWRAP = '1'
```

**权限验证**（`setup.ts`）：

- Unix 系统：`bypassPermissions` 不允许以 root/sudo 运行（除非在 sandbox）
- ANT 构建：验证 Docker/sandbox 环境
- BYOC 入口：跳过沙箱检查（外部合作伙伴自行负责安全）

**Policy Limits**（`src/services/policyLimits/`）：

远程管理的速率限制配置，通过 `waitForPolicyLimitsToLoad()` 在启动时加载，`isPolicyAllowed('allow_remote_control')` 在 Bridge 模式下门控功能访问。

---

<a name="chapter-6"></a>
# Chapter 6 — Hook 系统

## 6.1 Hook 扩展点设计哲学

> **"Hook around the loop, never rewrite the loop"**

Hook 系统允许用户在 Agent Loop 的关键节点注入自定义逻辑，但不修改主循环本身。这是 Harness 工程的核心原则之一：让 Harness 可扩展，而不是可替换。

## 6.2 四种 Hook 类型（src/schemas/hooks.ts）

### 1. Command Hook（Shell 命令）

```typescript
{
  type: 'command'
  command: string          // Shell 命令
  if?: string              // 权限规则过滤（如 "Bash(git *)"）
  shell?: 'bash' | 'powershell'
  timeout?: number         // 秒
  statusMessage?: string   // Spinner 文本
  once?: boolean           // 执行后自动移除
  async?: boolean          // 后台执行
  asyncRewake?: boolean    // 退出码 2 时唤醒 Agent（隐含 async）
}
```

**示例**：格式化保存 Hook

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "command",
      "command": "prettier --write $TOOL_INPUT_FILE_PATH"
    }]
  }]
}
```

### 2. Prompt Hook（LLM 提示）

```typescript
{
  type: 'prompt'
  prompt: string           // LLM 提示，含 $ARGUMENTS 占位符
  if?: string
  timeout?: number
  model?: string           // 默认使用快速小模型
  statusMessage?: string
  once?: boolean
}
```

### 3. HTTP Hook（Webhook）

```typescript
{
  type: 'http'
  url: string              // POST 目标
  if?: string
  timeout?: number
  headers?: Record<string, string>  // 支持 $VAR 环境变量插值
  allowedEnvVars?: string[]         // 环境变量插值白名单
  statusMessage?: string
  once?: boolean
}
```

### 4. Agent Hook（Agent 验证）

```typescript
{
  type: 'agent'
  prompt: string           // 验证提示，含 $ARGUMENTS 占位符
  if?: string
  timeout?: number         // 默认 60s
  model?: string           // 默认 Haiku
  statusMessage?: string
  once?: boolean
}
```

## 6.3 Hook 事件与触发时机

| 事件 | 触发位置 | 说明 |
|------|---------|------|
| `SessionStart` | `setup.ts` 初始化后 | 会话开始，每个会话仅触发一次 |
| `PreToolUse` | 工具执行前 | 可修改输入、阻止执行 |
| `PostToolUse` | 工具执行后 | 可修改输出、触发后处理 |

`PermissionRequest` 事件用于自定义权限审批流（如 CI 环境中的审批队列）。

## 6.4 Hook 执行机制与 `if` 过滤

`if` 字段使用与权限规则相同的语法，在 Fork 子进程前先做过滤：

```typescript
// 仅对 "Bash(git *)" 类型的工具调用触发
{ "if": "Bash(git *)", "type": "command", "command": "..." }

// 对所有 Write 或 Edit 工具触发
{ "matcher": "Write|Edit", ... }
```

`if` 过滤在 Hook 执行前检查，避免不必要的子进程创建。

**错误隔离**：单个 Hook 失败不会崩溃主循环，错误被捕获并记录为 `hook_error_during_execution` 附件消息。

---

<a name="chapter-7"></a>
# Chapter 7 — 任务规划与命令系统

## 7.1 命令类型体系

Claude Code 的命令系统支持四种命令类型：

```typescript
type Command =
  | PromptCommand      // 向 LLM 发送提示（/commit、/review）
  | LocalCommand       // 同步本地操作（/clear）
  | LocalJSXCommand    // React UI 面板（/model、/config）
  | LocalMCPCommand    // MCP 工具调用
  | LocalPluginCommand // 插件命令
```

### PromptCommand

```typescript
type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  progressMessage: string
  contentLength: number              // 字符估算（token 计数用）
  getPromptForCommand(args, context): ContentBlockParam[]

  allowedTools?: string[]            // 限制 LLM 可使用的工具
  model?: string                     // 覆盖模型
  source: 'builtin' | 'mcp' | 'plugin' | 'bundled' | SettingSource
  hooks?: HooksSettings              // 调用时注册的 Hooks
  context?: 'inline' | 'fork'       // inline=展开到对话 / fork=子 Agent 执行
  paths?: string[]                   // Glob 模式，条件激活
  disableModelInvocation?: boolean   // 不显示在 LLM 工具列表中
  userInvocable?: boolean
  immediate?: boolean
}
```

### LocalJSXCommand

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  name: string
  load: () => Promise<LocalJSXCommandModule>  // 懒加载 React 组件
}

type LocalJSXCommandModule = {
  call(
    onDone: (result, options?: { display?; shouldQuery?; metaMessages?; nextInput? }) => void,
    context: CommandContext,
    args: string,
  ): React.ReactNode
}
```

## 7.2 静态命令注册（commands.ts）

**主注册表**（Memoized，约 120+ 条命令）：

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, brief, bughunter,
  clear, claudeApi, commit, commitPushPr, config,
  // ...80+ 更多命令
  version,
], () => getProjectRoot())  // 按 projectRoot 缓存键
```

**Memoize 键**：`getProjectRoot()`。当工作目录变更时（如进入 Worktree），命令列表重新计算。

**Feature Gate 裁剪**：

```typescript
const proactive = (feature('PROACTIVE') || feature('KAIROS'))
  ? require('./commands/proactive.js').default : null

const COMMANDS = memoize(() => [
  // ...
  ...(proactive ? [proactive] : []),
])
```

**仅内部命令**（ANT 构建）：

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, debugToolCall, // ...
]
// 外部构建时打包工具排除这些命令
```

## 7.3 动态命令：Skill/Plugin 运行时注入

```typescript
async function getSkillToolCommands(cwd: string): Promise<Command[]> {
  const [
    skillDirCommands,      // ~/项目/.claude/skills/ 下的技能文件
    pluginSkills,          // 插件清单中的技能
    bundledSkills,         // 启动时注册的内置技能
    builtinPluginSkills,   // 内置插件的技能
  ] = await Promise.all([
    getSkillDirCommands(cwd),
    getPluginSkills(),
    getBundledSkills(),
    getBuiltinPluginSkillCommands(),
  ])

  return [...skillDirCommands, ...pluginSkills, ...bundledSkills, ...builtinPluginSkills]
}
```

## 7.4 命令可用性过滤

```typescript
type CommandAvailability = 'claude-ai' | 'console'
// claude-ai: OAuth 订阅用户（Pro/Max/Team/Enterprise）
// console: 直接 Console API Key 用户（api.anthropic.com）
```

Bedrock、Vertex、自定义 base URL 的用户看不到 `claude-ai` 和 `console` 限定的命令。

## 7.5 懒加载大型命令

```typescript
// insights.ts 约 113KB（3,200 行），仅在调用时加载
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    return real.getPromptForCommand(args, context)
  }
}
```

## 7.6 计划先行：TodoWrite 与 plan-then-execute 模式

Claude Code 内置了 `TodoWrite` 工具，实现"先计划、再执行"的工作方式：

```typescript
// TodoWrite 的输入 Schema
type TodoItem = {
  id: string
  content: string
  status: 'pending' | 'in_progress' | 'completed'
  priority: 'high' | 'medium' | 'low'
}
```

Agent 在执行复杂任务前，通常先调用 `TodoWrite` 创建任务清单，然后逐项执行并更新状态。这对应 `s05_todo_write` 课程的核心机制，研究表明"先计划"使任务完成率提升约一倍。

---

<a name="chapter-8"></a>
# Chapter 8 — 子 Agent 与上下文隔离

## 8.1 子 Agent 派发：AgentTool

AgentTool（`src/tools/AgentTool/`）是创建子 Agent 的主要机制：

```typescript
type AgentToolInput = {
  description: string    // 子 Agent 的任务描述
  prompt: string         // 初始提示
  // 可选：指定 Agent 类型（general-purpose / Explore / Plan / fork）
}
```

**fork 模式**：继承父 Agent 的完整上下文（system prompt、工具列表、对话历史），适合需要共享知识的场景。

**独立模式**：创建全新的 `messages[]` 数组，上下文完全隔离，带回执行结果。

## 8.2 fresh messages[]：上下文隔离的核心机制

```typescript
// 子 Agent 启动时：全新的空消息历史
const subAgentMessages: Message[] = []

// 父 Agent 的 tools、model 等通过 CacheSafeParams 传递
const subAgent = new QueryEngine({
  ...parentCacheSafeParams,   // 共享 system prompt 和工具列表
  messages: subAgentMessages, // 独立的消息历史
})
```

**为什么隔离很重要**：子 Agent 执行的大量工具调用（文件读取、搜索结果）不应污染父 Agent 的上下文窗口，子 Agent 只需向父 Agent 返回最终结论。

## 8.3 CacheSafeParams：父子 Agent 复用 System Prompt + 工具列表

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]   // Fork 模式下的父上下文
}
```

`CacheSafeParams` 确保父子 Agent 的 API 请求前缀相同，从而共享 KV Cache：

- **System Prompt** 相同 → Cache Hit
- **工具列表**（按名称排序）相同 → Cache Hit
- **Model** 相同 → Cache Hit

> **警告**：在 fork 模式下为子 Agent 设置不同的 `maxOutputTokens` 会改变 `budget_tokens`，导致 Cache Miss。

## 8.4 Coordinator 模式（src/coordinator/）

```typescript
// coordinatorMode.ts：协调器模式入口
// workerAgent.ts：Worker Agent 通信

// 协调器模式下的特殊权限设置
{ awaitAutomatedChecksBeforeDialog: true }
// 含义：在弹出权限对话框前，先等待自动化检查完成
```

Coordinator 模式支持一个 Agent 管理多个 Worker Agent 的工作流，通过共享任务列表协调工作。

---

<a name="chapter-9"></a>
# Chapter 9 — 上下文管理：五级递进压缩

> **这是 Claude Code 里最精妙的工程设计之一。** 不是一上来就全量压缩，而是从轻到重逐级尝试，每一级都在尽可能保护 KV Cache。

## 9.1 为什么需要分级压缩

LLM 的上下文窗口是有限物理资源（如 200K tokens）。一次典型的代码工程会话可能包含：
- 数十次文件读取（每次 1-5K tokens）
- 大量 grep/bash 输出（每次 5-50K tokens）
- 多轮对话历史

单纯截断历史会丢失重要信息；全量重新摘要代价高昂且破坏 Cache。五级递进策略在**保护信息完整性**和**最小化 LLM 调用成本**之间取得平衡。

## 9.2 L0 — Tool Output Budget：磁盘持久化与预览截断

**永远发生，零 LLM 开销。**

### 双层预算

```typescript
// src/constants/toolLimits.ts
const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000       // 单工具输出上限：5 万字符
const MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000 // 单条 user message 总输出上限：20 万字符
const PREVIEW_SIZE_BYTES = 2_000                   // 持久化后保留的预览大小
```

**Per-Tool 预算**：每个工具声明 `maxResultSizeChars`，实际上限取 `Math.min(工具声明值, 50_000)`。特例：
- `FileReadTool`: `Infinity`（永不持久化，避免读取死循环）
- `BashTool`: 100K 字符
- `GrepTool`: 100K 字符

**Per-Message 聚合预算**：一次并行工具调用（同一 `user` 消息）的所有输出之和超过 20 万字符时，从最大结果开始贪心地依次持久化，直到总量满足预算。

### 持久化实现（src/utils/toolResultStorage.ts）

```typescript
async function maybePersistLargeToolResult(toolResultBlock, toolName) {
  const size = contentSize(content)
  const threshold = getPersistenceThreshold(toolName, tool.maxResultSizeChars)

  if (size <= threshold) return toolResultBlock  // 未超出，原样返回

  // 超出：写入 {sessionDir}/tool-results/{tool_use_id}.txt
  // flag: 'wx'（文件存在则跳过）—— 防止 MicroCompact 重放历史时重复写入
  const result = await persistToolResult(content, toolResultBlock.tool_use_id)

  // 上下文中替换为预览消息
  return {
    ...toolResultBlock,
    content: buildLargeToolResultMessage(result)
    // "<persisted-output>\nOutput too large (XXX chars). Full output saved to: {path}\nPreview:\n..."
  }
}
```

**Prompt Cache 不变量**：已见过的 `tool_use_id` 的处理结果（持久化 or 不持久化）被冻结，重放历史时不改变命运，维持 Cache 稳定性。

## 9.3 L1 — Snip Compact：裁剪旧消息保留 Cache Prefix

**每轮执行，零 LLM 开销。**（feature(`HISTORY_SNIP`) 门控）

Snip Compact 从消息历史前端裁剪旧内容，使历史适配当前 context 窗口。裁剪方向选择 `'from'`（从前保留），保护已经缓存的消息 Prefix，最大化 KV Cache 命中。

## 9.4 L2 — MicroCompact：按时间衰减压缩 tool_result

**每轮执行，零 LLM 开销。**

```typescript
// src/services/compact/microCompact.ts
const COMPACTABLE_TOOLS = new Set([
  'File read', 'Shell tools', 'Grep', 'Glob',
  'Web search', 'Web fetch', 'File edit', 'File write'
])

const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

**时间衰减算法**：根据消息时间戳，越旧的 `tool_result` 压缩得越狠：
- 最新 N 条：保留完整内容
- 较旧：保留内容摘要
- 很旧：替换为 `TIME_BASED_MC_CLEARED_MESSAGE`

`CACHED_MICROCOMPACT` 特性启用时，还支持基于 Cache Prefix 的 MicroCompact，专门压缩已被 Cache 保护的消息段（因为 Cache 命中时 LLM 仍能"看到"这些内容，压缩后的占位符不影响模型理解）。

## 9.5 L3 — Context Collapse：约 85% 窗口触发

**按需触发，一次 LLM 调用。**（feature(`CONTEXT_COLLAPSE`) 门控）

```typescript
// 触发阈值
const threshold = getEffectiveContextWindowSize(model) * 0.85

// 折叠式压缩：生成结构化摘要，保留关键细节
// 与 AutoCompact 的区别：更保守，保留更多细节
await contextCollapse.collapse(messagesForQuery)
```

Context Collapse 尝试以**最小信息损失**压缩对话历史。实现方式是向 LLM 发送压缩指令，要求生成结构化摘要（含已完成任务、关键发现、待处理事项）。

**Collapse Drain**：在 `prompt_too_long` 错误发生后，优先尝试 Collapse 而非直接 Compact，因为 Collapse 的信息保留更好。

## 9.6 L4 — AutoCompact：约 90% 窗口触发，六阶段流水线

**按需触发，较重 LLM 调用。**

### 触发阈值计算

```typescript
function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 预留 13,000 tokens
}

function getEffectiveContextWindowSize(model: string): number {
  const reservedForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY  // 上限 20,000
  )
  return contextWindow - reservedForSummary
}
```

**强制压缩**（97%+ 阈值）：

```typescript
if (tokenUsage > 0.97) {
  await forceAutoCompact(messages)  // 阻塞！必须压缩才能继续
} else if (tokenUsage > 0.90) {
  if (autoCompactCircuitBreaker.isOpen()) return  // 熔断器打开则跳过
  await autoCompact(messages)
}
```

### 六阶段流水线

```
Stage 1: PreCompact Hooks    ← 压缩前清理、日志
Stage 2: compactConversation ← 主摘要生成（LLM 调用）
Stage 3: trySessionMemoryCompaction ← 会话记忆压缩（KAIROS 门控）
Stage 4: PostCompact Hooks   ← 压缩后验证、通知
Stage 5: runPostCompactCleanup ← 状态重置
Stage 6: Analytics           ← tokenStatsToStatsigMetrics()
```

## 9.7 熔断器保护（CircuitBreaker）

```typescript
type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  turnId: string                // 每轮唯一 ID
  consecutiveFailures?: number  // 连续失败次数，成功后重置
}

const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 连续失败 3 次后停止触发 AutoCompact
// 背景：BQ 数据显示，全球每天约 25 万次 AutoCompact 因上下文已不可恢复而浪费
```

## 9.8 压缩方向与 KV Cache 协同

所有压缩策略统一选择 `direction: 'from'`（从前保留）：

```
原始消息历史：[MSG_1, MSG_2, MSG_3, ..., MSG_N]
                ↑ Cache Prefix（已缓存）    ↑ 最新消息

压缩方向 'from'：保留前缀 → 裁剪中段 → 保留末尾
压缩方向 'to'：保留末尾 → 裁剪前段 → 保留开头
```

`'from'` 方向保护了已被 Anthropic API Cache 保护的历史前缀，使后续请求仍能命中 Cache。这是 KV-Cache 感知设计渗透到压缩策略的体现。

---

<a name="chapter-10"></a>
# Chapter 10 — 持久化记忆系统

## 10.1 记忆三子系统

```
Memory System
├── 选择（Selection）：哪些信息值得记忆？
│   └── findRelevantMemories.ts：Sonnet 驱动的相关性评分
├── 提取（Extraction）：从对话中提取记忆
│   └── Stop Hooks 中的 EXTRACT_MEMORIES 特性
└── 整合（Consolidation）：记忆去重与合并
    └── DreamTask：定期后台整合旧会话
```

## 10.2 记忆目录架构（src/memdir/）

### 目录解析顺序

```typescript
// paths.ts
function resolveMemoryDir(): string {
  // 1. 环境变量覆盖（测试用）
  if (process.env.CLAUDE_COWORK_MEMORY_PATH_OVERRIDE) {
    return process.env.CLAUDE_COWORK_MEMORY_PATH_OVERRIDE
  }
  // 2. 设置文件中的自定义路径
  // 仅信任来源：policy/local/user（不信任 projectSettings）
  const custom = getAutoMemoryDirectory()
  if (custom) return custom
  // 3. 默认路径
  return path.join(os.homedir(), '.claude', 'projects',
    sanitizeProjectName(getProjectRoot()), 'memory')
}
```

按 `getProjectRoot()` memoize——切换 Worktree 时重新计算。

### MEMORY.md 索引入口

```
上限：200 行 OR 25KB（分别检查）
截断：先按行截断（自然边界），再按字节截断（最后换行处）
格式：- [标题](file.md) — 一行说明（每条约 150 字符）
```

超出限制时追加警告：`"197KB (limit 25KB) — index entries are too long"`

## 10.3 记忆类型四分法

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| 类型 | 作用域 | 内容 | 示例 |
|------|--------|------|------|
| `user` | 始终私有 | 用户角色、偏好、专业领域 | "用户是数据科学家，刚开始接触 React" |
| `feedback` | 默认私有 | 来自纠正和确认的工作方式指导 | "不要 mock 数据库，上季度 mock 导致漏测生产 bug" |
| `project` | 倾向团队 | 工作状态、里程碑、事件 | "合并冻结 2026-03-05，移动团队在切 release 分支" |
| `reference` | 通常团队 | 外部系统指针 | "Pipeline bug 在 Linear 项目 INGEST 中追踪" |

每类记忆包含 `**Why:**` 和 `**How to apply:**` 字段，让未来会话能做边界判断而非盲从规则。

## 10.4 相关记忆检索算法（findRelevantMemories.ts）

```typescript
async function findRelevantMemories(
  query: string,
  memoryDir: string,
  alreadySurfaced: string[] = []
): Promise<MemoryResult[]> {
  // Step 1: 扫描 200 个最新 .md 文件，读取 frontmatter
  const files = await memoryScan(memoryDir, 200)

  // Step 2: 构建清单
  // 格式：[type] filename (timestamp): description
  const manifest = formatManifest(files)

  // Step 3: 调用 Sonnet 选择相关记忆
  const selected = await callSonnetWithSchema(
    SELECT_MEMORIES_SYSTEM_PROMPT + manifest,
    { query, count: 5 }
  )

  // Step 4: 过滤已展示的记忆，最多返回 5 条
  return selected
    .filter(f => !alreadySurfaced.includes(f))
    .slice(0, 5)
    .map(filename => ({
      path: path.join(memoryDir, filename),
      mtimeMs: files.find(f => f.filename === filename)?.mtimeMs ?? 0,
    }))
}
```

### Chapter 10.5 - 记忆老化与生命周期管理（memoryAge.ts）

记忆文件的新鲜度通过 mtime（文件修改时间）计算：

- 0 天前：显示 "today"
- 1 天前：显示 "yesterday"  
- 2+ 天前：显示 "{N} days ago" 并附加新鲜度警告

新鲜度警告文本提示读者：该记忆是某时间点的快照，建议对照当前代码（file:line）验证其准确性。

### Chapter 10.6 - 团队记忆路径与安全（teamMemPaths.ts）

团队记忆存储在 `<autoMemPath>/team/` 子目录，有严格的安全校验：

```typescript
// sanitizePathKey()：拒绝危险路径
// - null 字节
// - URL 编码的路径遍历（../）
// - Unicode 归一化后的 ../
// - 反斜杠
// - 绝对路径

// realpathDeepestExisting()：检测符号链接逃逸
// 1. 找到最深存在的祖先目录
// 2. realpath() 解析该目录的真实路径
// 3. 重新拼接尾部路径
// 4. 验证结果仍在允许的根目录内
```

团队记忆有独立的 MEMORY.md 索引。当私有记忆和团队记忆都存在时，两者合并展示，使用 `<scope>` 标签区分作用域。

### Chapter 10.7 - KAIROS 日志模式

feature('KAIROS') 启用时，记忆系统切换为追加日志范式：
- 写入路径：`logs/YYYY/MM/YYYY-MM-DD.md`（追加日志，而非维护活跃 MEMORY.md）
- 夜间 /dream 技能将日志蒸馏为主题文件和 MEMORY.md
- System Prompt 中的记忆 section 通过 systemPromptSection() 缓存——跨午夜可能过期，模型从 date_change 附件推断当前日期

---

## Chapter 11 — System Prompt 运行时组装

### 11.1 Section 架构

System Prompt 不是一个硬编码字符串，而是由多个独立 Section 在运行时组装而成：

```typescript
// systemPromptSections.ts
function systemPromptSection(name: string, compute: () => Promise<string | null>) {
  // 创建带缓存的 Section（/clear 或 /compact 后重新计算）
  return { name, compute, cacheBreak: false }
}

function DANGEROUS_uncachedSystemPromptSection(name, compute, reason) {
  // 易变 Section：值变化时破坏 Cache
  // 使用场景：MCP 指令（服务器连接/断开时变化）
  return { name, compute, cacheBreak: true }
}
```

`cacheBreak: true` 的 Section 每次变化都会产生新的 Cache Key 变体（2^N 个可能组合），因此尽量减少使用。

### 11.2 完整 Section 列表（getSystemPrompt in prompts.ts）

**静态区（全局 Cache，约 4K tokens）**——位于 SYSTEM_PROMPT_DYNAMIC_BOUNDARY 之前：

| Section | 内容 |
|---------|------|
| SimpleIntro | "You are an interactive agent..." |
| SimpleSystem | 工具执行、权限提示、压缩说明 |
| DoingTasks | 代码风格、验证、不镀金原则 |
| Actions | 可逆性、爆炸半径、危险操作指导 |
| UsingYourTools | "Prefer Read/Write/Edit/Glob/Grep over Bash" |
| ToneAndStyle | emoji 指导、代码引用格式（file:line） |

**动态区（用户/会话级 Cache，约 6K tokens）**——位于 SYSTEM_PROMPT_DYNAMIC_BOUNDARY 之后：

| Section | 内容 | 缓存类型 |
|---------|------|---------|
| session_guidance | Agent/SubAgent/Skill/Verification 调用指导 | 缓存 |
| memory | MEMORY.md 内容 + 相关记忆 | 缓存 |
| env_info_simple | CWD、Git 状态、平台、模型名、知识截止日期 | 缓存 |
| mcp_instructions | MCP 服务器使用说明 | **DANGEROUS_uncached** |
| language | "Always respond in [language]" | 缓存 |
| output_style | 自定义输出风格 | 缓存 |
| scratchpad | 临时目录路径 | 缓存 |
| frc | Function Result Clearing 指令（CACHED_MICROCOMPACT） | 缓存 |
| summarize_tool_results | "记录重要信息，结果可能被清除" | 缓存 |
| numeric_length_anchors | 字数限制（ANT 内部） | 缓存 |
| token_budget | Token 预算指令（TOKEN_BUDGET） | 缓存 |

### 11.3 SIMPLE 模式（CLAUDE_CODE_SIMPLE env）

```typescript
if (process.env.CLAUDE_CODE_SIMPLE) {
  return [`You are Claude Code... CWD: ${cwd} Date: ${date}`]
  // 单字符串数组，跳过所有 Section，立即返回
}
```

### 11.4 运行时拼接流程

```typescript
async function getSystemPrompt(tools, model, additionalDirs, mcpClients): Promise<string[]> {
  const sections = await resolveSystemPromptSections()
  // Promise.all 并行计算所有 Section
  // 使用 section 缓存（仅在 /clear 或 /compact 后重置）
  return sections.filter(Boolean)  // 过滤 null Section（feature 未启用）
}
```

MCP 指令特殊处理：
- `isMcpInstructionsDeltaEnabled() = false`：整合到 system prompt
- `isMcpInstructionsDeltaEnabled() = true`：通过 mcp_instructions_delta 附件注入（增量方式，减少全量重计算）

---

## Chapter 12 — 错误恢复策略

### 12.1 错误分类

query.ts 中的错误处理区分两类：

**可扣押错误**（Withheld）：发现后暂不向调用方 yield，尝试恢复：
- `prompt_too_long`（413）：上下文超出模型窗口
- `max_output_tokens`：输出被截断
- `media_size_error`：包含过大的图像

**不可扣押错误**（Non-withheld）：直接 yield 给调用方：
- 网络错误、认证失败、服务端 5xx

### 12.2 Prompt-Too-Long 三级降级

```
prompt_too_long 错误触发
       │
       ▼
Tier 1: Context Collapse（feature('CONTEXT_COLLAPSE')）
  contextCollapse.recoverFromOverflow(messagesForQuery)
  成功 → transition: collapse_drain_retry → 继续循环
       │ 失败
       ▼
Tier 2: Reactive Compact（feature('REACTIVE_COMPACT')）
  reactiveCompact.tryReactiveCompact(messages)
  成功 → transition: reactive_compact_retry → 继续循环
       │ 失败（hasAttemptedReactiveCompact = true）
       ▼
Tier 3: 放弃恢复
  yield lastMessage  // 向调用方暴露错误
  return { reason: 'prompt_too_long' }
```

### 12.3 Max-Output-Tokens 多轮恢复

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

// 策略一：Token 升级（每轮仅一次）
if (!hasEscalated && !maxOutputTokensOverride) {
  maxOutputTokensOverride = ESCALATED_MAX_TOKENS  // 8K → 64K
  transition = { reason: 'max_output_tokens_escalate' }
  continue
}

// 策略二：注入恢复消息（最多 3 次）
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  maxOutputTokensRecoveryCount++
  // 注入："Output token limit hit. Resume directly from where you left off..."
  logEvent('tengu_max_output_tokens_recovery', { attempt: maxOutputTokensRecoveryCount })
  transition = { reason: 'max_output_tokens_recovery' }
  continue
}

// 3 次均失败：正常继续（不再扣押错误）
```

### 12.4 媒体尺寸错误

```typescript
// 响应式压缩：从保留的尾部消息中剥离图像
if (reactiveCompact?.isWithheldMediaSizeError) {
  const success = await reactiveCompact.tryReactiveCompact(messages)
  // 压缩器会找到最近的图像消息并将其替换为文本描述
  if (success) {
    transition = { reason: 'reactive_compact_retry' }
    continue
  }
  return { reason: 'image_error' }
}
```

---

## Chapter 13 — Task 系统

### 13.1 Task 类型体系（src/Task.ts，126 行）

```typescript
export type TaskType =
  | 'local_bash'           // 本地 Shell 命令
  | 'local_agent'          // 后台 Agent（异步派生）
  | 'remote_agent'         // 云端 Agent 会话
  | 'in_process_teammate'  // 进程内队友
  | 'local_workflow'       // 工作流脚本
  | 'monitor_mcp'          // MCP 工具监控
  | 'dream'                // 自动记忆整合 Agent

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

**Task ID 生成**：前缀字母（b/a/r/t/w/m/d）+ 8 位随机字符（0-9a-z，约 2.8 万亿种组合），大小写不敏感字母表防御符号链接攻击。

### 13.2 TaskStateBase（所有 Task 共有字段）

```typescript
type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string        // 创建此 Task 的工具调用 ID
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string        // 标准输出/错误的磁盘路径
  outputOffset: number      // 流式追加的偏移量
  notified: boolean         // 防止重复通知
}
```

### 13.3 LocalAgentTask：后台 Agent 生命周期

**注册（registerAsyncAgent）**：
1. 创建独立 AbortController（或父 Agent 的子控制器）
2. 初始化输出符号链接（指向 Agent transcript 路径）
3. 创建 `LocalAgentTaskState`（`status: 'running'`，`isBackgrounded: true`）
4. 注册清理处理器（进程退出时 kill Agent）
5. 注入 AppState

**前台注册（registerAgentForeground）**：

```typescript
// 返回 backgroundSignal Promise
// 以下任一情况触发：
// 1. 用户主动后台化
// 2. autoBackgroundMs 超时自动后台化
{
  taskId: string
  backgroundSignal: Promise<void>
  cancelAutoBackground?: () => void
}
```

**通信（Pending Messages）**：
- `queuePendingMessage()`：SendMessage 工具在轮次中途调用，消息入队
- `drainPendingMessages()`：在工具轮次边界清空队列，注入到下次 API 调用

**进度追踪**：

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number              // input + output 累计
  lastActivity?: ToolActivity     // 最近工具调用
  recentActivities?: ToolActivity[] // 最近 5 次（上限）
  summary?: string                // 后台摘要
}
```

**UI 内存管理**：
- `TEAMMATE_MESSAGES_UI_CAP = 50`：AppState 中最多保留 50 条消息（防止 RSS 爆炸）
- `retain: boolean`：UI 正在查看该 Agent 时设为 true，阻止驱逐
- `evictAfter?: number`：Terminal 状态后 30s（PANEL_GRACE_MS）驱逐

### 13.4 DreamTask：自动记忆整合

DreamTask 是一个**不可见的后台 Agent**，负责定期整合旧会话的记忆/上下文。它在 Task 注册表中可见（足迹显示在状态栏 pill 和 Shift+Down 对话框中），但不在 REPL 中交互。

```typescript
type DreamPhase = 'starting' | 'updating'

type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase
  sessionsReviewing: number     // 正在整合的会话数
  filesTouched: string[]        // 已修改的文件列表
  turns: DreamTurn[]            // 最近 30 轮（限制）
  abortController?: AbortController
  priorMtime: number            // 记忆锁回滚用
}
```

### 13.5 RemoteAgentTask：云端会话管理

```typescript
type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: 'remote-agent' | 'ultraplan' | 'ultrareview' | 'autofix-pr' | 'background-pr'
  sessionId: string
  command: string
  title: string
  todoList: TodoList
  log: SDKMessage[]
  pollStartedAt: number         // 用于超时检测
  isLongRunning?: boolean       // 不会在首次结果后完成
  reviewProgress?: {            // ultrareview 专用
    stage?: 'finding' | 'verifying' | 'synthesizing'
    bugsFound: number
    bugsVerified: number
    bugsRefuted: number
  }
}
```

远程 Task 通过 `RemoteTaskCompletionChecker` 轮询 GitHub/API 检查完成状态。

### 13.6 Task 足迹标签（pillLabel.ts）

状态栏 pill 的紧凑文本：

```
1 shell            → 1 个 Bash 任务
2 monitors         → 2 个监控任务
◇ 1 cloud session  → 1 个远程 Agent
◆ ultraplan ready  → ultraplan 计划已就绪
dreaming           → DreamTask 运行中
3 background tasks → 混合类型任务
```

---

## Chapter 14 — 后台任务与 Cron 调度

### 14.1 后台执行模型

LocalShellTask 实现后台命令执行：命令在子进程中运行，输出流式追加到 `outputFile`。主 Agent 继续处理其他工作，子进程完成时通过 Task 通知机制注入结果。

### 14.2 LocalWorkflowTask

工作流脚本执行，支持多步骤流水线：

```typescript
type LocalWorkflowTask = TaskStateBase & {
  type: 'local_workflow'
  // 工作流定义和执行状态
}
```

### 14.3 MonitorMcpTask

监控特定 MCP 工具的持续运行状态：

```typescript
type MonitorMcpTask = TaskStateBase & {
  type: 'monitor_mcp'
  // MCP 工具监控配置
}
```

### 14.4 Cron 调度器（DeferExecuteTool: CronCreate）

Cron 调度器作为 Deferred Tool 提供，通过 `ToolSearch` 发现：
- 支持 cron 表达式和一次性定时触发
- 作用域限于当前会话（会话结束时 Job 消失）
- 典型用途：定期检查任务状态、定时提醒、周期性数据收集

---

## Chapter 15 — Agent 团队架构

### 15.1 Mailbox 协议（src/utils/mailbox.ts，74 行）

Mailbox 是多 Agent 通信的基础数据结构：

```typescript
type Message = {
  id: string
  source: 'user' | 'teammate' | 'system' | 'tick' | 'task'
  content: string
  from?: string
  color?: string
  timestamp: string
}

class Mailbox {
  private queue: Message[] = []
  private waiters: Waiter[] = []

  send(msg: Message): void {
    // 若有匹配的 waiter → 立即 resolve（不入队）
    // 否则 → 入队
  }

  receive(filter?: (msg: Message) => boolean): Promise<Message> {
    // 若队列有匹配项 → 立即返回
    // 否则 → 返回 Promise，等待下次 send()
  }

  poll(filter?): Message | undefined {
    // 非阻塞检查
  }

  subscribe = this.changed.subscribe  // React 重渲染触发
}
```

**特性**：
- `revision` 计数器：每次 send 递增，用于 React re-render 触发
- waiter 机制：先 receive 后 send 时，不进入队列直接 resolve

### 15.2 团队创建（TeamCreateTool，241 行）

```typescript
async function call(input, context): Promise<Output> {
  // 1. 检查：Leader 只能管理一个团队
  if (appState.teamContext?.teamName) throw new Error('Already leading a team')

  // 2. 生成唯一团队名
  const finalTeamName = generateUniqueTeamName(team_name)

  // 3. 写入团队文件
  const teamFile: TeamFile = {
    name: finalTeamName,
    leadAgentId: formatAgentId('team-lead', finalTeamName),
    leadSessionId: getSessionId(),
    members: [/* team-lead 成员 */]
  }
  await writeTeamFileAsync(finalTeamName, teamFile)

  // 4. 创建共享任务列表目录
  await resetTaskList(sanitizeName(finalTeamName))

  // 5. 更新 AppState
  setAppState(prev => ({
    ...prev,
    teamContext: { teamName, leadAgentId, teammates: {...} }
  }))
}
```

### 15.3 SendMessage 工具：消息路由

```typescript
// 目标格式：
// - "teammate_name" → 写入指定队友的 mailbox
// - "*" → 广播给所有队友
// - "uds:<socket-path>" → 通过 Unix Domain Socket 发送
// - "bridge:<session-id>" → 发送给远程会话

// 消息类型（结构化 JSON）：
// - 普通文本
// - { type: 'shutdown_request' }
// - { type: 'shutdown_response', approve: boolean }
// - { type: 'plan_approval_response', approve: boolean }
```

### 15.4 InProcessTeammateTask：进程内队友

与 LocalAgentTask 的关键区别：

| 特性 | LocalAgentTask | InProcessTeammateTask |
|------|---------------|----------------------|
| 执行方式 | 独立进程 / 并发 | 同一 Node.js 进程，AsyncLocalStorage 隔离 |
| 上下文 | 完全独立 | 共享进程内存，隔离身份 |
| 通信 | Pending Messages 队列 | Mailbox 协议 |
| 团队感知 | 否 | 是（TeammateIdentity） |
| 计划模式 | 否 | 支持（planModeRequired） |
| 消息历史 | 完整 transcript 在磁盘 | AppState 中上限 50 条 |

```typescript
type TeammateIdentity = {
  agentId: string            // "researcher@my-team"
  agentName: string          // "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string    // Leader 的 session ID
}
```

### 15.5 自组织：idle cycle 与自动任务认领

队友在完成当前工作后进入 `isIdle: true` 状态，此时：
1. 触发 TeammateIdle Stop Hook
2. 检查团队任务列表中未认领的任务
3. 若有可用任务（`status: 'pending'`，无 blockedBy），认领并开始执行
4. 更新任务状态为 `'in_progress'`，设置 owner

这实现了**自组织多 Agent 协作**：Leader 无需逐一分配任务，队友主动认领。

### 15.6 Plan Mode 与审批流

```typescript
// 队友 planModeRequired = true 时
// 1. 队友进入计划模式（只读操作）
// 2. 完成分析后调用 ExitPlanMode
// 3. ExitPlanMode 发送 plan_approval_request 给 Leader
// 4. Leader 调用 SendMessage({type: 'plan_approval_response', approve: true/false})
// 5. 队友收到批准后才开始执行写操作
```

---

## Chapter 16 — Worktree 隔离与远程模式

### 16.1 Worktree 隔离（EnterWorktreeTool/LeaveWorktreeTool）

Git Worktree 为每个任务提供独立的工作目录：

```
主仓库:     /project/             (main 分支)
Worktree A: /project/.worktrees/feature-a/  (feature-a 分支)
Worktree B: /project/.worktrees/feature-b/  (feature-b 分支)
```

**任务-目录绑定**：每个 WorktreeRecord 与一个 Task ID 绑定，确保并行 Agent 不互相干扰：

```typescript
type WorktreeRecord = {
  taskId: string        // 绑定的 Task ID
  worktreePath: string  // 独立工作目录
  branch: string        // 独立 Git 分支
  tmuxSessionName?: string  // 可选 tmux 集成
}
```

### 16.2 Bridge 模式架构（src/bridge/，20+ 文件）

Bridge 模式允许远程客户端（如 Claude.ai Web 端）控制本地 Claude Code 实例。

**组件**：
- `bridgeMain.ts`：轮询循环，管理多个并发会话
- `bridgeApi.ts`：HTTP 客户端（环境注册、工作轮询、心跳）
- `bridgeMessaging.ts`：REPL 消息处理
- `jwtUtils.ts`：JWT 解析与 Token 刷新调度
- `createSession.ts`：POST /v1/sessions（含 Git 上下文）
- `bridgePermissionCallbacks.ts`：权限请求/响应桥接

**JWT Token 刷新调度器**：

```typescript
createTokenRefreshScheduler({
  getAccessToken,
  onRefresh: (sessionId, token) => { /* 交付给子进程或触发服务端重派发 */ },
  refreshBufferMs: 5 * 60 * 1000  // 到期前 5 分钟刷新
})
```

**特性**：
- 生成计数器（generation counter）检测过期 Timer
- 失败重试（上限 3 次）
- v1 会话：直接交付 OAuth Token；v2 会话：触发服务端重新派发

**Bridge 主循环**：

```typescript
while (!aborted) {
  await pollForWork()           // 获取新会话
  await heartbeatActiveWorkItems()  // 保持会话活跃
  await sleep(POLL_INTERVAL_MS)
}
```

### 16.3 Remote Session Manager（src/remote/）

```typescript
// RemoteSessionManager.ts：管理多个并发远程会话
// SessionsWebSocket.ts：WebSocket 通信（消息流式传输）
// remotePermissionBridge.ts：权限请求的远程代理
// sdkMessageAdapter.ts：SDK 消息格式适配
```

### 16.4 SSH 会话管理（src/ssh/）

```typescript
// SSHSessionManager.ts：SSH 会话的创建和生命周期
// createSSHSession.ts：建立 SSH 连接，处理密钥认证

// SSH 模式特殊处理：
// - 支持 --permission-mode 参数
// - 禁止与 --print/-p 组合使用
// - 支持 -c/--continue/--resume/--model 参数转发
```

### 16.5 CCR Upstream Proxy（src/upstreamproxy/）

容器侧 Upstream Proxy 配置（用于 CCR 会话）：

1. 从 `/run/ccr/session_token` 读取会话 Token
2. 设置 `prctl(PR_SET_DUMPABLE, 0)`（阻止 ptrace 堆转储，防止 Token 泄露）
3. 下载 CCR CA 证书，与系统 CA 包合并
4. 启动 CONNECT→WebSocket 中继
5. 删除 Token 文件（Token 留在堆内存，文件层已清理）
6. 暴露 `HTTPS_PROXY` 和 `SSL_CERT_FILE` 环境变量

**NO_PROXY 白名单**：loopback、RFC1918 私有地址、IMDS、anthropic.com（所有形式）、github.com、npm/pypi/crates.io 等包注册表。

**设计原则**：每一步都失败安全——代理配置损坏不会中断正常会话。

### 16.6 Assistant 模式（src/assistant/）

```typescript
// index.ts：Assistant 模式入口（Kairos/常驻助手模式）
// sessionDiscovery.ts：发现历史 Assistant 会话
// sessionHistory.ts：历史会话读取与管理
```

---

## Chapter 17 — Skill 与 Plugin 系统

### 17.1 Skill 系统架构（src/skills/loadSkillsDir.ts，1087 行）

Skill 是以 Markdown 文件定义的 PromptCommand，支持丰富的 frontmatter 配置：

```markdown
---
name: commit-push-pr
description: 创建 commit、push 并开 PR
allowedTools:
  - Bash(git add:*)
  - Bash(git commit:*)
  - Bash(git push:*)
model: claude-opus-4
context: fork
paths:
  - "**/*.ts"
  - "**/*.py"
hooks:
  PostToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: echo "Tool used"
---

Analyze the staged changes and create a commit...
```

**Frontmatter 字段解析（parseSkillFrontmatterFields）**：

| 字段 | 类型 | 说明 |
|------|------|------|
| name | string | 命令名 |
| description | string | 命令描述 |
| allowedTools | string[] | 限制可用工具 |
| model | string | 覆盖模型 |
| context | 'inline'\|'fork' | 执行上下文 |
| paths | string[] | 条件激活模式（gitignore 语法） |
| hooks | HooksSettings | 调用时注册的 Hooks |
| argumentHint | string | 参数提示 |
| whenToUse | string | AI 使用场景提示 |
| disableModelInvocation | boolean | 不加入 AI 工具列表 |
| userInvocable | boolean | 用户可通过 /name 调用 |

### 17.2 Skill 发现优先级

```
Managed > User > Project > Additional > Legacy commands

具体路径（按优先级）：
  ~/.claude/skills/          (User 级别)
  /项目/.claude/skills/      (Project 级别)
  /额外目录/.claude/skills/   (Additional 级别)
  /项目/.claude/commands/    (Legacy，兼容旧版)
```

**去重**：通过 `realpath()` 检测符号链接别名，防止同一文件注册两次。

**条件激活（paths 字段）**：

```typescript
// 仅当工作区包含匹配文件时，该 Skill 才可见
paths: ["**/*.ts", "src/**/*.py"]

// discoverSkillDirsForPaths() 从文件路径向上遍历找 .claude/skills/
// activateConditionalSkillsForPaths() 使用 gitignore 风格匹配
```

### 17.3 Bundled Skills（20 个内置技能）

```
src/skills/bundled/
├── batch.ts         批量操作
├── claudeApi.ts     Claude API 调用
├── debug.ts         调试辅助
├── dream.ts         记忆整合（Dream 模式）
├── hunter.ts        Bug 猎手
├── loop.ts          循环执行
├── remember.ts      记忆保存
├── skillify.ts      技能创建辅助
└── ...（共 20 个）
```

通过 `registerBundledSkill()` 在启动时注册，`safeWriteFile()` 使用 O_NOFOLLOW + O_EXCL + 0o600 权限安全写入参考文件。

### 17.4 Plugin 系统（src/plugins/）

```typescript
// builtinPlugins.ts
type BuiltinPlugin = {
  id: string             // "{name}@builtin"
  name: string
  description: string
  defaultEnabled: boolean
  skills: BundledSkillDefinition[]
}

// 用户可通过 /plugin 命令启用/禁用内置插件
// 用户偏好 > 默认值
function getBuiltinPlugins(): { enabled: BuiltinPlugin[]; disabled: BuiltinPlugin[] }
```

**加载流程**：
1. `initBuiltinPlugins()`：初始化内置插件（`src/plugins/bundled/index.ts`）
2. `loadPluginHooks()`：从插件清单加载 Hooks
3. `getBuiltinPluginSkillCommands()`：将启用的插件技能转为 Command 对象（source: 'bundled'）

### 17.5 MCP Skill Builders

```typescript
// mcpSkillBuilders.ts：注册表桥接（避免循环导入）
// 写一次注册：registerMcpSkillBuilders(createSkillCommand, parseSkillFrontmatterFields)
// mcpSkills.ts：fetchMcpSkillsForClient() 返回空数组（MCP_SKILLS 特性门控，暂未实现）
```

---

## Chapter 18 — MCP 协议与外部工具池

### 18.1 MCP 协议架构

MCP（Model Context Protocol）是 Anthropic 开发的开放协议，允许 LLM 应用连接外部工具服务器。Claude Code 实现了完整的 MCP 客户端。

**支持的传输层**：

| 传输层 | 使用场景 |
|--------|---------|
| Stdio | 本地进程通信（最常见） |
| SSE（Server-Sent Events） | HTTP 流式传输 |
| StreamableHTTP | HTTP 请求-响应 |
| WebSocket | 双向实时通信 |

### 18.2 MCP 客户端实现（src/services/mcp/client.ts，3000+ 行）

**关键常量**：
```typescript
const DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000  // ~27.8 小时（实际不超时）
const MAX_MCP_DESCRIPTION_LENGTH = 2048           // 工具描述截断上限
```

**错误类型**：
```typescript
class McpAuthError extends Error          // OAuth 认证失败（401）
class McpSessionExpiredError extends Error // HTTP 404 + JSON-RPC -32001
class McpToolCallError extends Error       // 工具执行错误
```

**工具生成**：MCP 服务器提供的每个工具，客户端动态创建 `MCPTool` 实例，覆盖以下字段：
- `name`：`mcp__{serverName}__{toolName}` 格式
- `description()`：来自服务器的工具描述
- `inputSchema`：服务器提供的 JSON Schema（转换为 Zod）
- `call()`：通过 MCP 协议调用服务器工具
- `checkPermissions()`：MCP 权限检查

### 18.3 工具池组装与 MCP 集成

```typescript
// assembleToolPool() 将内置工具与 MCP 工具合并：
// 内置工具：按名称排序（稳定 Cache Prefix）
// MCP 工具：按名称排序（独立分区）
// 合并：内置工具在前，MCP 工具在后
// 去重：同名取第一个（内置优先）
```

**MCP 工具权限**：MCP 工具可被 `alwaysDenyRules` 按服务器或工具粒度拒绝：

```
"mcp__server1"           → 拒绝来自 server1 的所有工具
"mcp__server1__tool1"    → 拒绝特定工具
"mcp__server1__*"        → 通配符拒绝
```

### 18.4 ListMcpResources 与 ReadMcpResource

```typescript
// ListMcpResourcesTool：列出 MCP 服务器的可用资源
// - 资源类似"文件"，由 MCP 服务器管理
// - 支持按 URI 模板过滤

// ReadMcpResourceTool：读取特定 MCP 资源内容
// - 内容可以是文本、blob（base64 编码）
// - 用于读取数据库记录、配置文件等
```

### 18.5 MCP 认证（src/services/mcp/auth.ts）

```typescript
// ClaudeAuthProvider：向 MCP 服务器提供 Claude.ai OAuth Token
// 支持 Token 刷新（过期前自动刷新）

// XAA IdP Login（xaa.ts, xaaIdpLogin.ts）：
// 企业身份提供者集成，用于内部 MCP 服务器
```

---

## Chapter 19 — 多 LLM Provider 支持

### 19.1 Provider 抽象层

```typescript
// src/utils/model/providers.ts
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

// 另有 OpenAI 兼容类型：
type OpenAICompatibleProvider = 'openai-compatible' | 'github-models' | 'github-copilot'
```

所有 Provider 通过 `getAnthropicClient()` 工厂函数创建客户端，屏蔽底层 SDK 差异。

### 19.2 各 Provider 实现

**firstParty（直连 Anthropic API）**：
```
端点：api.anthropic.com/v1/messages
认证：ANTHROPIC_API_KEY 环境变量 或 OAuth Token
默认：是
```

**bedrock（AWS Bedrock）**：
```
认证：AWS SDK 默认凭证链（env var / ~/.aws/credentials / IAM Role）
区域：AWS_REGION 或 AWS_DEFAULT_REGION（默认 us-east-1）
特殊：ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION（Haiku 专用区域覆盖）
启用：CLAUDE_CODE_USE_BEDROCK=true 或配置文件
```

**vertex（Google Cloud Vertex AI）**：
```
认证：Google Auth Library（DefaultCredential）
项目：ANTHROPIC_VERTEX_PROJECT_ID（必须）
区域：按模型族独立配置：
  VERTEX_REGION_CLAUDE_3_5_SONNET
  VERTEX_REGION_CLAUDE_3_7_SONNET
  CLOUD_ML_REGION（全局回退，默认 us-east5）
启用：CLAUDE_CODE_USE_VERTEX=true 或配置文件
```

**foundry（Azure Anthropic Foundry）**：
```
端点：https://{resource}.services.ai.azure.com/anthropic/v1/messages
认证：ANTHROPIC_FOUNDRY_API_KEY 或 DefaultAzureCredential
资源：ANTHROPIC_FOUNDRY_RESOURCE 或 ANTHROPIC_FOUNDRY_BASE_URL
启用：CLAUDE_CODE_USE_FOUNDRY=true 或配置文件
```

**GitHub Models / GitHub Copilot（OpenAI 兼容）**：
```
认证：GH_TOKEN → GITHUB_TOKEN → gh auth token（按优先级）
端点：GitHub Models API / GitHub Copilot API
支持模型：claude-sonnet-4.6, claude-opus-4.6, claude-haiku-4.5
```

### 19.3 Model 选择与 Thinking 模式

```typescript
// src/utils/model/ 目录
// providers.ts：Provider 检测与配置
// providerConfig.ts：活跃 Provider 配置获取
// client.ts：客户端工厂

// Thinking 模式（src/utils/thinking.ts）
type ThinkingConfig = {
  type: 'enabled' | 'disabled'
  budget_tokens?: number    // 思考 token 预算
}

// 自适应思考预算：
// - 检测当前轮次剩余 token
// - 动态调整 budget_tokens
// - 若 budget_tokens 改变，会导致 KV Cache Miss（需注意）
```

---

## Chapter 20 — KV-Cache 感知：贯穿全系统的优化哲学

> **"Claude Code 的所有设计决策都围绕'如何最大化 Prompt Cache 命中'展开。"**

这不仅仅是一个性能优化。当 Agent 每轮都需要传入完整上下文时，Cache 命中率直接决定：
- **成本**：Cache Hit 降低 90% 费用
- **延迟**：Cache Hit 降低 80% 首 token 延迟

### 20.1 Anthropic Prompt Cache 机制

Anthropic API 支持 Prompt Cache：若请求前缀与上次相同（按 Token 精确匹配），API 会复用已缓存的 KV-Cache，大幅降低计算开销。

**Cache Key 构成**（按顺序）：
1. System Prompt 内容
2. 工具列表（名称 + Schema）
3. 对话消息历史

任何一处变化都会导致该位置及其后的所有内容 Cache Miss。

**TTL 配置**：
```typescript
getCacheControl({ scope, querySource }) → {
  type: 'ephemeral',
  ttl?: '1h',    // 满足条件时（ANT 用户或订阅用户）
  scope?: 'org'  // 组织级 Cache 共享
}
// TTL 在启动时锁定（防止会话中途 TTL 变化破坏 Cache）
```

### 20.2 工具列表按名称排序

```typescript
// src/tools.ts
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)

return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

**为什么分区排序**：
- 平排序（所有工具一起排序）会使 MCP 工具穿插在内置工具中
- 若 MCP 工具出现/消失，会改变所有内置工具后缀的 Cache Key
- **分区排序**：内置工具构成稳定的连续 Prefix，MCP 工具附加在后
- 内置工具区域的 Cache 不受 MCP 工具变化影响

**实践建议**：新增工具时，按字典序插入，而非追加到末尾。

### 20.3 cache_control 标记位置

```typescript
// 仅在最后一条消息上添加 cache_control
// 这保护了该消息之前的所有内容的 Cache
messages[messages.length - 1].cache_control = getCacheControl({ querySource })
```

同时在 System Prompt 末尾和工具列表末尾也添加 `cache_control`，形成分层缓存：

```
[System Prompt] cache_control  → 全局 Cache（跨会话）
[Tool List]     cache_control  → 工具级 Cache
[Messages]      cache_control  → 消息级 Cache（最后一条）
```

### 20.4 CacheSafeParams：Fork Agent 复用父请求 Prefix

```typescript
// src/utils/forkedAgent.ts
type CacheSafeParams = {
  systemPrompt: SystemPrompt    // 与父 Agent 完全相同
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext // 含相同工具列表
  forkContextMessages: Message[] // 父上下文消息（作为 prefix）
}
```

**关键约束**：fork 模式下不能为子 Agent 设置不同的 `maxOutputTokens`，因为这会改变 API 请求中的 `budget_tokens`，导致 Cache Miss。

### 20.5 压缩方向选择：'from' 保护 Prefix

```typescript
// 所有压缩操作统一使用 'from' 方向
compact({ direction: 'from', messages })
// 保留前缀 → 截断中段 → 保留末尾
```

已被缓存的前缀内容（系统提示、早期对话历史）不会因压缩而改变，维持 Cache Hit。

### 20.6 promptCacheBreakDetection：Cache 断裂追踪

`src/services/api/promptCacheBreakDetection.ts`（728 行）实现了完整的 Cache 断裂检测系统：

**Phase 1：预调用状态录制**

```typescript
function recordPromptState(snapshot: PromptStateSnapshot): void {
  // 记录：
  // - System Prompt Hash（去除 cache_control）
  // - 工具 Schema Hash（去除 cache_control）
  // - cacheControlHash（含 TTL/scope——检测 TTL 变化导致的 Cache Miss）
  // - 每个工具的独立 Hash（定位具体是哪个工具变化）
  // - Beta 标志、Auto 模式状态、Overage 状态
}
```

**Phase 2：后调用分析**

```typescript
function checkResponseForCacheBreak(querySource, cacheReadTokens, cacheCreationTokens, ...): void {
  // 检测条件：
  // 1. cacheReadTokens < prevCacheRead * 0.95（>5% 下降）
  // 2. tokenDrop >= MIN_CACHE_MISS_TOKENS（最小 2,000 tokens）
  // 3. 非 Haiku 模型（Haiku 缓存行为不同）

  // 若检测到断裂：
  // - 创建 diff 文件（调试用）
  // - 发送 tengu_prompt_cache_break 分析事件
}
```

**追踪键路由**：

| querySource | 追踪键 |
|-------------|--------|
| repl_main_thread | 'repl_main_thread' |
| compact | 'repl_main_thread'（共享主线程 Cache） |
| sdk | 'sdk' |
| agent:custom | 使用唯一 agentId |
| agent:default | 使用唯一 agentId |
| speculation | 不追踪（短生命周期） |

**Analytics 事件 tengu_prompt_cache_break**：

记录 12 个布尔标志（systemPromptChanged、toolSchemasChanged、modelChanged 等）、工具增删情况（MCP 工具折叠为 'mcp' 以脱敏）、Token 计数、距上次请求时间、TTL 过期检测结果。

### 20.7 系统级 KV-Cache 最佳实践总结

以下设计决策均受 KV-Cache 感知驱动：

| 设计决策 | Cache 价值 |
|---------|-----------|
| System Prompt 按 Section 分区，动态区用 DANGEROUS_uncached | 静态区全局共享 Cache |
| 工具列表分区排序（内置 + MCP 分开） | 内置工具 Prefix 稳定 |
| cache_control 仅放最后一条消息 | 保护历史消息 Cache |
| Fork Agent 复用父 CacheSafeParams | 父子共享 System+Tools Cache |
| 压缩方向选 'from' | 保护已缓存的历史 Prefix |
| TTL 在启动时锁定 | 防止会话中 TTL 变化 |
| Tool Output Budget：持久化超大输出 | 减少消息体积，稳定 Prefix |
| MicroCompact 使用固定占位符文本 | 占位符内容不变 = Cache Hit |
| 工具排序按名称字典序 | 新增工具不改变现有顺序 |

---

## Chapter 21 — 终端 UI 层

### 21.1 自研 Ink 渲染引擎（src/ink/）

Claude Code 的 UI 层基于**自研的 Ink 渲染引擎**，而非标准 ink.js 库。两者的关键差异：

| 特性 | 标准 Ink.js | 自研版本 |
|------|------------|---------|
| 渲染策略 | 立即渲染到终端 | 帧级渲染（Diff 优化） |
| 布局引擎 | 简单换行 | Facebook Yoga Flexbox |
| 滚动支持 | 有限 | 虚拟滚动（DECSTBM 优化） |
| 事件系统 | 基础 | 两阶段捕获/冒泡（React 风格） |
| 双向文字 | 无 | Unicode Bidi Algorithm |
| 性能监控 | 无 | 帧时序插桩（各阶段耗时） |

### 21.2 DOM 抽象层（src/ink/dom.ts，485 行）

自定义 DOM 节点类型：

```typescript
type DOMElement = {
  nodeName: 'ink-root' | 'ink-box' | 'ink-text' | 'ink-virtual-text'
           | 'ink-link' | 'ink-progress' | 'ink-raw-ansi'
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  textStyles?: TextStyles

  // 脏标记系统
  dirty: boolean

  // Yoga 集成
  yogaNode?: LayoutNode       // 需要布局的节点才有此字段

  // 滚动状态（overflow: 'scroll' 的 box）
  scrollTop?: number
  pendingScrollDelta?: number
  scrollClampMin?: number
  scrollClampMax?: number
  stickyScroll?: boolean
  scrollAnchor?: { el: DOMElement; offset: number }

  // 调试
  debugOwnerChain?: string[]  // React 组件调用栈
}
```

**节点类型与 yogaNode 对应**：

| 节点类型 | 有 yogaNode | 说明 |
|---------|:-----------:|------|
| ink-root | ✓ | 文档根，拥有 FocusManager |
| ink-box | ✓ | 容器/布局节点 |
| ink-text | ✓ | 文本（有 measure 函数） |
| ink-virtual-text | ✗ | 无布局的文本 |
| ink-link | ✗ | 超链接包装 |
| ink-progress | ✗ | 进度条 |
| ink-raw-ansi | ✗ | 预渲染 ANSI（已知尺寸） |

**关键优化——脏标记去重**：

```typescript
function setStyle(node: DOMElement, style: StyleRecord): void {
  // React 每次渲染分配新对象，但值可能相同
  // 浅相等检查避免不必要的 Yoga 重计算
  if (stylesEqual(node.style, style)) return
  applyStyle(node.yogaNode, style)
}
```

### 21.3 Yoga 布局引擎集成

布局层有三个抽象级别：

```
src/ink/layout/engine.ts      工厂（委托给 Yoga）
src/ink/layout/node.ts        LayoutNode 接口定义
src/ink/layout/yoga.ts        YogaLayoutNode 适配器
src/native-ts/yoga-layout/    纯 TypeScript Yoga 移植（2500 行 C++ → TS）
```

**LayoutNode 接口**（完整 Flexbox 支持）：
- Flex 方向、对齐、换行
- 尺寸：像素、百分比、auto
- 约束：min/max width/height
- 间距：margin、padding、border、gap
- 定位：absolute / relative
- 溢出：visible、hidden、scroll

**纯 TypeScript Yoga 移植**的实现范围：
- 实现：Ink 使用的全部特性 + 部分规范功能（margin:auto、多轮 flex 钳制等）
- **未实现**：aspect-ratio、RTL 方向（始终 LTR）、content-box 盒模型

### 21.4 事件系统（src/ink/events/）

完整的两阶段事件分发（模仿 react-dom）：

```typescript
class Dispatcher {
  dispatch(target: EventTarget, event: TerminalEvent): boolean {
    // Phase 1: 收集监听器
    // 捕获阶段：root → parent → target（unshift 插入，根优先）
    // 冒泡阶段：target → parent → root（push 追加，目标优先）

    // Phase 2: 执行（支持 stopPropagation）
    // 错误捕获：单个 handler 失败不中断分发
  }
}
```

**事件优先级映射**（对应 React 调度优先级）：

| 事件类型 | 优先级 |
|---------|--------|
| keydown, keyup, click, focus, blur, paste | DiscreteEventPriority（最高） |
| resize, scroll, mousemove | ContinuousEventPriority |
| 其他 | DefaultEventPriority |

**支持的事件类型**（12 种）：
- `click-event`, `input-event`, `keyboard-event`
- `focus-event`, `paste-event`, `resize-event`
- `terminal-event`, `terminal-focus-event`

### 21.5 Frame 与 Cursor 管理（src/ink/frame.ts）

```typescript
type Frame = {
  readonly screen: Screen           // 渲染后的字符网格
  readonly viewport: Size           // 终端尺寸
  readonly cursor: Cursor           // 位置 + 可见性
  readonly scrollHint?: ScrollHint  // DECSTBM 滚动优化（仅 alt-screen）
  readonly scrollDrainPending?: boolean  // 需要再渲染一帧
}
```

**Patch 类型**（输出序列的最小单元）：

```typescript
type Patch =
  | { type: 'stdout'; content: string }
  | { type: 'clear'; count: number }
  | { type: 'clearTerminal'; reason: FlickerReason }
  | { type: 'cursorMove'; x: number; y: number }
  | { type: 'cursorHide' | 'cursorShow' }
  | { type: 'styleStr'; str: string }
  | { type: 'hyperlink'; uri: string }
```

**帧时序插桩**（完整渲染管线的各阶段耗时）：

```
renderer      → DOM → Yoga → 字符网格
diff          → 新旧帧差异计算
optimize      → Patch 合并/去重
write         → Patch → ANSI → stdout
yoga          → calculateLayout() 耗时
commit        → React 协调耗时
yogaVisited   → layoutNode() 调用次数
yogaMeasured  → 文本换行/宽度计算次数
yogaCacheHits → Yoga 缓存命中次数
```

**清屏触发条件**：

```typescript
function shouldClearScreen(prevFrame, frame): FlickerReason | undefined {
  // 1. 终端尺寸变化 → 'resize'
  // 2. 当前帧高度 ≥ 视口高度 → 'offscreen'
  // 3. 上一帧高度 ≥ 视口高度 → 'offscreen'
}
```

### 21.6 ANSI 与色彩处理

**Ansi 组件**（292 行）：

```tsx
<Ansi dimColor>
  {"\x1b[31mRed text\x1b[0m Normal text"}
</Ansi>
```

解析流程：ANSI 转义码 → `termio.Parser` → styled spans → `<Text>` + `<Link>` 组件

**色彩系统**（colorize.ts）：

支持的格式：
- `ansi:red`（命名颜色）
- `ansi256(42)`（256 色板）
- `#FF6B6B`（Hex RGB）
- `rgb(255, 107, 107)`

**终端能力适配**：
- VS Code（`TERM_PROGRAM=vscode`）：chalk.level 2→3（xterm.js 支持真彩色）
- tmux：chalk.level 3→2（除非设置 `CLAUDE_CODE_TMUX_TRUECOLOR`）

**双向文字支持**（bidi.ts，140 行）：

```typescript
function needsBidi(): boolean {
  return process.platform === 'win32' ||
         typeof process.env['WT_SESSION'] === 'string' ||  // WSL in Windows Terminal
         process.env['TERM_PROGRAM'] === 'vscode'
}
```

Windows 终端缺乏原生 RTL 支持，通过 `bidi-js` 重排逻辑顺序为视觉顺序。macOS 终端（iTerm2、Terminal.app）有原生 Bidi 支持，无需处理。

### 21.7 React 组件架构与 AppState 状态管理

**Store 模式**（src/state/store.ts，35 行）：

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: () => void) => () => void
}

// 浅相等检查（Object.is()）防止相同引用触发不必要的 re-render
```

**AppState**（100+ 字段）主要分组：
- 设置与模型配置
- UI 状态（展开视图、Footer 选择、Coordinator 任务）
- Agent 与系统信息
- 远程会话状态（Viewer 模式）
- Bridge 配置
- 插件/服务器状态
- 工具权限与通知
- Analytics 门控

**SpeculationState**（预测执行）：

```typescript
| { status: 'idle' }
| {
    status: 'active'
    id: string
    abort: () => void
    messagesRef: { current: Message[] }     // 可变引用（避免每条消息 spread）
    writtenPathsRef: { current: Set<string> }
    boundary: CompletionBoundary | null     // 预测在哪里停止
    isPipelined: boolean
  }
```

**CompletionBoundary** 记录预测执行停止的原因（complete/bash/edit/denied_tool），用于决定是否将预测结果合并到主会话。

### 21.8 React Context 体系（src/context/）

| Context | 用途 |
|---------|------|
| QueuedMessageContext | 待处理消息队列 |
| MailboxContext | Agent 间消息（Kairos） |
| ModalContext | 模态对话框栈 |
| OverlayContext | 覆盖层渲染 |
| PromptOverlayContext | 提示输入覆盖 |
| NotificationsContext | Toast 通知队列（自动消失） |
| StatsContext | 性能指标（p50/p95/p99 分位数） |
| FpsMetricsContext | 帧渲染计时 |
| VoiceContext | 语音输入/输出状态 |

### 21.9 屏幕组件（src/screens/）

**Doctor.tsx**：诊断屏幕，展示：
- 设置校验错误
- 插件错误
- MCP 服务器解析警告
- Keybinding 冲突
- Sandbox/容器诊断
- 版本锁定信息（CCR PID 锁状态）
- Agent 定义（活跃 Agent、源路径）
- 更新可用性（npm/gcs dist-tags）

**REPL.tsx**：主交互提示屏幕（核心 UI 组件）

**ResumeConversation.tsx**：恢复历史对话屏幕

### 21.10 Keybinding 系统（src/keybindings/，15 文件）

```
parser.ts      → 解析 "ctrl+a", "alt+shift+l" 等快捷键字符串
match.ts       → 键盘事件与绑定的匹配
resolver.ts    → 优先级解析（用户 > 插件 > 默认）
validate.ts    → 配置校验
defaultBindings.ts → 内置快捷键
loadUserBindings.ts → 用户 ~/.config/keybindings.json
reservedShortcuts.ts → OS 保留快捷键（不可覆盖）
KeybindingContext.tsx → React Context Provider
useKeybinding.ts → 组件级绑定 Hook
```

### 21.11 Vim 模式模拟（src/vim/，5 文件）

```typescript
// types.ts - Vim 状态机类型
type VimMode = 'normal' | 'insert' | 'visual' | 'command'

// transitions.ts - 状态转移（键按下 → 新状态）
// motions.ts - 移动命令（h/j/k/l, w/e/b, G, 0, $, %, ...）
// operators.ts - 操作符（d/c/y/>/< 等）
// textObjects.ts - 文本对象（iw/aw/ip/ap 等）
```

### 21.12 交互辅助层（src/interactiveHelpers.tsx，58KB）

20 个对话框的顺序启动管线（首次运行时按序显示）：

1. Onboarding（主题选择）
2. TrustDialog（工作区信任 + claude.md 验证）
3. GrowthBook 重置
4. MCP 服务器审批（mcp.json 配置）
5. Claude.md 外部引用警告
6. Grove 策略对话框
7. 自定义 API Key 审批
8. Bypass 权限对话框（CI/CD 模式）
9. Auto 模式同意（TRANSCRIPT_CLASSIFIER）
10. 开发渠道对话框

所有对话框通过 Suspense + 异步加载，避免将所有组件打包在启动 bundle 中。

---

## Chapter 22 — 服务层横切关注点

### 22.1 API 服务（src/services/api/，20+ 文件）

核心文件：

| 文件 | 职责 |
|------|------|
| claude.ts | Claude API 客户端，含 KV-Cache 控制 |
| client.ts | HTTP 客户端基础层 |
| bootstrap.ts | 启动时认证与配置获取 |
| errors.ts / errorUtils.ts | API 错误分类与处理 |
| logging.ts | 请求/响应日志 |
| filesApi.ts | 文件上传/检索 API |
| usage.ts | Token/Credit 用量追踪 |
| promptCacheBreakDetection.ts | Cache 断裂检测（728 行） |

**错误分类**：

```typescript
// 可重试错误
type RetryableError = {
  status: 429 | 500 | 502 | 503 | 504
  retryAfter?: number
}

// 终止错误
type TerminalError = {
  status: 400 | 401 | 403  // 认证/格式问题
}
```

### 22.2 Analytics / Telemetry / OpenTelemetry 集成

**事件日志 API**（src/services/analytics/index.ts）：

```typescript
// 同步：fire-and-forget（主线程不阻塞）
export function logEvent(eventName: string, metadata: LogEventMetadata): void

// 异步：等待发送完成
export function logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>

// Sink 注册（在 initSinks() 前排队的事件不丢失）
export function attachAnalyticsSink(sink: AnalyticsSink): void
```

**元数据类型约束**：

```typescript
type LogEventMetadata = Record<string, number | boolean | undefined>
// 注意：禁止 string 类型！防止意外记录代码路径或文件名（PII 风险）
// 特殊 _PROTO_* 键：PII 标记列（Datadog 分流前被 stripProtoFields() 移除）
```

**多 Sink 路由**：
- `sink.ts`：Datadog（生产指标）
- `firstPartyEventLoggingExporter.ts`：BigQuery（内部分析）
- `datadog.ts`：Datadog 客户端

**OpenTelemetry 集成**（`@opentelemetry/sdk-node`）：支持分布式追踪（spans）、指标（counters/histograms）和日志，通过 OTLP 导出到 Prometheus 等后端。

### 22.3 Upstream Proxy（src/upstreamproxy/，2 文件）

见 Chapter 16.5。核心功能：CCR 容器环境中的 CONNECT→WebSocket 中继，为容器内的网络请求提供出口。

### 22.4 其他横切服务速览

| 服务目录 | 职责 |
|---------|------|
| services/oauth/ | OAuth 2.0 流程（授权码、刷新 Token） |
| services/lsp/ | Language Server Protocol 客户端（代码智能） |
| services/plugins/ | 插件加载与生命周期管理 |
| services/skillSearch/ | 技能发现（本地 + 远程） |
| services/SessionMemory/ | 会话级状态持久化 |
| services/AgentSummary/ | 后台 Agent 摘要生成 |
| services/MagicDocs/ | 自动文档生成 |
| services/PromptSuggestion/ | 提示建议 |
| services/autoDream/ | 自动 Dream 调度 |
| services/tips/ | 上下文感知提示 |
| services/compact/ | 压缩服务（autoCompact/microCompact） |
| services/contextCollapse/ | Context Collapse 实现 |
| services/remoteManagedSettings/ | 远程配置同步 |
| services/settingsSync/ | 设置同步上传 |
| services/teamMemorySync/ | 团队记忆同步监控 |
| services/policyLimits/ | 速率限制与策略执行 |

---

## Chapter 23 — 原生模块与垫片层

### 23.1 Shim 架构设计（shims/ 目录，11 个模块）

Claude Code 依赖部分原生模块（C++/Swift），在还原源码树中通过 Shim（垫片）提供兼容层：

```
shims/
├── ant-claude-for-chrome-mcp/   Chrome 扩展 MCP 桥接
├── ant-computer-use-input/      计算机控制输入处理
├── ant-computer-use-mcp/        Computer Use MCP 协议
├── ant-computer-use-swift/      macOS Swift 集成（辅助功能 API）
├── audio-capture-napi/          音频捕获（C++ Node Addon）
├── color-diff-napi/             颜色差异计算（C++）
├── image-processor-napi/        图像处理（C++，Sharp 替代）
├── modifiers-napi/              键盘修饰键处理（C++）
└── url-handler-napi/            URL scheme 处理（C++）
```

每个 Shim 的结构：
```
{shim-name}/
  package.json     # 本地 file:// 依赖声明
  index.ts/js      # TypeScript 绑定或 JavaScript 回退实现
```

**设计原则**：还原时对无法恢复的原生模块提供最小化回退实现，保证主程序可以启动和运行基本功能。

### 23.2 图像处理（image-processor-napi）

- 处理 `FileReadTool` 读取的图像文件
- 尺寸检测、格式转换（JPEG/PNG/WebP）
- 生成 base64 编码供 API 使用
- 回退：使用 `sharp` npm 包

### 23.3 音频捕获（audio-capture-napi）

- 麦克风输入捕获（用于语音模式）
- 音频流处理
- 回退：语音功能不可用

### 23.4 Computer Use / Chrome MCP 垫片

**Computer Use**（ant-computer-use-*）：

允许 Agent 控制 GUI 应用：
- `ant-computer-use-input/`：鼠标/键盘模拟
- `ant-computer-use-mcp/`：Computer Use MCP 协议服务器
- `ant-computer-use-swift/`：macOS 辅助功能 API（CGEvent, AXUIElement）

**Chrome MCP**（ant-claude-for-chrome-mcp/）：

Chrome 扩展与 Claude Code 的 MCP 桥接，允许 Agent 控制浏览器。

### 23.5 Yoga 布局引擎与 native-ts 绑定

**native-ts/yoga-layout/**（约 2,500 行 TypeScript）：

Facebook Yoga Flexbox 引擎的纯 TypeScript 移植，避免 C++ 原生依赖：

```typescript
// 值类型
type Value = { unit: Unit, value: number }
const UNDEFINED_VALUE: Value = { unit: 'undefined', value: NaN }
const AUTO_VALUE: Value = { unit: 'auto', value: NaN }

// 核心算法
function calculateLayout(node: YogaNode, width?: number, height?: number): void
function resolveValue(value: Value, ownerSize: number): number  // 百分比 → 像素
```

**native-ts/color-diff/**：颜色差异计算（可能用于 diff 渲染的颜色对比）

**native-ts/file-index/**：文件索引工具（快速文件发现/搜索）

### 23.6 vendor/ 目录

vendored 源码构建产物（构建时产物，非编辑对象）：

```
vendor/
├── audio-capture-src/     音频捕获原生模块 C++ 源码
├── image-processor-src/   图像处理原生模块 C++ 源码
├── modifiers-napi-src/    键盘修饰键原生模块 C++ 源码
└── url-handler-src/       URL 处理原生模块 C++ 源码
```

这些是对应 NAPI 模块的 C++ 源文件，用于从源码编译原生绑定（生产构建）。在还原源码树中，Shim 层提供了 JavaScript/TypeScript 回退。

---

## Chapter 24 — 附录

### 24.1 关键文件速查表

**核心执行链路**：

| 文件 | 行数 | 职责 |
|------|------|------|
| src/bootstrap-entry.ts | 6 | 最小入口，注入 MACRO |
| src/entrypoints/cli.tsx | 303 | 快速路径分发器 |
| src/main.tsx | 4,690 | 主 REPL 循环与初始化 |
| src/query.ts | 1,730 | Agent Loop 核心 |
| src/QueryEngine.ts | 1,295 | 多轮会话管理 |
| src/Tool.ts | 792 | 工具框架与权限 |
| src/tools.ts | ~800 | 工具注册总入口 |
| src/commands.ts | ~410 | 命令注册 |

**重要服务**：

| 文件 | 行数 | 职责 |
|------|------|------|
| src/services/mcp/client.ts | 3,000+ | MCP 客户端 |
| src/skills/loadSkillsDir.ts | 1,087 | 技能加载 |
| src/services/api/promptCacheBreakDetection.ts | 728 | Cache 断裂检测 |
| src/query/stopHooks.ts | 474 | Stop Hook 执行器 |
| src/interactiveHelpers.tsx | ~1,500 | 对话框管线 |
| src/ink/dom.ts | 485 | 自研 DOM 层 |

**关键常量文件**：

| 文件 | 关键常量 |
|------|---------|
| src/constants/toolLimits.ts | DEFAULT_MAX_RESULT_SIZE_CHARS=50K, MAX_TOOL_RESULTS_PER_MESSAGE_CHARS=200K |
| src/services/compact/autoCompact.ts | AUTOCOMPACT_BUFFER_TOKENS=13K, MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3 |
| src/query/tokenBudget.ts | DIMINISHING_THRESHOLD=500, COMPLETION_THRESHOLD=0.9 |

### 24.2 核心数据流总图

```
用户输入
   │
   ▼
src/entrypoints/cli.tsx
   │ (默认路径)
   ▼
src/main.tsx  ──────────────────────────────────────────────────────┐
   │                                                                │
   │ launchRepl()                                                   │
   ▼                                                                │
src/screens/REPL.tsx (React/Ink TUI)                                │
   │ 用户消息                                                        │
   ▼                                                                │
src/QueryEngine.ts::submitMessage()                                  │
   │                                                                │
   │ await query(params)                                             │
   ▼                                                                │
src/query.ts::queryLoop()  ←──────────────────────────────── ◄─────┘
   │                                                         │
   │ ┌─────────────────────────────────────────────────────┐ │
   │ │ Phase 2: 压缩管线（L0→L4）                           │ │
   │ │   L0: applyToolResultBudget()                       │ │
   │ │   L1: snipCompact()                                 │ │
   │ │   L2: microcompact()                                │ │
   │ │   L3: contextCollapse.collapse()                    │ │
   │ │   L4: autoCompact()                                 │ │
   │ └─────────────────────────────────────────────────────┘ │
   │                                                         │
   │ Phase 3: callModel() (Anthropic API)                    │
   │   ↓ 流式 token                                          │
   │   StreamingToolExecutor.addTool()  ← 边收边执行          │
   │                                                         │
   │ Phase 5: 工具执行                                        │
   │   StreamingToolExecutor.getRemainingResults()           │
   │   ├─ 并发安全工具 → 并行执行                              │
   │   └─ 非并发工具  → 串行执行                              │
   │                                                         │
   │ toolResults 追加到 messages                              │
   │ needsFollowUp = true → 下一轮 ──────────────────────────┘
   │ needsFollowUp = false → 返回 Terminal
   │
   ▼
src/QueryEngine.ts 汇总结果
   │
   ▼
React 组件 re-render
   │
   ▼
src/ink/dom.ts → Yoga Layout → Frame → Diff → ANSI → stdout
```

### 24.3 KV-Cache 数据流

```
API 请求构建
   │
   ▼ (assembleToolPool)
工具列表排序  ──────────────── 内置工具（按名排序）+ MCP 工具（按名排序）
   │                          └── 稳定前缀，MCP 变化不影响内置工具 Cache
   ▼
System Prompt 组装
   ├── 静态区（全局 Cache）  ← 不变内容，跨会话共享
   └── 动态区（会话 Cache）  ← 用户/环境相关
   │
   ▼
cache_control 标记放置
   ├── System Prompt 末尾
   ├── 工具列表末尾
   └── Messages 最后一条  ← 保护所有历史消息
   │
   ▼
API 调用 → 响应含 cache_read_input_tokens
   │
   ▼
promptCacheBreakDetection 分析
   ├── 计算 Cache 读取量下降
   ├── 识别断裂原因（系统/工具/消息变化）
   └── 发送 tengu_prompt_cache_break 事件
```

### 24.4 压缩决策树

```
每轮 query() 开始时
   │
   ▼
L0: applyToolResultBudget()  ← 永远执行
   │ 超出 per-tool 50K → 持久化到磁盘
   │ 超出 per-message 200K → 贪心持久化最大结果
   │
   ▼
L1: snipCompact()  ← HISTORY_SNIP 门控
   │ 裁剪旧消息，保护 Cache Prefix
   │
   ▼
L2: microcompact()  ← 每轮执行
   │ 时间衰减压缩 tool_result
   │
   ▼
检查 token 使用率
   ├── < 85%：不触发主动压缩
   ├── 85-90%：L3 Context Collapse
   │            一次 LLM 调用，折叠式摘要
   └── > 90%：L4 AutoCompact
                六阶段流水线（含熔断器）
                若 > 97%：强制阻塞压缩
```

### 24.5 权限决策流

```
工具调用 → validateInput() → checkPermissions()
                                     │
                              通用权限系统
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              alwaysDeny?      alwaysAllow?    alwaysAsk?
                 deny             allow           ask
                                                   │
                                          mode == 'auto'?
                                         /             \
                                       yes              no
                                        │               │
                                 ML 分类器          直接 ask
                                /         \
                          safe             risky/error
                          allow            ask/deny
```

### 24.6 术语表

| 术语 | 定义 |
|------|------|
| Harness | 给 LLM 提供工具、知识、权限边界的执行环境 |
| Agent Loop | while(true) 的 AsyncGenerator 查询循环 |
| Turn | 一次 LLM API 调用的完整周期 |
| Tool Output Budget | 单工具/单消息的输出字符上限（L0 压缩） |
| MicroCompact | 基于时间衰减的工具结果压缩（L2） |
| AutoCompact | 90% 窗口触发的六阶段全量摘要（L4） |
| KV-Cache | Anthropic API 的 Prompt 缓存机制 |
| Cache Prefix | API 请求中被缓存的前缀部分 |
| cache_control | 标记缓存分界点的 API 字段 |
| CacheSafeParams | Fork Agent 用于共享父请求 Cache 的参数集 |
| Withheld Error | 被暂扣的错误消息（等待恢复后再决定是否 yield） |
| Transition | 状态机的轮次转移原因（记录在 state.transition） |
| Swarm | 多 Agent 团队协作模式 |
| Mailbox | Agent 间异步消息队列 |
| DreamTask | 后台记忆整合 Agent 任务 |
| Worktree | Git 工作树，为并行 Agent 提供独立工作目录 |
| Bridge | 远程客户端控制本地 Claude Code 的桥接协议 |
| Shim | 原生模块的 JavaScript/TypeScript 垫片实现 |
| Feature Gate | 编译期（feature()）或运行时（GrowthBook）功能开关 |
| PermissionMode | 权限操作模式（default/auto/bypass/plan 等） |
| ToolUseContext | 工具执行的完整运行时上下文对象 |
| StreamingToolExecutor | 边接收 LLM 响应边执行工具的并发执行器 |

---

*文档生成于 2026-06-26，基于 `@anthropic-ai/claude-code 999.0.0-restored` 还原源码树分析。*

*覆盖范围：src/ 下全部 39 个子目录、22 个根级文件、shims/ 下 11 个原生模块垫片、vendor/ 构建产物。*