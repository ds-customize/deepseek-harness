# DeepSeek Harness 代码地图：框架与"一切皆为插件"

[English](codebase-map.md) | 中文

本文是一份中文导读：把 DeepSeek Harness（dsh）的框架机制、"一切皆为插件"的核心逻辑，以及各功能模块的代码位置映射到具体文件与行号，便于快速定位实现。

英文权威文档是 [architecture.md](architecture.md)（改 `packages/` 前必读）；本文不替代它，只做代码导航。Cordis 基础概念见 [cordis-primer.md](cordis-primer.md) 与 [cordis-tutorial/index.md](cordis-tutorial/index.md)。

行号以当前 `master` 为准，会随提交漂移；找不到时以文中给出的类名/函数名搜索即可。

## 1. 项目概览

DeepSeek Harness 是基于 vendored Cordis 的插件化 agent harness：模型适配器、工具注册表、会话日志、agent loop 本身全部是插件，任何部分都可以从配置替换。没有需要打补丁的特权核心——扩展 dsh 的方式是"在别的插件旁边挂载自己的插件"。

| 顶层目录 | 内容 |
|---|---|
| `vendor/` | Vendored Cordis 框架源码（`cordis`、`cosmokit`、`loader`、`include`、`timer` 等），同步流程见 [../vendor/README.md](../vendor/README.md) |
| `packages/` | 全部 `@deepseek-ai/dsh-*` 工作区包，按 `packages/<group>/<pkg>/` 组织，分组见 [../packages/README.md](../packages/README.md) |
| `apps/cli/` | `dsh` 命令行启动器：参数解析、profile 装配、模式分发（`src/bin.ts`、`src/args.ts`、`src/profile-boot.ts`、`src/dump-config.ts`） |
| `apps/web/` | Web 前端 Vite 构建（`@deepseek-ai/dsh-web-frontend`），产物 `dist/` 由 `dsh web` 的 host 提供 |
| `python/` | Python SDK 与捆绑运行时（见 `python/README.md`） |
| `native/` | `@deepseek-ai/node-addon-landlock-run` 原生模块源（见 `native/README.md`） |
| `examples/` | 可运行的 `cordis.yml` 叶子（见 `examples/AGENTS.md`） |
| `.agents/` | Agent 工作流、技能（`.agents/skills/`）与决策记录（`.agents/notes/`） |
| `docs/` | 架构文档与生成目录（见 [AGENTS.md](AGENTS.md)） |
| `scripts/` | 仓库门禁与生成器（门禁清单在 `scripts/run-gates.ts`） |
| `website/` | VitePress 文档站，投影自 `docs/` 选定页面 |

## 2. 框架底座：vendored Cordis

Cordis 源码在 `vendor/cordis/src/`：`context.ts`（上下文与服务仓库）、`service.ts`（`Service` 基类）、`fiber.ts`（fiber 生命周期）、`events.ts`（事件分发）、`registry.ts`、`reflect.ts`。五个核心概念：

1. **插件是实现 `Service` 的对象**：要么是带可选 `inject` 与 `apply(ctx)` 字段的函数插件，要么是 `Service` 子类，由 Cordis 挂载进当前上下文。
2. **上下文是服务仓库**：服务认领稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`），其他插件按键查找而不是导入具体实现。
3. **`inject` 声明依赖**：命名了必需服务的插件会等这些服务出现才激活，加载顺序由服务需求表达，不需要手工编排。
4. **类型化事件通信**：事件名通过 TypeScript 声明合并声明，按语义选分发模式。
5. **注册是可逆效应**：提示词段落、工具 schema、适配器、provider、监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，重载与拆卸可预测回退。

分发模式（详表见 [cordis-primer.md](cordis-primer.md)）：

| 模式 | 是否等待 | 顺序 | 返回值 |
|---|---|---|---|
| `emit` | 否 | 注册顺序观察 | 无 |
| `waterfall` | 否 | 注册顺序的环绕中间件 | 有 |
| `parallel` | 是 | 并行观察 | 无 |
| `serial` | 是 | 注册顺序 | 有 |

`ctx.waterfall` 是 around 中间件：监听器收到 `(...args, next)`，必须调用 `next()` 委托（不调用即短路）；单决策事件以短路为设计。新事件用 `@mode` 标注分发模式，生成目录据此校验声明与分发点一致。

Loader 配置：`cordis.yml` 只允许插件 `config` 与条目 `disabled` 使用 `!!js` 表达式，其余元数据保持字面量；环境选择插件时用 overlay（见 [cordis-primer.md](cordis-primer.md) 的 Loader Configuration 一节）。

## 3. "一切皆为插件"的核心逻辑

这一节是理解整个仓库的钥匙，共八条机制，每条都落到代码。

### 3.1 没有特权核心：产品 API 脊梁也是插件

构成"核心"的服务本身就是普通插件，与第三方扩展同一种挂载方式：

| 能力 | Service 类 | ctx key | 定义位置 |
|---|---|---|---|
| 会话日志与内存库 | `SessionStore` | `ctx.sessions` | `packages/core/session/src/index.ts:792` |
| 系统提示装配 | `SystemPrompt` | `ctx.systemPrompt` | `packages/core/system-prompt/src/index.ts:338` |
| 工具注册表与执行管道 | `ToolRuntime` | `ctx.tools` | `packages/core/tools/src/index.ts:787` |
| Agent 接口与注册表 | `AgentRegistry` | `ctx.agents` | `packages/core/agent/src/index.ts:256` |
| 默认循环驱动 | `AgentLoop` | `ctx.agentLoop` | `packages/core/agent-loop/src/index.ts:296` |
| LLM 适配器接缝 | `LlmRuntime` | `ctx.llm` | `packages/llm/llm/src/index.ts:284` |

每个包的 `src/index.ts` 默认导出这个 Service 子类供挂载；`/invariant` 子路径是伴随插件，向 `ctx.invariants`（`packages/runtime-diagnostics/invariants/src/index.ts:69`）注册包拥有的运行时断言。轻量贡献（如 `packages/core/agent-tool-presentation/src/index.ts:28` 的 `name` + `inject` + `apply`）就是函数插件。

### 3.2 运行时是配置组装出的插件树：profile / bundle / patch

一个运行中的 `dsh` 是启动时按有序分层组装出的插件树，装配代码在两处：

- **profile 机制**：`packages/boot/app-boot/src/profile.ts`。profile 是 `$DSH_HOME/profiles/<name>` 目录，`package.json` 的 `dsh.profile.bundles` 字段按序列出所叠放的 bundle（`profile.ts:1-22`）；模板 `PROFILE_TEMPLATES`（`profile.ts:114-117`）定义 `web` 与 `headless`。
- **启动装配**：`apps/cli/src/profile-boot.ts`。分层顺序（`composeProfile`，`profile-boot.ts:142-171`）：① profile 列出的 bundle 层（`dsh-base` 最先）；② profile 自己的 `cordis.patch.yml`；③ home 级 `$DSH_HOME/cordis.patch.yml`；④ 命令行 `--patch` overlay（按 argv 顺序）；⑤ 启动器注入的 overlay（shipped agent-preset 根、`DSH_TELEMETRY_DISABLED` 开关）。

所有层对空的根 `cordis.yml`（`PROFILE_ROOT_CONFIG`，`profile-boot.ts:60-64`）做**一次** `applyEntryPatches` 调用（`profile.ts:413-420`）。patch 按 id 定位行并整行替换 config（不做深合并），同 id 后写胜出；`--dump-config`（`apps/cli/src/dump-config.ts:30-52`）与真实启动走同一条装配路径，因此 dump 永远不漂移。

bundle 是 Cordis 配置行及其所挂代码的发行格式：`package.json` 的 `dsh.bundle.patch` 指向自己的 patch 文件（`profile.ts:41-45`）。

| Bundle | 位置 | 作用 |
|---|---|---|
| `dsh-base` | `packages/bundle/base/cordis.patch.yml` | 每个 profile 的第一层：一行 `insert` 挂载共享核心——运行时行（`timer`/`hmr`/`llm`/`session`/`typert` 等，`:16-37`）、标题、交互核心（`agent`/`agent-default-model`/`jobs`/`llm-retry`，`:55-72`）、settings 与 credentials（`:78-96`）、持久化与检索（`:98-127`）、遥测（`:148-161`）、沙箱与审批（`:163-205`）、全部工具行（`:207-253`）、goal/plan（`:256-279`）、上下文管理（token-meter/compaction/subagent 家族/workflow，`:281-341`）、守卫与裁剪（`:343-394`）、web 搜索（`:404-418`）、可被上层改写的"变量行"（`tools`/`system-prompt`/`agent-loop`/`fs-sandbox`/`llm-deepseek`，`:424-451`） |
| `dsh-web-app` | `packages/bundle/web-app/cordis.patch.yml` | 叠在 base 上加浏览器应用：host 侧行（`code-runtime`/`storage`/`api-gateway`/`webserver`/`web-runtime`，`:48-137`）、浏览器 `dsh.client` 行（`:151-274`，约 35 个 `ui-*`）、把 agent 侧工具行改为禁用以留在 host 平面（`:293-408`，内联说明哪些跨会话注册表必须留在 host）、`agent-presets`（`:420-424`）；运行时胶水插件在 `packages/bundle/web-app/src/index.ts`（name `web-app`，`:26`） |
| `dsh-headless` | `packages/bundle/headless/cordis.patch.yml` | 无服务器的一次性运行器：`headless-startup` 与 `headless-runner`（`:24-35`），runner 在 `packages/bundle/headless/src/index.ts`（name `headless-runner`，`:25`） |

想看本机实际启动的插件树：`dsh --profile web --dump-config`（`apps/cli/src/args.ts:133-134` 定义旗标，`apps/cli/src/bin.ts:45-49` 分发模式）。生成的 config 字段目录见 [config-catalog.md](config-catalog.md)。

### 3.3 注册即效应（registrations are effects)

一切贡献走 `ctx.effect()` / `ctx.on()`，注册的寿命与注册它的上下文严格相同。两个层次：

- **注册表存储**：`packages/core/scope/src/store.ts` 的 `ScopedLayers`（`:159`）持有全局层 + 按 scope 的覆盖层；`effect(ctx, action, options)`（`:226-266`）把原子变更包进生成器 `ctx.effect`，yield 出的 disposer 撤销变更、清空层并触发 `onChange`。`ctx.tools.register()`（`packages/core/tools/src/index.ts:1057-1061`）、`ctx.systemPrompt.section()`（`packages/core/system-prompt/src/index.ts:385-389`）等全部经由它。
- **生命周期组合**：拆卸顺序重要时用生成器效应 yield 精确的子 disposer。例：`SessionStore.create` 先 yield `enter` 的脱离器再广播 `session/created`（`packages/core/session/src/index.ts:836-839`）；`AgentRegistry.register` 同构（`packages/core/agent/src/index.ts:450-457`）；`AgentLoop` 的属主融合生命周期（`packages/core/agent-loop/src/index.ts:524-530`）；`ToolRuntime.presentAs` 组合层效应与提示词段落注册（`packages/core/tools/src/index.ts:951-971`）。

`Scope.rawDispose` 存在的原因：Cordis 按函数身份去重嵌套效应，组合拆卸必须 yield 精确的 `fiber.dispose`（`packages/core/scope/src/index.ts:104-112`）。

### 3.4 作用域：每个 Agent 拥有自己的注册世界

`packages/core/scope` 提供按 agent 隔离注册的原语：

- **铸造**：`createScope(ctx, key)`（`packages/core/scope/src/index.ts:137`）挂载共享空插件为独立 Cordis fiber，在其上下文打上 `kScope` 符号标签。agent 的作用域在 loop 内构造（`packages/core/agent-loop/src/agent.ts:94-95`）：`this.ctx = this.scope.ctx.extend({ agent: this })`，即插件通过 `agent.ctx` 拿到的上下文。
- **读取与父链**：`scopeOf(ctx)`（`index.ts:154`）沿上下文继承找最近标签；`bindScopeParent`（`:72`）一次性、带环检查地建父链；`scopeChainOf`（`:98`）返回最近优先的键链。链驱动两个方向：注册视图**向下**继承（子可见祖先的层，最近者遮蔽），事件准入**向上**扩展（祖先监听器收到后代事件）。web 的 preset 平面就靠 `agent → preset → global` 链解析（见第 8.4 节）。
- **作用域过滤分发**：`scopeTarget(base, key)`（`index.ts:170-185`）构造仅用于路由的载体：未打标签的监听器全局准入，打标签的监听器当且仅当其标签等于分发键或其祖先时准入。服务用它分发（如 `ctx.waterfall(scopeTarget(this, scope), 'system-prompt/assemble', ...)`，`packages/core/system-prompt/src/index.ts:532-535`）。`agentEvents`（`packages/core/agent/src/dispatch.ts:107-149`）把 agent 主体融合进载体，保证载荷里的 `agent` 与路由键永不分离。

效果：给某一个 agent 挂一个工具/提示词段落/监听器，写在它的 `agent.ctx` 上；agent 销毁时其 fiber 卸载，全部注册随之回退。

### 3.5 类型化事件就是扩展点

事件分三个域，选对域是多数改动的第一个决策（事件的生产者/消费者全景见 [event-producer-consumer.md](event-producer-consumer.md)）：

| 域 | 载体 | 用途 | 典型成员与位置 |
|---|---|---|---|
| 会话事件（durable） | `SessionEventMap` 声明合并，追加进日志并经 `session/event` 广播 | 事实必须能在重载后存活 | `turn/start`、`step/*`、`user/message`、`assistant/*`、`tool/*`（`packages/core/session/src/types.ts:236-333`）；后续包合并 `agent/inbox/spliced`、`compaction/*`、`llm/retry`、`session/title`、`plan/mode` 等 |
| Agent 事件（live） | `Events` 接口声明合并，携带活 `Agent` | 观察或拦截进行中的工作 | `agent/created`、`agent/status`、`agent/inbox/*`（`packages/core/agent/src/runtime-types.ts:146-292`） |
| 能力事件 | 接缝自己的 `Events` 声明 | 给接缝挂策略与适配器，不导入 loop | `tools/pre-execute` 等（`packages/core/tools/src/index.ts:142-208`）、`fs/write-intent` 等（`packages/fs/fs/src/index.ts:49-77`）、`llm/stream`（`packages/llm/llm/src/index.ts:64`） |

规约：事件的 JSDoc 需要 `@mode` 与载荷 `@param`；`SessionEventMap` 成员默认"读取时必需"——不认识其类型的构建会拒绝该日志，除非事件带信封的 `ignorable: true`；只有结构性格式变更才提升 `SESSION_FORMAT_VERSION`。瀑布监听器必须调用 `next()`。

### 3.6 模型可见 ⟺ 已记录（model-visible means logged）

任何到达模型请求的内容都必须能从会话日志重建。实现闭环：

- `Session.append`（`packages/core/session/src/index.ts:604`）做 lossless-JSON 快照校验、拒绝重入、以 `seq = log.length` 构造冻结事件，并在入日志前先经 `SurfaceManager.validateNext`（`packages/core/session/src/surface.ts:398`）验证表面转移。
- `Session.deriveMessages`（`index.ts:726`）沿表面节点投影模型历史（`deriveEventMessage`，`surface.ts:83`），按节点缓存；压缩替换代际变化时重建。
- loop 每次模型请求都重新投影：`this.session.deriveMessages()`（`packages/core/agent-loop/src/agent.ts:341`）。
- `packages/core/agent-loop/src/invariant.ts` 的运行时断言验证 loop 构造的 LLM 请求确实可从日志重建。

因此新增模型可见输入的路径固定：扩展 `SessionEventMap`，从日志渲染。分叉、恢复、转写、遥测、持久化全部派生自这条流。

### 3.7 能力接缝三角色（capability seam）

一个接缝是可替换的能力，含三个角色：**Service Definition**（声明接口与 ctx key）、**Service Provider**（实现它）、**Consumer**（使用它，通常是模型可见工具）。一个包可以合并角色，但只有一个角色不成接缝；新增能力意味着三个角色一起设计（接缝全景见 [capability-seams.md](capability-seams.md)，术语见 [glossary.md](glossary.md)）。

依赖规约（`packages/README.md`）：扩展插件依赖 Service Definition，永不依赖具体 provider——这是换一个 provider 就改变整个产品的结构保证。

最能体现接缝威力的是**执行世界共享**：`SubprocessRuntime` 的三个动词 `resolveExecutable`/`spawn`/`spawnTerminal`（`packages/subprocess/subprocess/src/index.ts:118-139`）与 `FileSystem`（`packages/fs/fs/src/index.ts:86`）是仅有的两个 OS 适配器。`bash-local`、`terminal-bash`（PTY 经 `ctx.subprocess.spawnTerminal`，`packages/terminal/terminal-bash/src/index.ts:110`）、`lsp-stdio`（经 `ctx.subprocess` 与 `ctx.fs`，`packages/lsp/lsp-stdio/src/index.ts:146-157`）都不直接碰 OS。把 `subprocess-local` + `fs-local` 换成 `subprocess-e2b` + `fs-e2b`（`packages/e2b/`），Bash/PTY/LSP/搜索就整体迁入远程沙箱，无需任何 provider 分叉；同理 `fs-sandbox`/`bash-sandbox` 在本地围栏。

### 3.8 循环本身可替换

`Agent` 接口（`packages/core/agent/src/runtime-types.ts:64-144`：`id`/`session`/`inbox`/`status`/`ctx`/`cancel`/`whenIdle`/`send`/`steer`/`inject` 等）对 loop 零依赖。唯一的具体实现 `ReactLoopAgent` 是 `agent-loop` 的包内私有类（`packages/core/agent-loop/src/agent.ts:64`），通过 `ctx.agents.setFactory(this)`（`agent-loop/src/index.ts:350`；注册机制 `packages/core/agent/src/index.ts:372-388`）注入注册表，消费侧只经 `ctx.agents.create/resume`（`:405-430`）。换掉 loop = 挂一个提供新 factory 的插件。规约"插件优先于改 loop"：新行为落在文档化的扩展点上；确实要改 `agent-loop` 时必须同步更新 [architecture.md](architecture.md)。

## 4. 核心包代码地图（packages/core/）

| 包 | 职责 | ctx key |
|---|---|---|
| `core/scope` | 按 agent 的作用域注册原语 | 库，无 key |
| `core/session` | 追加式 `SessionEvent` 日志与内存库；LLM 消息历史从它派生 | `ctx.sessions` |
| `core/agent` | `Agent` 接口、活注册表、发起方作用域、`agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 默认循环驱动：turn/step 生命周期 | `ctx.agentLoop` |
| `core/system-prompt` | 提示词段落与工具 schema 装配 | `ctx.systemPrompt` |
| `core/tools` | 工具注册表与守卫执行管道 | `ctx.tools` |
| `core/agent-default-model` | 入口创建无本地模型选择的 Agent 时用的部署默认 | `ctx.agentDefaultModel` |
| `core/agent-tool-presentation` | preset 携带的行：工具以 `native`/`code`/`both` 形态呈现给模型 | 无（函数插件） |

### 4.1 core/scope

`src/index.ts`（铸造/读取/父链/载体）与 `src/store.ts`（`ScopedLayers` 分层注册存储，变更即效应）。关键导出：`createScope`（`index.ts:137`）、`scopeOf`（`:154`）、`scopeTarget`（`:170`）、`bindScopeParent`（`:72`）、`scopeChainOf`（`:98`）、`ScopedLayers.effect`（`store.ts:226`）。`scoped-events.generated.ts` 是生成的作用域事件表，供 `invariant.ts` 校验每次作用域过滤分发都携带与载荷主体对应的载体。

### 4.2 core/session

| 文件 | 内容 |
|---|---|
| `src/index.ts` | `Session` 类与 `SessionStore`（`:792`）；事件 `session/created`/`disposed`/`event`/`flush`（`:54-85`） |
| `src/types.ts` | `SessionEventMap`（`:236-333`）、`SessionEvent`、`SurfaceOp`、`SessionHeader` |
| `src/surface.ts` | 有序表面投影：`deriveEventMessage`（`:83`）、`foldSurface`、`SurfaceManager`（`:398`，含压缩 `replace` 区间拼接与来源校验） |
| `src/json.ts` | lossless-JSON 校验/快照 |
| `src/request-header.ts`、`src/runtime-context` 类的请求上下文折叠 | 请求头/上下文的规范化折叠 |
| `src/repair.ts` | 崩溃恢复：为孤儿 turn 合成闭合事件 |
| `src/chunk-rows.ts` | `assistant/chunk` 增量运行的存储打包 |
| `src/preparation.ts` | 未发布 `Session` 的 `Disposable` 包装 |
| `src/invariant.ts` | 单调 seq、turn/step 包络、工具调用/结果配对断言 |

主要 API：`Session.append`（`index.ts:604`）、`requestHeader`（`:670`）、`requestContext`（`:691`）、`deriveMessages`（`:726`）、`surface`（`:431`）；`SessionStore.create`（`:830`）、`prepare`（`:863`）、`enter`（`:913`）、`announce`（`:968`）、`flush`（`:1022`，持久化插件在此排空）、`fork`（`:1081`）。

### 4.3 core/agent

| 文件 | 内容 |
|---|---|
| `src/index.ts` | `AgentRegistry`（`:256`）与发起方作用域（`AsyncLocalStorage`）：`currentInitiator`（`:309`）、`withInitiator`（`:341`）、`setFactory`（`:372`）、`create`（`:405`）、`resume`（`:424`）、`register`（`:450`） |
| `src/runtime-types.ts` | `Agent` 接口（`:64-144`）与 `agent/*` 事件（`:146-292`：emit 类 `created`/`status`/`inbox/*`；waterfall 类 `pre-step`/`request`/`request-error`；serial 类 `turn-stopping`） |
| `src/dispatch.ts` | 融合分发 `agentEvents`（`:107`）与 `assembleContextFor` |
| `src/inbox.ts` | `Inbox`：收件箱事件的增量投影 |
| `src/consumed-work.ts` | 已消费/未运行丢弃工作的单遍核算 |
| `src/model-selection.ts` | 会话内可变模型选择接入 `system-prompt/assemble` 与 `agent/request` 瀑布 |

输入经一个收件箱到达驱动器：部分消息立即唤醒，注入的上下文在收件箱等待下一条消息。

### 4.4 core/agent-loop

| 文件 | 内容 |
|---|---|
| `src/index.ts` | `AgentLoop`（`:296`，`inject = ['agents','sessions','llm','tools','systemPrompt']`）：创建事务 `prepare`/`publish`/`dispose`、`FactoryOwnership` 拆卸簿记、`create`（`:589`）、`resume`（`:653`） |
| `src/agent.ts` | `ReactLoopAgent`（`:64`）：Phase 状态机（`:38-48`）、`wakeDriver`（`:172`）、`kick`（`:210`）、`turn`（`:246`）、`step`（`:332`）、`buildRequest`（`:407`） |
| `src/tool-calls.ts` | `executeToolCalls`（`:59`）与 `runGroup`（`:121`）：有界滚动并行池，`maxParallelToolCalls` 实时读配置（默认 10，`constants.ts`），按模型顺序提交结果，中止时为跳过的调用补写合成错误结果 |
| `src/runtime-context.ts` | 最近运行时上下文快照的持久跟踪 |
| `src/invariant.ts` | "loop 构造的请求可从日志重建"断言 |

### 4.5 core/system-prompt

`src/index.ts`：`SystemPrompt`（`:338`）持有有序段落、命名上下文段、工具 schema 与 `{{variable}}` 变量；事件 `system-prompt/assemble`（waterfall，`:31`）与 `system-prompt/change`（`:37`）。API：`section`（`:381`）、`context`（`:398`）、`suppressRuntimeContext`（`:415`）、`tools`（`:430`）、`variable`（`:446`）、`assemble`（`:467`）；渲染器 `renderPrompt`（`:212`）。构造时自注册 harness 身份段落与 `deployment:persona` 槽位（`:357-369`）。

### 4.6 core/tools

| 文件 | 内容 |
|---|---|
| `src/index.ts` | `ToolRuntime`（`:787`）：注册/限制/守卫/Code Mode 呈现与分阶段执行管道 |
| `src/schema.ts` | `defineTool`（`:545`）与参数 schema 词汇 |
| `src/json-schema.ts` | 受支持 JSON-Schema 的值校验 |
| `src/code-mode.ts` | Code Mode 的保留传输工具 `run_code`（`RUN_CODE_NAME` `:20`、`createRunCodeTool` `:294`） |
| `src/ts-types.ts`、`src/py-types.ts` | Code Mode 的 TS/Python SDK 渲染器 |
| `src/presentation.ts` | `ToolCallView`/`ToolResultView` 渲染意图词汇（`generic`/`terminal`/`diff`/`locations`） |
| `src/types.ts`、`src/invariant.ts` | `tool/code-dispatch*` 会话事件合并；管道单调性与冻结终态断言 |

API：`register`（`:1037`）、`restrict`（`:1071`）、`guard`（`:1110`）、`presentAs`（`:946`）、`schemas`（`:1234`）、`execute`（`:1342`）。事件：`tools/pre-execute`（`:152`，允许/拒绝/询问门）、`tools/execute`（`:163`，超时/重试/指标的环绕分发）、`tools/post-execute`（`:175`，接受/替换/阻断）、`tools/code-dispatch-log`（`:189`）、`tools/result`（`:197`，只读观察冻结终态）、`tools/change`（`:207`）。分阶段调度器 `TOOL_RUNTIME_SCHEDULER`（`:466`）的四个阶段在 `:1459`/`:1569`/`:1609`/`:1631`。

### 4.7 core/agent-default-model 与 core/agent-tool-presentation

`agent-default-model` 的 `AgentDefaultModelConfig`（`src/index.ts:64`）经 settings 命名空间存部署默认（base bundle 配置为 `deepseek-official` / `deepseek-v4-flash`）。`agent-tool-presentation`（函数插件，`src/index.ts:28-72`）按 preset 行调用 `ctx.tools.presentAs(mode)`，code 模式经 `ctx.inject(['codeRuntime'], ...)` 等待运行时就绪。

## 5. 回合执行流与工具管道

一个 **step** 是一次模型请求加它调用的工具；一个 **turn** 是零或多个 step。事件序（durable 会话事件为粗体域，其余是跨三域的活扩展点）：

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   reject | enter(messages)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     append entered messages as user/message
     derive model history from the log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

实现位置（均在 `packages/core/agent-loop/src/agent.ts` 除非另注）：`turn/start` 追加在 `turn()`（`:246-330`）开头；`agent/pre-step` 提议在 `:266`；`step/start` 与 `user/message` 在 `:279-284`；`step()`（`:332-401`）内 `buildRequest`（`:407-495`，跑 `agent/request` 瀑布、折叠 `request/header`/`request/context`）→ `ctx.llm.stream` 逐块追加 `assistant/chunk`（`:345-349`）→ 失败走 `agent/request-error` 瀑布（`:355-365`）→ `assistant/message` 引用块 seq（`:381-390`）→ 无工具调用即完成，否则 `executeToolCalls`（`packages/core/agent-loop/src/tool-calls.ts:59`）。turn 收尾是数据驱动的：step 结束且无 next-step 输入时等待 `agent/turn-stopping`（serial，`:296`），再读一次收件箱，新转向继续跑 step，没有则关闭 turn。

工具执行管道的五个事件与位置见第 4.6 节；详细语义见 [tool-execution-pipeline.md](tool-execution-pipeline.md) 与 [agent-lifecycle.md](agent-lifecycle.md)。取消与错误恢复见 [subsystems/core.md](subsystems/core.md)。

## 6. 能力接缝代码地图

每个接缝的三角色一览。路径省略 `packages/` 前缀；"工具"列是模型可见的工具名。

| 组 | Service Definition（ctx key） | Provider | Consumer（工具） |
|---|---|---|---|
| `llm/` | `llm/llm`：`ctx.llm`（`llm/llm/src/index.ts:46`），抽象 `LlmAdapter`（`:180-233`），`llm/stream` 瀑布（`:64`），`StreamChunk` 词汇（`llm/llm/src/types.ts:291-300`） | `llm/llm-deepseek`（`DeepSeekAdapter`，`adapter.ts:158`，路由 `deepseek-official`）、`llm/llm-pi-ai`（多 profile 动态路由）、`llm/token-meter`（`ctx.tokenMeter`）、`llm/llm-retry`（消费 `agent/request-error`，写 `llm/retry` 会话事件） | 无工具；循环本身是消费方 |
| `shell/` | `shell/shell`：`ctx.shell`（`shell/shell/src/index.ts:40`），抽象 `resolve(request): spec`（`:85`） | `shell/bash-local`（经 `ctx.subprocess` 跑 `bash -c`）、`bash-sandbox`（`resolve` 盖 `sandboxPolicy` 默认）、`pwsh-local`、`pwsh-sandbox`、`shell-env`（`ctx.shellEnv`，可信 `DSH_*` 注册表） | `shell/tool-bash`（工具 `bash`）、`tool-pwsh`（`pwsh`）、`tool-bash-persistent`（PTY 上的持久 `bash`） |
| `subprocess/` | `subprocess/subprocess`：`ctx.subprocess`（`subprocess/subprocess/src/index.ts:68`），三动词 `resolveExecutable`/`spawn`/`spawnTerminal`（`:118-139`） | `subprocess/subprocess-local`（node-pty）、`e2b/subprocess-e2b` | 无直接工具；bash/PTY/LSP/搜索的地基 |
| `terminal/` | `terminal/terminal`：`ctx.terminals`（`terminal/terminal/src/index.ts:48`） | `terminal/terminal-bash`（PTY 委托 `ctx.subprocess.spawnTerminal`） | `terminal/tool-terminal`：`terminal_open/send/read/signal/close/list`（`tool-terminal/src/index.ts:162-386`） |
| `fs/` | `fs/fs`：`ctx.fs`（`fs/fs/src/index.ts:44`），事件 `fs/write-intent`/`edit-intent`（waterfall）、`fs/observed`（emit）（`:49-77`） | `fs/fs-local`、`fs/fs-sandbox`（模式围栏）、`e2b/fs-e2b`、策略插件 `fs/fs-observation-policy` | `fs/tool-fs`：`read`/`write`/`edit`/`read_image`（`tool-fs/src/*.ts`）；`fs/tool-fs-search`：`glob`/`grep`（打包的 ripgrep 经 `ctx.subprocess`）；`fs/tool-str-replace-editor` |
| `web/` | `web/web`：`ctx.web`（`web/web/src/index.ts:35`） | `web-search-exa`、`web-search-perplexity`、`web-search-deepseek`、`web-fetch-http` | `web/tool-web`：`web_search`、`web_fetch` |
| `subagent/` | `subagent/subagent`：`ctx.subagents`（`subagent/subagent/src/index.ts:129`），事件 `subagent/provider-added`/`start`/`end` | `spawn-in-process`（默认 `spawn`）、`fork-in-process`、`subagent-acp`、`subagent-codex`、`subagent-claude-code`、`subagent-dsh-sdk` | `tool-subagent`（默认名 `subagent`）、`tool-subagent-control`（`send_message`/`interrupt_agent`/`list_agents`）、`tool-subagent-report`（子作用域 `report`） |
| `lsp/` | `lsp/lsp`：`ctx.lsp`（`lsp/lsp/src/index.ts:38`），恰好四个操作：定义/引用/实现/悬停 | `lsp/lsp-stdio`（通用 stdio host，经共享执行世界） | `lsp/tool-lsp`：工具 `lsp` |
| `skill/` | `skill/skill`：`ctx.skills`（`skill/skill/src/index.ts:284`），事件 `skills/change` | `skill/skill-filesystem`、`skill/skill-badge` | `skill/tool-skill`：工具 `skill`（目录 + 加载器） |
| `sandbox/` | `sandbox/sandbox`：`ctx.sandbox`（`sandbox/sandbox/src/index.ts:146`），`confine(argv, policy)` 失败关闭（`:151-175`）；`sandbox-policy`：`ctx.sandboxPolicy`（每会话持久模式） | `sandbox-local`（bwrap/Landlock/Seatbelt/Windows ACL 梯）、`sandbox-windows-acl` | 无独立工具；升级出现在 `bash`/`read`/`write`/`edit` 的 schema 里 |
| `e2b/` | `e2b/e2b`：`ctx.e2b`（`e2b/e2b/src/index.ts:63`，沙箱生命周期与共享 cwd） | `e2b/fs-e2b` + `e2b/subprocess-e2b`（整组换执行世界） | 复用 fs/shell 工具 |
| `compaction/` | `compaction/compaction`：`ctx.compaction`（`compaction/compaction/src/index.ts:81`），durable 事件 `compaction/start`/`summary`/`end`/`prune`（`types.ts:17-86`） | `compaction/compaction-basic` | `compaction/command-compact`：人类命令 `/compact`；`compaction-tool-result-pruner`：`ctx.toolResultPruner` |
| `context/` | 无单一接缝 | — | `agent-instructions`（AGENTS.md 工作区指令，基线注入 + fs 触碰重注入）、`time-context`、`tmux-context`（均经 `agent/pre-step`）、`session-reference`（`ctx.sessionReferenceResolver`） |
| `jobs/` | `jobs/jobs`：`ctx.jobs`（`jobs/jobs/src/index.ts:29`） | `jobs/jobs-local` | `jobs/tool-jobs`：`job_output`/`job_list`/`job_kill` |
| `workflow/` | `workflow/workflow`：`ctx.workflowEngine`（`workflow/workflow/src/index.ts:31`），事件 `workflow/start`/`phase`/`log`/`agent-start`/`agent-end`/`end` | `workflow/workflow-worker-thread` | `tool-workflow`（默认名 `workflow`）、`tool-ralph`（固定策略 `ralph`） |
| `spill/` | `spill/spill`：`ctx.spillStore`（`spill/spill/src/index.ts:23`） | `spill/spill-local` | `spill/spill-policy`：监听 `tools/post-execute` 的工具结果外溢策略 |
| `attachment/` | `attachment/attachment`：`ctx.attachments`（`attachment/attachment/src/index.ts:22`） | `attachment/attachment-local`（内容寻址） | 无工具；字节经用户输入/提供方产出提交进入 |
| `code-runtime/` | `code-runtime/code-runtime`：`ctx.codeRuntime`（`code-runtime/code-runtime/src/index.ts:89`） | `code-runtime-worker-thread`（TS，worker 线程隔离） | `run_code`（保留传输工具，`packages/core/tools/src/code-mode.ts:20`，`tools: { mode: 'code' }` 时由 `ToolRuntime` 实例化） |
| `mcp/` | 无接缝（纯桥） | — | `mcp/mcp-client`：连接外部 MCP 服务器，工具以 `mcp__<server>__<name>` 注册进 `ctx.tools`（`mcp-client/src/tools.ts:140-162`） |
| `typert/` | `typert/protocol` 声明 `ctx.typert`（`typert/protocol/src/types.ts:487`） | `typert/registry`（`TypertRegistry`，`service.ts:446`） | `typert/loader`（发现 loader 条目并注册生成的宿主工件）；`typert/generator` 是构建期库 |

两个值得记住的模式：

- **shell 的显式 resolve 步**：`resolve(request): spec` 是接缝拥有的显式默认化步骤（`packages/shell/shell/src/index.ts:85`；本地实现补 `workdir`、钳制超时并透传 `sandboxPolicy`，`packages/shell/bash-local/src/index.ts:146-171`），`run(spec)`/`start(spec)` 只接受已解析的 spec，永不隐藏 `?? default`。围栏子类只需覆写 `resolve` 盖默认（`bash-sandbox/src/index.ts:84-85`）。这是"包边界显式优于隐式"规约的模板。
- **LLM 适配器接缝**：`LlmRuntime` 把每次调用包进 `llm/stream` 瀑布（`packages/llm/llm/src/index.ts:917-927`），把适配器抛错转成终态 `finish` 块（`:931-939`）；路由注册 `registerAdapter`（`:338`）返回原子 `replace` 句柄，settings 变更即可换路由（DeepSeek 提供方 `packages/llm/llm-deepseek/src/index.ts:251-266`）。

## 7. 会话数据平面（packages/session/ 与 packages/session-query/）

持久化单位就是 `SessionEvent` 本身；`SessionHeader` 单独随行。

| 包 | 作用 | 关键位置 |
|---|---|---|
| `session/session-persistence` | 持久化 Service Definition：`ctx.sessionPersistence` | `src/index.ts:61-63`，默认导出 `:243` |
| `session/session-persistence-jsonl` | JSONL 后端：`<root>/<规范化cwd>/<编码id>/session.jsonl.zstd`（校验和头帧 + 追加帧） | `src/index.ts:967` |
| `session/session-persistence-sqlite` | 单库多会话：`events` 表 1:1 行（含 `assistant/chunk`），WAL，单调 `SCHEMA_VERSION` | `src/index.ts:414` |
| `session/session-checkpoint-policy` | 零配置函数插件：每次模型请求前、顶层工具副作用前、每个 `agent/pre-step` 检查点 | `src/index.ts:15` |
| `session/session-projection` | 投影注册表 `ctx.sessionProjections`：一致读切面、按键注册、状态版本冲突拒绝 | `src/index.ts:25-27`、`:428` |
| `session/session-projection-cache` | 投影缓存 `ctx.sessionProjectionCache`（web 配置 200 事件/5s 落盘） | `src/index.ts:31-33` |
| `session/session-stats` | 全日志 turn/step 计数投影 | `src/index.ts:18` |
| `session/session-title` 及 LLM 变体 | 日志背书标题：接受的每次修订是 log-only `session/title` 事件；`ctx.sessionTitle` | `session-title/src/index.ts:89-91` |
| `session/session-telemetry` + `-otel` | 遥测后端接缝 `ctx.sessionTelemetry` + OTLP/HTTP 导出器（`DSH_TELEMETRY_MODE` 控制，默认关） | `session-telemetry/src/index.ts:20-22`、`session-telemetry-otel/src/index.ts:301` |
| `session-query/session-query` | 检索接缝 `ctx.sessionQuery`：活库优先合并读、事件三表面分类（`current`/`shadowed`/`log-only`） | `src/index.ts:69-71` |
| `session-query/session-query-sqlite` | FTS5 全文检索：字面短语、谓词预算、世代绑定游标；`openAt: never` 默认不开库 | 组 README 及 `src/index.ts:69-72`（启动器持有路径） |
| `session-query/session-log-export` | 浏览器 `/export` 命令 + `GET /api/session.export` ZIP 流 | 组 README |
| `session-query/tool-session-query` | 模型可见的 `session_search` 等工具：cwd 相等授权、游标不外露 | 组 README |

## 8. 人机协作、编排与自修改

### 8.1 interaction/（人机协作平面）

| 包 | 作用 | ctx key |
|---|---|---|
| `interaction/user-approval` | 通道中立的一次性审批接缝：结果 `allowed-once`/`rejected`/`cancelled`/`unavailable`，失败关闭；`approval/request` 瀑布 + `approval/asked`/`decided` 审计对 | `ctx.approval`（`src/index.ts:18-20`） |
| `interaction/permission-presets` | 面向用户的权限预设（read-only / workspace-write / danger-full-access），捆绑 `sandbox/mode` 与 `approval/policy` | `ctx.permissionPresets`（`src/index.ts:37-39`） |
| `interaction/commands` | 插件拥有的人类命令注册表，不经模型轮次直接分发；作用域内子注入可遮蔽同名 | `ctx.commands`（`src/index.ts:91-93`） |
| `interaction/user-questions` | 工具/权限插件用来暂停并向人类提问的服务 | `ctx.userQuestions`（`src/index.ts:15-17`） |
| `interaction/tool-ask-user` | 模型可见的 `ask_user_question` 工具 | 消费 `ctx.userQuestions` |

### 8.2 状态与反馈类

`plan/plan-mode`（日志化 plan 状态：`plan/mode` log-only 事件、`/plan` 命令、经审阅的 `exit_plan_mode`；`ctx.planMode`，`src/index.ts:58-60`）；`todo/tool-todo`（整表替换的 `todo_write` 工具 + `todo/write` 快照事件）；`goal/`（`dsh-goal` 同会话目标 `ctx.goals`、`goal-round-driver` 把 armed goal 变成顺序 goal 轮、`command-goal` 的 `/goal`、`tool-goal` 的模型工具）；`schedule/schedule`（会话内 durable 提醒：`after_seconds`/`at`/`every_seconds`，只装在更晚创建的运行时根 agent 上）；`feedback/`（`command-feedback` 不可变的 `/remark`；`message-feedback` 每条助手消息的点赞/点踩 + 附注，存本地 sidecar，永不进入对话）。

### 8.3 guard/ 与 hooks/

`guard/repeat-tool-reminder`：咨询式循环破坏器——统计连续同参工具调用，按阈值（base 配置 `[3,5,8]`）注入升级提醒，从否决。`guard/timeout-policy`：零配置单监听器，从每个工具声明的 `timeoutMs` 起合作期限，超时返回结构化 `TOOL_TIMEOUT`。

`hooks/hook-protocol` 是 Claude Code / Codex 钩子线协议共享库（匹配器、退出码/stdout 编解码、`ctx.shell` 执行、最严格合并），不是 Cordis 插件；`hooks-claude-code` 与 `hooks-codex` 是两个方言桥，把用户已有的钩子配置接到 harness 的拦截点上。

### 8.4 preset/（每会话 agent 组合）

`preset/agent-presets`：preset 是带一个 `agent.cordis.yml` 的目录，挂到常设作用域后每个加入的会话拥有自己的工具/提示词段落而会话间保持隔离（`ctx.agentPresets`，`src/index.ts:70-72`）。机制正是第 3.4 节的链：`dsh-tools`/`dsh-system-prompt` 把注册归档进调用方上下文的作用域层，`agent → preset → global` 最近者遮蔽。`preset/persona` 是可组合的人设 preset 行。web bundle 把大量 agent 侧工具行禁用、改由 preset 平面按会话供给（`packages/bundle/web-app/cordis.patch.yml:293-424`），其中跨会话注册表（jobs/skill/goal/token-meter/subagent）留在 host 平面并在 patch 内联说明理由。

### 8.5 extensions/（agent 修改自己的运行时）

设计记录见 `.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md`。

| 包 | 角色 |
|---|---|
| `extensions/tool-cordis` | 模型可见的自修改工具集：`cordis_inspect`（检查服务/fiber/工具/动态包及 `api`/`events`/`client` 表面）、`cordis_define`（记录 + 语法预检，零副作用）、`cordis_run`（求值宿主半 + 递交浏览器半）、`cordis_stop`、`cordis_undefine`（`src/index.ts:26-35` 及 `src/inspect.ts`、`src/providers.ts`） |
| `extensions/cordis-host-runner` | 动态包宿主半：`define`/`undefine` 只记账（铸 `dyn-<n>`）；`run` 在 `vm` 中于 `cordis-dynamic` 组 fiber 下求值，含浏览器半的包触发可应答的 `cordis/request-run` 往返（人批准）；`ctx.dynamicCordisRunner`（`src/index.ts:81-84`） |
| `extensions/cordis-client-runner` | 浏览器半：订阅公告，用守卫门面在真实 fiber ctx 上求值，经 `loader.create` 入座 |
| `extensions/ui-cordis` | 运行批准的全局覆盖面板 + 转写里的定义卡片 |

### 8.6 boot/（共享启动胶水）

`boot/app-boot/src/index.ts` 是通道中立的启动库（不是插件）：分层 `.env` 装载（`:177-198`）、失败即响的 Loader 守卫 `installFailLoud`（`:578-649`）、`cordis:include`/`cordis:group` 内建与根 include 挂载（`:486-529`）、用户 patch 的热重载协调 `watchUserPatches`（`:232-265`）、装配后审计（`:658-725`）、总启动序列 `boot()`（`:757-802`）、`harness:source` 提示词段落（`:805-829`）。profile 机制在同包 `src/profile.ts`（见第 3.2 节）。`boot/cmdline` 提供启动器到应用的命令行交接：`ctx.cmdlineArgs` 与 `ctx.appExit`（`src/index.ts:45-49`）。

## 9. 远程与嵌入表面

### 9.1 api/ 与 typert/（远程 BFF 与类型化 RPC）

`api/gateway` 是 Typert 网关：宿主侧 `TypertGatewayService`（`src/index.ts:90`，`inject = ['typert']`）把分发绑定到传输的 `/api` 拦截（`:104-111`），端点认领来自严格生成描述符与从 `ctx.reflect.props` 扫描的 SRC 标记（`:114-137`）；客户端半在 `src/client/index.ts` 安装类型化 Client Remote（`ctx.remote`）。`api/remotes` 是宿主 Agent/Session 查找策略 + 客户端 Remote 贡献装配（命令、目标、动态包、插件清单、消息反馈），转发事件白名单 `API_REMOTE_FORWARDED_EVENTS` 带编译期形状门（`src/index.ts:33-41`）。`typert/` 三件套：`generator`（构建期从源码生成工件）、`registry`（运行时注册表 `ctx.typert`）、`loader`（发现条目并注册）。

### 9.2 host/ 与 client/（Web GUI 的两半）

host 半是 API 网关 + HTTP 路由服务器：`host/webserver`（`ctx.webServer`，`node:http` 路由注册 + 唯一 fallback 座，`src/index.ts:59-127`）、`host/apiproxy`（共享宿主 API 网关，`ctx.apiProxy`）、`host/frontend-static`（SPA dist 落在 fallback 座）、`host/directory-picker*`（目录选择接缝 + 原生/浏览后端 + 自适应组合）、`host/plugin-inventory`（只读 Loader 条目投影）。

client 半是浏览器外壳：`client/web` 的 `src/boot.tsx` 是内核（解析 `window.__DSH_BOOT__` → 建模块系统 → 预取 → 挂 vendored Cordis Loader → fiber 全扫 → 翻转就绪信号）；`client/modules` 双面（node 半扫描树组装 boot 清单并提供 `/plugins/<id>/client.js`）；`client/connection` 双端传输（宿主半把网关绑到 `/api`，浏览器半是 fetch/SSE 客户端）；`client/runtime` 共享客户端服务；约 35 个 `ui-*` 包是 React 功能面（完整名单见 `packages/bundle/web-app/cordis.patch.yml:174-274` 与 `packages/client/README.md`）。

### 9.3 sdk/ 与 acp/（进程外驱动与自动化）

`sdk/protocol` 是线协议：换行分隔 JSON-RPC over stdio（`src/transport.ts`）与命名请求/通知（`initialize`、`session/prompt`、`shutdown`；通知 `session.event`、`session.status`、`subagent.*`）。`sdk/server` 是 stdio 上的 Cordis 插件（`src/index.ts:20-46`，事件→通知映射在 `src/server.ts:71-102`）。`sdk/client` 是 TS 客户端：拥有子进程，EOF → SIGTERM → SIGKILL 的销毁梯。

`acp/acp` 是自动化专用的 Agent Client Protocol 服务器（JSON-RPC over stdio，依赖 `@agentclientprotocol/sdk`；`src/index.ts:44-47`）：程序化客户端创建新 agent、提示、收集已提交文本、一次性权限、取消；协议面含 `session/new`、`session/prompt`（每会话一个在途，等空闲）、`session/request_permission`。

### 9.4 apps/（入口）

`apps/cli` 是 `dsh` 启动器：`src/bin.ts` 模式分发、`src/args.ts` 旗标、`src/profile-boot.ts` profile 装配与热重载（见第 3.2 节）、`src/dump-config.ts`。`apps/web` 是前端的 Vite 构建入口，`dist/` 由 host 的 `frontend-static` 提供。三个可运行示例在 `packages/examples/`：`agent-spine-demo`（无执行器无 UI 的 agent 脊柱，单一 bundle 插件）、`acp-demo`、`jsonrpc-demo`。

## 10. 基础设施工件

| 包组 | 内容 |
|---|---|
| `identity/anonymous-user-id` | `$DSH_HOME/.anonymous-user-id` 的随机 UUID，遥测身份（非账户）；库而非插件 |
| `settings/` | `dsh-settings` 命名空间解析接缝 `ctx.settings`（`src/index.ts:132-134`）+ `settings-file` 文件后端（`$DSH_HOME/settings.yaml`，热重载、原子写） |
| `credentials/` | 凭据引用解析接缝 `ctx.credentials` + `credentials-local`（继承环境 > 托管 `.credentials.yaml` > `.env`，逐请求解析，永不物化进进程环境） |
| `storage/` | 非会话存储枢纽 `ctx.storage` + JSON/SQLite 后端 + `storage-domain` 类型化域设施 |
| `workspace/workspace` | 持久工作区：用户目录 + 标题 + 有序会话成员，`ctx.workspaceRegistry` |
| `runtime-diagnostics/invariants` | 运行时断言注册表 `ctx.invariants`（`src/index.ts:69-71`），各包 `/invariant` 伴随插件的落点 |
| `util/` | 零依赖原语：`atomic-write`、`brand`（`Branded<B>`）、`home-paths`（`$DSH_HOME` 解析）、`launch-environment`、`native-command`、`output-retention`、`timeout` |
| `support/` | 测试基建（testkit、不变量、重放、Loader 冒烟），兼容预期更低 |

## 11. 新行为放哪里（速查）

| 目标 | 机制 | 相关代码 |
|---|---|---|
| 加模型提供方 | 在 `ctx.llm` 注册适配器 | `packages/llm/llm/src/index.ts:338`；指南 [cookbook/adding-an-llm-adapter.md](cookbook/adding-an-llm-adapter.md) |
| 加模型可见能力 | 在 `ctx.tools` 注册，schema 自动并入提示装配 | `packages/core/tools/src/index.ts:1037`；指南 [cookbook/adding-a-tool.md](cookbook/adding-a-tool.md) |
| 给一个会话不同能力集 | 组合 agent preset（service 行需 `isolate` realm） | `packages/preset/agent-presets/src/index.ts:70` |
| 加 shell 执行 | 注册 `ctx.shell` 后端 | `packages/shell/shell/src/index.ts:65` |
| 加人类命令 | 在 `ctx.commands` 注册，不经模型轮次分发 | `packages/interaction/commands/src/index.ts:91` |
| 加后台工作 | 在 `ctx.jobs` 注册；`job_*` 工具收集/停止 | `packages/jobs/jobs/src/index.ts:29` |
| 加文件访问或策略 | 注册 `ctx.fs` provider 或监听 `fs/*` 事件 | `packages/fs/fs/src/index.ts:44-77` |
| 限制派生进程 | 用 `ctx.sandbox` 后端；消费者在 spawn 前包装 argv | `packages/sandbox/sandbox/src/index.ts:146` |
| 拦截请求/工具/轮次 | 用对应 `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 停轮 | `packages/core/agent/src/runtime-types.ts:146-292` |
| 加模型可见上下文 | `agent.inject()`，落在下一个被接纳的请求 | `packages/core/agent/src/runtime-types.ts`（`Agent.inject`） |
| 加 UI/编辑器集成 | 驱动 `ctx.agents`，从 `session/event` 渲染 | `packages/core/session/src/index.ts:76` |
| 加 durable 会话状态 | 扩展 `SessionEventMap`，从日志渲染与重放 | `packages/core/session/src/types.ts:236` |
| 分叉活会话 | `ctx.sessions.fork(source, boundary?, childSessionId?)` | `packages/core/session/src/index.ts:1081` |
| 把注册限定到一个 agent | 用那个 agent 的 `agent.ctx` | `packages/core/agent-loop/src/agent.ts:94` |

功能到能力的完整映射与分步指南索引见 [cookbook/extension-cookbook.md](cookbook/extension-cookbook.md)。

## 12. 工程规约与验证速览

常用命令（完整清单见根 [../AGENTS.md](../AGENTS.md) 与 [development.md](development.md)）：`pnpm run test`（vitest 单测）、`test:coverage`（CI 覆盖门，按文件 100%）、`test:e2e`（真实 API，无 `DEEPSEEK_API_KEY` 自跳过）、`test:snapshot`（无键 ACP/无头重放）、`typecheck`、`lint`、`build`、`hygiene`、`doc-sync`（全部文档门）、`pnpm dsh --profile headless "task"`（从源码跑一次任务）。

与"一切皆为插件"直接相关的硬规约（全文在根 [../AGENTS.md](../AGENTS.md)）：

- 注册即效应；注册表的 `register()` 返回处置器。
- 瀑布监听器必须调用 `next()` 委托。
- 判别联合用 tag 切换：封闭联合以 `assertNever` 收尾。
- 跨边界的透明 id 用 `Branded<B>` 品牌化（`packages/util/brand`），不是裸 `string`。
- 包边界显式优于隐式：默认化是 owning 实现里显式的 `resolve(request): Spec` 步骤（模板是 `dsh-shell`）。
- 插件里不硬编码可调参数：随部署变化的选项是可在 cordis.yml 改的已校验 `Config` 字段。
- 错配置响亮失败：能在加载时自足判定的就在加载时，否则在最早可解析点；绝不静默跳过缺失引用。
- 非平凡改动必须在同一 PR 附 Agent Note（`.agents/notes/`）；非平凡的模型或产品用户可见行为变化必须经真实可运行示例加无键快照（策略见 [testing.md](testing.md)）。
- 防御性模式（生命周期/并发/子进程/拆卸）见 [defensive-patterns.md](defensive-patterns.md)。

## 13. 权威文档索引

| 文档 | 内容 |
|---|---|
| [architecture.md](architecture.md) | 权威架构图：组合、核心包、循环、接缝、扩展点 |
| [cordis-primer.md](cordis-primer.md) / [cordis-tutorial/index.md](cordis-tutorial/index.md) | Cordis 概念与上手教程 |
| [subsystems/README.md](subsystems/README.md) | 每个子系统一页：类型定义、语义、生成的 Cordis API |
| [capability-seams.md](capability-seams.md) / [event-producer-consumer.md](event-producer-consumer.md) | 接缝全景图 / 事件生产者消费者全景 |
| [module-graph.md](module-graph.md) | 生成的包依赖图 |
| [tool-catalog.md](tool-catalog.md) / [config-catalog.md](config-catalog.md) / [persistence-catalog.md](persistence-catalog.md) | 生成的工具 / 配置 / 持久化目录 |
| [agent-lifecycle.md](agent-lifecycle.md) / [tool-execution-pipeline.md](tool-execution-pipeline.md) | 时序图与工具管道细节 |
| [cookbook/extension-cookbook.md](cookbook/extension-cookbook.md) | 扩展烹饪书与分步指南索引 |
| [development.md](development.md) / [testing.md](testing.md) / [defensive-patterns.md](defensive-patterns.md) | 贡献者流程 / 测试策略 / 防御性模式 |
| [glossary.md](glossary.md) | 术语表（capability seam 等定义） |
| [../AGENTS.md](../AGENTS.md) / [../packages/README.md](../packages/README.md) | 仓库级规约 / 包分组与发布预期 |
