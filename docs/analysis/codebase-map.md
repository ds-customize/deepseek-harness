# DeepSeek Harness Codebase Map: the Framework and "Everything Is a Plugin"

English | [中文](codebase-map.zh.md)

This is a guided reading map in the style of a code inventory: it maps the framework mechanics of DeepSeek Harness (dsh), the core logic of "everything is a plugin", and each feature area's code to concrete files and line numbers so implementations are quick to locate.

The authoritative document is [architecture.md](architecture.md) (required reading before changing `packages/`); this map complements it for code navigation. Cordis fundamentals are in [cordis-primer.md](cordis-primer.md) and [cordis-tutorial/index.md](cordis-tutorial/index.md).

Line numbers reflect the current `master` and drift with commits; when a reference no longer lands, search for the class or function name given alongside it.

## 1. Project overview

DeepSeek Harness is a plugin-based agent harness on vendored Cordis: the model adapter, the tool registry, the session log, and the agent loop itself are all plugins, so any part is replaceable from configuration. There is no privileged core to patch — the way to extend dsh is to mount your own plugin beside the others.

| Top-level directory | Contents |
|---|---|
| `vendor/` | Vendored Cordis framework source (`cordis`, `cosmokit`, `loader`, `include`, `timer`, and more); sync procedure in [../vendor/README.md](../vendor/README.md) |
| `packages/` | All `@deepseek-ai/dsh-*` workspace packages at `packages/<group>/<pkg>/`; groups in [../packages/README.md](../packages/README.md) |
| `apps/cli/` | The `dsh` CLI launcher: argument parsing, profile composition, mode dispatch (`src/bin.ts`, `src/args.ts`, `src/profile-boot.ts`, `src/dump-config.ts`) |
| `apps/web/` | Web frontend Vite build (`@deepseek-ai/dsh-web-frontend`); its `dist/` is served by the host in `dsh web` |
| `python/` | Python SDK and bundled runtime (see `python/README.md`) |
| `native/` | `@deepseek-ai/node-addon-landlock-run` native module source (see `native/README.md`) |
| `examples/` | Runnable `cordis.yml` leaves (see `examples/AGENTS.md`) |
| `.agents/` | Agent workflows, skills (`.agents/skills/`), and decision records (`.agents/notes/`) |
| `docs/` | Architecture docs and generated catalogs (see [AGENTS.md](AGENTS.md)) |
| `scripts/` | Repo gates and generators (gate list in `scripts/run-gates.ts`) |
| `website/` | VitePress documentation site projecting selected `docs/` pages |

## 2. The foundation: vendored Cordis

Cordis source lives in `vendor/cordis/src/`: `context.ts` (context as a service repository), `service.ts` (the `Service` base class), `fiber.ts` (fiber lifecycle), `events.ts` (event dispatch), `registry.ts`, `reflect.ts`. Five core ideas:

1. **A plugin is an object that implements Service.** It can be a function plugin with optional `inject` and `apply(ctx)` fields, or a `Service` subclass that Cordis mounts into the current context.
2. **A context is a repository of services.** A service claims a stable `ctx.<key>` such as `ctx.tools` or `ctx.llm`; other plugins find services by key instead of importing a concrete implementation.
3. **`inject` declares dependencies.** A plugin that names required services waits until those services exist, so load order is expressed through service requirements rather than manual boot sequencing.
4. **Typed events for communication.** Event names are declared through TypeScript declaration merging; pick the dispatch mode that matches the semantics.
5. **Registrations are reversible effects.** Prompt sections, tool schemas, adapters, providers, and listeners are installed through `ctx.effect()` or `ctx.on()`, so reload and teardown unwind them predictably.

Dispatch modes (full table in [cordis-primer.md](cordis-primer.md)):

| Mode | Awaited | Order | Return value |
|---|---|---|---|
| `emit` | No | observe in registration order | None |
| `waterfall` | No | around middleware in registration order | Yes |
| `parallel` | Yes | all listeners observe in parallel | None |
| `serial` | Yes | registration order | Yes |

`ctx.waterfall` is around-middleware: a listener receives `(...args, next)`, must call `next()` to delegate (returning without it short-circuits); single-decision events use short-circuiting by design. New events document their dispatch mode with `@mode` so the generated catalog can check declarations against dispatch sites.

Loader configuration: a `cordis.yml` allows `!!js` expressions only under plugin `config` and entry `disabled`; other metadata stays literal, and environment-selected plugins use overlays (see the Loader Configuration section of [cordis-primer.md](cordis-primer.md)).

## 3. The core logic of "everything is a plugin"

This section is the key to the whole repository: eight mechanisms, each grounded in code.

### 3.1 No privileged core: the product API spine is plugins too

The services that form the "core" are ordinary plugins, mounted the same way as any third-party extension:

| Capability | Service class | ctx key | Definition |
|---|---|---|---|
| Session log and in-memory store | `SessionStore` | `ctx.sessions` | `packages/core/session/src/index.ts:792` |
| System-prompt assembly | `SystemPrompt` | `ctx.systemPrompt` | `packages/core/system-prompt/src/index.ts:338` |
| Tool registry and execution pipeline | `ToolRuntime` | `ctx.tools` | `packages/core/tools/src/index.ts:787` |
| Agent interface and registry | `AgentRegistry` | `ctx.agents` | `packages/core/agent/src/index.ts:256` |
| Default loop driver | `AgentLoop` | `ctx.agentLoop` | `packages/core/agent-loop/src/index.ts:296` |
| LLM adapter seam | `LlmRuntime` | `ctx.llm` | `packages/llm/llm/src/index.ts:284` |

Each package's `src/index.ts` default-exports that Service subclass for mounting; the `/invariant` subpath is a companion plugin that registers package-owned runtime assertions into `ctx.invariants` (`packages/runtime-diagnostics/invariants/src/index.ts:69`). Lightweight contributions (such as the `name` + `inject` + `apply` at `packages/core/agent-tool-presentation/src/index.ts:28`) are function plugins.

### 3.2 The runtime is a plugin tree composed from configuration: profile / bundle / patch

A running `dsh` is a plugin tree composed at boot from ordered layers; the composition code lives in two places:

- **Profile machinery**: `packages/boot/app-boot/src/profile.ts`. A profile is a `$DSH_HOME/profiles/<name>` directory whose `package.json` `dsh.profile.bundles` field lists the stacked bundles in order (`profile.ts:1-22`); templates `PROFILE_TEMPLATES` (`profile.ts:114-117`) define `web` and `headless`.
- **Boot composition**: `apps/cli/src/profile-boot.ts`. Layer order (`composeProfile`, `profile-boot.ts:142-171`): (1) bundle layers in the profile's listed order (`dsh-base` first); (2) the profile's own `cordis.patch.yml`; (3) the home-level `$DSH_HOME/cordis.patch.yml`; (4) `--patch` overlays in argv order; (5) launcher-injected overlays (the shipped agent-preset root, the `DSH_TELEMETRY_DISABLED` switch).

All layers apply over the empty root `cordis.yml` (`PROFILE_ROOT_CONFIG`, `profile-boot.ts:60-64`) in **one** `applyEntryPatches` call (`profile.ts:413-420`). A patch targets a row by id and replaces its whole config (no deep merge), last write wins; `--dump-config` (`apps/cli/src/dump-config.ts:30-52`) walks the same composition path as real boots, so dumps never drift.

A bundle is a distribution format for Cordis config rows and the code they mount: the `package.json` `dsh.bundle.patch` field points at its patch file (`profile.ts:41-45`).

| Bundle | Location | Role |
|---|---|---|
| `dsh-base` | `packages/bundle/base/cordis.patch.yml` | The first layer of every profile: one `insert` mounting the shared core — runtime rows (`timer`/`hmr`/`llm`/`session`/`typert`, `:16-37`), titles, interaction core (`agent`/`agent-default-model`/`jobs`/`llm-retry`, `:55-72`), settings and credentials (`:78-96`), persistence and query (`:98-127`), telemetry (`:148-161`), sandbox and approval (`:163-205`), all tool rows (`:207-253`), goal/plan (`:256-279`), context management (token-meter/compaction/the subagent family/workflow, `:281-341`), guards and pruning (`:343-394`), web search (`:404-418`), and the overridable "variable rows" (`tools`/`system-prompt`/`agent-loop`/`fs-sandbox`/`llm-deepseek`, `:424-451`) |
| `dsh-web-app` | `packages/bundle/web-app/cordis.patch.yml` | Stacked on base, adds the browser application: host-plane rows (`code-runtime`/`storage`/`api-gateway`/`webserver`/`web-runtime`, `:48-137`), the browser `dsh.client` roster (`:151-274`, about 35 `ui-*` rows), agent-plane tool rows disabled to stay on the host plane (`:293-408`, with inline rationale for which cross-session registries must stay host-side), and `agent-presets` (`:420-424`); the runtime glue plugin is `packages/bundle/web-app/src/index.ts` (name `web-app`, `:26`) |
| `dsh-headless` | `packages/bundle/headless/cordis.patch.yml` | A one-shot runner with no server: `headless-startup` and `headless-runner` (`:24-35`); the runner is `packages/bundle/headless/src/index.ts` (name `headless-runner`, `:25`) |

To see the plugin tree your machine actually boots: `dsh --profile web --dump-config` (flags at `apps/cli/src/args.ts:133-134`, mode dispatch at `apps/cli/src/bin.ts:45-49`). The generated config-field catalog is [config-catalog.md](config-catalog.md).

### 3.3 Registrations are effects

Every contribution goes through `ctx.effect()` / `ctx.on()`; a registration's lifetime is exactly that of the context that registered it. Two levels:

- **Registry storage**: `ScopedLayers` (`packages/core/scope/src/store.ts:159`) holds a global layer plus per-scope overlays; `effect(ctx, action, options)` (`:226-266`) wraps the atomic mutation in a generator `ctx.effect` whose yielded disposer undoes the mutation, deletes an emptied layer, and fires `onChange`. `ctx.tools.register()` (`packages/core/tools/src/index.ts:1057-1061`) and `ctx.systemPrompt.section()` (`packages/core/system-prompt/src/index.ts:385-389`) all route through it.
- **Lifecycle composition**: when teardown order matters, a generator effect yields the exact child disposers. Examples: `SessionStore.create` yields the enter-detach before announcing `session/created` (`packages/core/session/src/index.ts:836-839`); `AgentRegistry.register` is isomorphic (`packages/core/agent/src/index.ts:450-457`); `AgentLoop`'s owner-fused lifecycle (`packages/core/agent-loop/src/index.ts:524-530`); `ToolRuntime.presentAs` composes the layer effect with the prompt-section registrations (`packages/core/tools/src/index.ts:951-971`).

Why `Scope.rawDispose` exists: Cordis dedupes nested effects by function identity, so a composite teardown must yield the exact `fiber.dispose` (`packages/core/scope/src/index.ts:104-112`).

### 3.4 Scopes: every Agent owns its own registration world

`packages/core/scope` provides the per-agent registration primitive:

- **Minting**: `createScope(ctx, key)` (`packages/core/scope/src/index.ts:137`) mounts a shared no-op plugin as a fresh Cordis fiber and tags its context with the `kScope` symbol. An agent's scope is constructed inside the loop (`packages/core/agent-loop/src/agent.ts:94-95`): `this.ctx = this.scope.ctx.extend({ agent: this })` — the context plugins receive via `agent.ctx`.
- **Reading and parent chains**: `scopeOf(ctx)` (`index.ts:154`) walks context inheritance to the nearest tag; `bindScopeParent` (`:72`) builds the parent link once, cycle-checked; `scopeChainOf` (`:98`) returns the nearest-first key chain. The chain drives both directions: registration views inherit **down** (a child sees ancestor layers, nearest shadows), and event admission extends **up** (an ancestor listener receives descendant events). The web preset plane resolves through exactly this `agent → preset → global` chain (see section 8.4).
- **Scope-filtered dispatch**: `scopeTarget(base, key)` (`index.ts:170-185`) builds a routing-only carrier: untagged listeners are admitted globally, a tagged listener is admitted if and only if its tag equals the dispatch key or an ancestor of it. Services dispatch through it (for example `ctx.waterfall(scopeTarget(this, scope), 'system-prompt/assemble', ...)` at `packages/core/system-prompt/src/index.ts:532-535`). `agentEvents` (`packages/core/agent/src/dispatch.ts:107-149`) fuses the agent subject into the carrier so the payload's `agent` and the routing key can never diverge.

The effect: to attach a tool, prompt section, or listener to one agent, write it on that agent's `agent.ctx`; when the agent is destroyed its fiber unloads and every registration unwinds with it.

### 3.5 Typed events are the extension points

Events fall into three domains, and picking the right domain is the first decision in most changes (the full producer/consumer panorama is [event-producer-consumer.md](event-producer-consumer.md)):

| Domain | Carrier | Purpose | Representative members and locations |
|---|---|---|---|
| Session events (durable) | `SessionEventMap` declaration merging; appended to the log and broadcast through `session/event` | The fact must survive a reload | `turn/start`, `step/*`, `user/message`, `assistant/*`, `tool/*` (`packages/core/session/src/types.ts:236-333`); later packages merge in `agent/inbox/spliced`, `compaction/*`, `llm/retry`, `session/title`, `plan/mode`, and more |
| Agent events (live) | `Events` interface declaration merging; carry a live `Agent` | Observe or intercept work in flight | `agent/created`, `agent/status`, `agent/inbox/*` (`packages/core/agent/src/runtime-types.ts:146-292`) |
| Capability events | the seam's own `Events` declarations | Attach policy and adapters to a seam without importing the loop | `tools/pre-execute` and siblings (`packages/core/tools/src/index.ts:142-208`), `fs/write-intent` and siblings (`packages/fs/fs/src/index.ts:49-77`), `llm/stream` (`packages/llm/llm/src/index.ts:64`) |

Conventions: event JSDoc needs `@mode` and payload `@param`s; a `SessionEventMap` member is required-on-read by default — a build that does not know its type refuses the log unless the event carries the envelope's `ignorable: true`; only a structural format change bumps `SESSION_FORMAT_VERSION`. Waterfall listeners must call `next()`.

### 3.6 Model-visible means logged

Anything that reaches a model request must be reconstructable from the session log. The closed loop:

- `Session.append` (`packages/core/session/src/index.ts:604`) does lossless-JSON snapshot validation, rejects reentrancy, builds the frozen event with `seq = log.length`, and pre-validates the surface transition through `SurfaceManager.validateNext` (`packages/core/session/src/surface.ts:398`) before the event enters the log.
- `Session.deriveMessages` (`index.ts:726`) projects model history along the surface nodes (`deriveEventMessage`, `surface.ts:83`) with per-node caching; a compaction replace-generation change rebuilds it.
- The loop re-derives at every model request: `this.session.deriveMessages()` (`packages/core/agent-loop/src/agent.ts:341`).
- The runtime assertion in `packages/core/agent-loop/src/invariant.ts` verifies that LLM requests the loop builds really do reconstruct from the log.

The path for a new model-visible input is therefore fixed: extend `SessionEventMap` and render from the log. Fork, resume, transcripts, telemetry, and persistence all derive from this stream.

### 3.7 Capability seams with three roles

A seam is a swappable capability with three roles: a **Service Definition** declaring the interface and ctx key, a **Service Provider** implementing it, and a **Consumer** using it — commonly a model-facing tool. A package may combine roles, but one role alone is not a seam; adding a capability means designing all three (the seam panorama is [capability-seams.md](capability-seams.md); definitions in [glossary.md](glossary.md)).

The dependency rule (`packages/README.md`): extension plugins depend on Service Definitions, never concrete providers — the structural guarantee that swapping one provider changes the whole product.

The clearest demonstration is the **shared execution world**: the three verbs of `SubprocessRuntime` — `resolveExecutable`/`spawn`/`spawnTerminal` (`packages/subprocess/subprocess/src/index.ts:118-139`) — and `FileSystem` (`packages/fs/fs/src/index.ts:86`) are the only two OS adapters. `bash-local`, `terminal-bash` (PTY via `ctx.subprocess.spawnTerminal`, `packages/terminal/terminal-bash/src/index.ts:110`), and `lsp-stdio` (via `ctx.subprocess` and `ctx.fs`, `packages/lsp/lsp-stdio/src/index.ts:146-157`) never touch the OS directly. Swap `subprocess-local` + `fs-local` for `subprocess-e2b` + `fs-e2b` (`packages/e2b/`) and Bash/PTY/LSP/search move wholesale into the remote sandbox with no provider forks; `fs-sandbox`/`bash-sandbox` fence locally the same way.

### 3.8 The loop itself is swappable

The `Agent` interface (`packages/core/agent/src/runtime-types.ts:64-144`: `id`/`session`/`inbox`/`status`/`ctx`/`cancel`/`whenIdle`/`send`/`steer`/`inject` and more) has zero loop dependency. The only concrete implementation, `ReactLoopAgent`, is package-private to `agent-loop` (`packages/core/agent-loop/src/agent.ts:64`) and is injected into the registry via `ctx.agents.setFactory(this)` (`agent-loop/src/index.ts:350`; registration mechanism at `packages/core/agent/src/index.ts:372-388`); consumers only ever go through `ctx.agents.create/resume` (`:405-430`). Replacing the loop means mounting a plugin that provides a new factory. The "plugins, not loop changes" convention: new behavior lands on documented extension points; genuinely changing `agent-loop` requires updating [architecture.md](architecture.md) in the same change.

## 4. Core package map (packages/core/)

| Package | Owns | ctx key |
|---|---|---|
| `core/scope` | Per-agent scoped-registration primitive | library, no key |
| `core/session` | Append-only `SessionEvent` log and in-memory store; the LLM message history derives from it | `ctx.sessions` |
| `core/agent` | `Agent` interface, live registry, initiator scope, `agent/*` events | `ctx.agents` |
| `core/agent-loop` | The default loop driver: turn/step lifecycle | `ctx.agentLoop` |
| `core/system-prompt` | Prompt-section and tool-schema assembly | `ctx.systemPrompt` |
| `core/tools` | Tool registry and guarded execution pipeline | `ctx.tools` |
| `core/agent-default-model` | Deployment default when an entry point creates an Agent with no session-local model selection | `ctx.agentDefaultModel` |
| `core/agent-tool-presentation` | The row a preset carries: which form of its tools the model sees, `native`/`code`/`both` | none (function plugin) |

### 4.1 core/scope

`src/index.ts` (minting/reading/parent chains/carriers) and `src/store.ts` (`ScopedLayers` layered registry storage whose mutations are effects). Key exports: `createScope` (`index.ts:137`), `scopeOf` (`:154`), `scopeTarget` (`:170`), `bindScopeParent` (`:72`), `scopeChainOf` (`:98`), `ScopedLayers.effect` (`store.ts:226`). `scoped-events.generated.ts` is the generated scoped-event table the `invariant.ts` companion uses to verify every scope-filtered dispatch carries the carrier keyed to the payload's subject.

### 4.2 core/session

| File | Contents |
|---|---|
| `src/index.ts` | `Session` class and `SessionStore` (`:792`); events `session/created`/`disposed`/`event`/`flush` (`:54-85`) |
| `src/types.ts` | `SessionEventMap` (`:236-333`), `SessionEvent`, `SurfaceOp`, `SessionHeader` |
| `src/surface.ts` | Ordered surface projection: `deriveEventMessage` (`:83`), `foldSurface`, `SurfaceManager` (`:398`, including compaction `replace` range splicing and provenance validation) |
| `src/json.ts` | Lossless-JSON validation/snapshotting |
| `src/request-header.ts` and the request-context fold | Canonical folding of request header/context |
| `src/repair.ts` | Crash recovery: synthesizes closers for orphaned turns |
| `src/chunk-rows.ts` | Storage packing of `assistant/chunk` delta runs |
| `src/preparation.ts` | `Disposable` wrapper for an unpublished `Session` |
| `src/invariant.ts` | Monotonic seq, turn/step enclosure, tool call/result pairing assertions |

Main API: `Session.append` (`index.ts:604`), `requestHeader` (`:670`), `requestContext` (`:691`), `deriveMessages` (`:726`), `surface` (`:431`); `SessionStore.create` (`:830`), `prepare` (`:863`), `enter` (`:913`), `announce` (`:968`), `flush` (`:1022`, where persistence plugins drain), `fork` (`:1081`).

### 4.3 core/agent

| File | Contents |
|---|---|
| `src/index.ts` | `AgentRegistry` (`:256`) and the initiator scope (`AsyncLocalStorage`): `currentInitiator` (`:309`), `withInitiator` (`:341`), `setFactory` (`:372`), `create` (`:405`), `resume` (`:424`), `register` (`:450`) |
| `src/runtime-types.ts` | The `Agent` interface (`:64-144`) and `agent/*` events (`:146-292`: emit-class `created`/`status`/`inbox/*`; waterfall-class `pre-step`/`request`/`request-error`; serial `turn-stopping`) |
| `src/dispatch.ts` | The fused dispatcher `agentEvents` (`:107`) and `assembleContextFor` |
| `src/inbox.ts` | `Inbox`: incremental projection of inbox events |
| `src/consumed-work.ts` | Single-pass accounting of consumed vs dropped-unrun work |
| `src/model-selection.ts` | Session-local mutable model selection wired into the `system-prompt/assemble` and `agent/request` waterfalls |

Input reaches the driver through one inbox: some messages wake it immediately; injected context waits in the inbox until another message arrives.

### 4.4 core/agent-loop

| File | Contents |
|---|---|
| `src/index.ts` | `AgentLoop` (`:296`, `inject = ['agents','sessions','llm','tools','systemPrompt']`): the `prepare`/`publish`/`dispose` creation transaction, `FactoryOwnership` teardown bookkeeping, `create` (`:589`), `resume` (`:653`) |
| `src/agent.ts` | `ReactLoopAgent` (`:64`): the Phase state machine (`:38-48`), `wakeDriver` (`:172`), `kick` (`:210`), `turn` (`:246`), `step` (`:332`), `buildRequest` (`:407`) |
| `src/tool-calls.ts` | `executeToolCalls` (`:59`) and `runGroup` (`:121`): a bounded rolling parallel pool, `maxParallelToolCalls` read live from config (default 10, `constants.ts`), results committed in model order, synthetic error results appended for skipped calls on abort |
| `src/runtime-context.ts` | Durable tracking of the last runtime-context snapshot |
| `src/invariant.ts` | The "loop-built requests reconstruct from the log" assertion |

### 4.5 core/system-prompt

`src/index.ts`: `SystemPrompt` (`:338`) holds ordered sections, named context sections, tool schemas, and `{{variable}}` variables; events `system-prompt/assemble` (waterfall, `:31`) and `system-prompt/change` (`:37`). API: `section` (`:381`), `context` (`:398`), `suppressRuntimeContext` (`:415`), `tools` (`:430`), `variable` (`:446`), `assemble` (`:467`); the renderer `renderPrompt` (`:212`). The constructor self-registers the harness identity section and the `deployment:persona` slot (`:357-369`).

### 4.6 core/tools

| File | Contents |
|---|---|
| `src/index.ts` | `ToolRuntime` (`:787`): registration/restriction/guards/Code Mode presentation and the staged execution pipeline |
| `src/schema.ts` | `defineTool` (`:545`) and the parameter-schema vocabulary |
| `src/json-schema.ts` | Value validation for the supported JSON-Schema subset |
| `src/code-mode.ts` | The reserved Code Mode transport tool `run_code` (`RUN_CODE_NAME` `:20`, `createRunCodeTool` `:294`) |
| `src/ts-types.ts`, `src/py-types.ts` | TypeScript/Python SDK renderers for Code Mode |
| `src/presentation.ts` | The `ToolCallView`/`ToolResultView` render-intent vocabulary (`generic`/`terminal`/`diff`/`locations`) |
| `src/types.ts`, `src/invariant.ts` | `tool/code-dispatch*` session-event merges; pipeline monotonicity and frozen-final assertions |

API: `register` (`:1037`), `restrict` (`:1071`), `guard` (`:1110`), `presentAs` (`:946`), `schemas` (`:1234`), `execute` (`:1342`). Events: `tools/pre-execute` (`:152`, the allow/deny/ask gate), `tools/execute` (`:163`, around-dispatch for timeout/retry/metrics), `tools/post-execute` (`:175`, accept/replace/block), `tools/code-dispatch-log` (`:189`), `tools/result` (`:197`, observe-only frozen final outcome), `tools/change` (`:207`). The staged scheduler `TOOL_RUNTIME_SCHEDULER` (`:466`) has its four stages at `:1459`/`:1569`/`:1609`/`:1631`.

### 4.7 core/agent-default-model and core/agent-tool-presentation

`agent-default-model`'s `AgentDefaultModelConfig` (`src/index.ts:64`) stores the deployment default through a settings namespace (the base bundle configures `deepseek-official` / `deepseek-v4-flash`). `agent-tool-presentation` (function plugin, `src/index.ts:28-72`) calls `ctx.tools.presentAs(mode)` per preset row and waits for `codeRuntime` via `ctx.inject(['codeRuntime'], ...)` in code modes.

## 5. Turn flow and the tool pipeline

A **step** is one model request plus the tools it calls; a **turn** is zero or more steps. Event order (bold domains are durable session events; the rest are live extension points across three domains):

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

Implementation locations (all in `packages/core/agent-loop/src/agent.ts` unless noted): the `turn/start` append opens `turn()` (`:246-330`); the `agent/pre-step` proposal at `:266`; `step/start` and `user/message` at `:279-284`; inside `step()` (`:332-401`), `buildRequest` (`:407-495`, running the `agent/request` waterfall and folding `request/header`/`request/context`) → `ctx.llm.stream` appending each `assistant/chunk` (`:345-349`) → failures run the `agent/request-error` waterfall (`:355-365`) → `assistant/message` cites chunk seqs (`:381-390`) → no tool calls completes the step, otherwise `executeToolCalls` (`packages/core/agent-loop/src/tool-calls.ts:59`). Turn closing is data-driven: when a step ends with no next-step input, the loop awaits `agent/turn-stopping` (serial, `:296`), re-reads the inbox, and fresh steering runs another step while none closes the turn.

The five tool-pipeline events and their locations are in section 4.6; detailed semantics in [tool-execution-pipeline.md](tool-execution-pipeline.md) and [agent-lifecycle.md](agent-lifecycle.md). Cancellation and error recovery: [subsystems/core.md](subsystems/core.md).

## 6. Capability seam map

The three roles of each seam at a glance. Paths omit the `packages/` prefix; the "tool" column lists model-visible tool names.

| Group | Service Definition (ctx key) | Providers | Consumers (tools) |
|---|---|---|---|
| `llm/` | `llm/llm`: `ctx.llm` (`llm/llm/src/index.ts:46`), abstract `LlmAdapter` (`:180-233`), `llm/stream` waterfall (`:64`), `StreamChunk` vocabulary (`llm/llm/src/types.ts:291-300`) | `llm/llm-deepseek` (`DeepSeekAdapter`, `adapter.ts:158`, route `deepseek-official`), `llm/llm-pi-ai` (dynamic multi-profile routes), `llm/token-meter` (`ctx.tokenMeter`), `llm/llm-retry` (consumes `agent/request-error`, writes `llm/retry` session events) | No tool; the loop itself is the consumer |
| `shell/` | `shell/shell`: `ctx.shell` (`shell/shell/src/index.ts:40`), abstract `resolve(request): spec` (`:85`) | `shell/bash-local` (runs `bash -c` via `ctx.subprocess`), `bash-sandbox` (`resolve` stamps the `sandboxPolicy` default), `pwsh-local`, `pwsh-sandbox`, `shell-env` (`ctx.shellEnv`, the trusted `DSH_*` registry) | `shell/tool-bash` (tool `bash`), `tool-pwsh` (`pwsh`), `tool-bash-persistent` (persistent `bash` over the PTY seam) |
| `subprocess/` | `subprocess/subprocess`: `ctx.subprocess` (`subprocess/subprocess/src/index.ts:68`), three verbs `resolveExecutable`/`spawn`/`spawnTerminal` (`:118-139`) | `subprocess/subprocess-local` (node-pty), `e2b/subprocess-e2b` | No direct tools; the substrate under bash/PTY/LSP/search |
| `terminal/` | `terminal/terminal`: `ctx.terminals` (`terminal/terminal/src/index.ts:48`) | `terminal/terminal-bash` (PTY delegated to `ctx.subprocess.spawnTerminal`) | `terminal/tool-terminal`: `terminal_open/send/read/signal/close/list` (`tool-terminal/src/index.ts:162-386`) |
| `fs/` | `fs/fs`: `ctx.fs` (`fs/fs/src/index.ts:44`), events `fs/write-intent`/`edit-intent` (waterfalls), `fs/observed` (emit) (`:49-77`) | `fs/fs-local`, `fs/fs-sandbox` (mode fence), `e2b/fs-e2b`, the policy plugin `fs/fs-observation-policy` | `fs/tool-fs`: `read`/`write`/`edit`/`read_image` (`tool-fs/src/*.ts`); `fs/tool-fs-search`: `glob`/`grep` (packaged ripgrep via `ctx.subprocess`); `fs/tool-str-replace-editor` |
| `web/` | `web/web`: `ctx.web` (`web/web/src/index.ts:35`) | `web-search-exa`, `web-search-perplexity`, `web-search-deepseek`, `web-fetch-http` | `web/tool-web`: `web_search`, `web_fetch` |
| `subagent/` | `subagent/subagent`: `ctx.subagents` (`subagent/subagent/src/index.ts:129`), events `subagent/provider-added`/`start`/`end` | `spawn-in-process` (default `spawn`), `fork-in-process`, `subagent-acp`, `subagent-codex`, `subagent-claude-code`, `subagent-dsh-sdk` | `tool-subagent` (default name `subagent`), `tool-subagent-control` (`send_message`/`interrupt_agent`/`list_agents`), `tool-subagent-report` (child-scope `report`) |
| `lsp/` | `lsp/lsp`: `ctx.lsp` (`lsp/lsp/src/index.ts:38`), exactly four operations: definition/references/implementation/hover | `lsp/lsp-stdio` (generic stdio host over the shared execution world) | `lsp/tool-lsp`: tool `lsp` |
| `skill/` | `skill/skill`: `ctx.skills` (`skill/skill/src/index.ts:284`), event `skills/change` | `skill/skill-filesystem`, `skill/skill-badge` | `skill/tool-skill`: tool `skill` (catalog + loader) |
| `sandbox/` | `sandbox/sandbox`: `ctx.sandbox` (`sandbox/sandbox/src/index.ts:146`), `confine(argv, policy)` fail-closed (`:151-175`); `sandbox-policy`: `ctx.sandboxPolicy` (durable per-session mode) | `sandbox-local` (the bwrap/Landlock/Seatbelt/Windows ACL ladder), `sandbox-windows-acl` | No standalone tool; escalation surfaces inside the `bash`/`read`/`write`/`edit` schemas |
| `e2b/` | `e2b/e2b`: `ctx.e2b` (`e2b/e2b/src/index.ts:63`, sandbox lifecycle and shared cwd) | `e2b/fs-e2b` + `e2b/subprocess-e2b` (swap the whole execution world) | Reuses the fs/shell tools |
| `compaction/` | `compaction/compaction`: `ctx.compaction` (`compaction/compaction/src/index.ts:81`), durable events `compaction/start`/`summary`/`end`/`prune` (`types.ts:17-86`) | `compaction/compaction-basic` | `compaction/command-compact`: the human command `/compact`; `compaction-tool-result-pruner`: `ctx.toolResultPruner` |
| `context/` | No single seam | — | `agent-instructions` (AGENTS.md workspace instructions, baseline injection + fs-touch re-injection), `time-context`, `tmux-context` (all via `agent/pre-step`), `session-reference` (`ctx.sessionReferenceResolver`) |
| `jobs/` | `jobs/jobs`: `ctx.jobs` (`jobs/jobs/src/index.ts:29`) | `jobs/jobs-local` | `jobs/tool-jobs`: `job_output`/`job_list`/`job_kill` |
| `workflow/` | `workflow/workflow`: `ctx.workflowEngine` (`workflow/workflow/src/index.ts:31`), events `workflow/start`/`phase`/`log`/`agent-start`/`agent-end`/`end` | `workflow/workflow-worker-thread` | `tool-workflow` (default name `workflow`), `tool-ralph` (fixed policy `ralph`) |
| `spill/` | `spill/spill`: `ctx.spillStore` (`spill/spill/src/index.ts:23`) | `spill/spill-local` | `spill/spill-policy`: the tool-result spill policy listening on `tools/post-execute` |
| `attachment/` | `attachment/attachment`: `ctx.attachments` (`attachment/attachment/src/index.ts:22`) | `attachment/attachment-local` (content-addressed) | No tool; bytes enter via user input/provider output commit |
| `code-runtime/` | `code-runtime/code-runtime`: `ctx.codeRuntime` (`code-runtime/code-runtime/src/index.ts:89`) | `code-runtime-worker-thread` (TypeScript, worker-thread isolation) | `run_code` (the reserved transport tool, `packages/core/tools/src/code-mode.ts:20`, instantiated by `ToolRuntime` under `tools: { mode: 'code' }`) |
| `mcp/` | No seam (pure bridge) | — | `mcp/mcp-client`: connects an external MCP server and registers its tools on `ctx.tools` as `mcp__<server>__<name>` (`mcp-client/src/tools.ts:140-162`) |
| `typert/` | `typert/protocol` declares `ctx.typert` (`typert/protocol/src/types.ts:487`) | `typert/registry` (`TypertRegistry`, `service.ts:446`) | `typert/loader` (discovers loader entries and registers generated host artifacts); `typert/generator` is a build-time library |

Two patterns worth remembering:

- **The shell's explicit resolve step**: `resolve(request): spec` is the seam-owned explicit defaulting step (`packages/shell/shell/src/index.ts:85`; the local implementation fills `workdir`, clamps the timeout, and carries `sandboxPolicy` through, `packages/shell/bash-local/src/index.ts:146-171`), and `run(spec)`/`start(spec)` accept only resolved specs — never a hidden `?? default`. A fencing subclass only overrides `resolve` to stamp its default (`bash-sandbox/src/index.ts:84-85`). This is the template for the "explicit over implicit at package boundaries" convention.
- **The LLM adapter seam**: `LlmRuntime` wraps every call in the `llm/stream` waterfall (`packages/llm/llm/src/index.ts:917-927`) and converts adapter throws into terminal `finish` chunks (`:931-939`); route registration `registerAdapter` (`:338`) returns an atomic `replace` handle, so a settings change swaps routes (DeepSeek provider at `packages/llm/llm-deepseek/src/index.ts:251-266`).

## 7. The session data plane (packages/session/ and packages/session-query/)

The persisted unit is the `SessionEvent` itself; the `SessionHeader` travels separately.

| Package | Role | Key locations |
|---|---|---|
| `session/session-persistence` | The persistence Service Definition: `ctx.sessionPersistence` | `src/index.ts:61-63`, default export `:243` |
| `session/session-persistence-jsonl` | JSONL backend: `<root>/<normalized-cwd>/<encoded-id>/session.jsonl.zstd` (checksummed header frame + append frames) | `src/index.ts:967` |
| `session/session-persistence-sqlite` | One database for all sessions: 1:1 `events` rows (including `assistant/chunk`), WAL, monotonic `SCHEMA_VERSION` | `src/index.ts:414` |
| `session/session-checkpoint-policy` | Zero-config function plugin: checkpoints before each model request, before top-level tool side effects, and at each `agent/pre-step` | `src/index.ts:15` |
| `session/session-projection` | Projection registry `ctx.sessionProjections`: consistent read cuts, per-key registration, state-version collision refusal | `src/index.ts:25-27`, `:428` |
| `session/session-projection-cache` | Projection cache `ctx.sessionProjectionCache` (web config: 200 events / 5s flush) | `src/index.ts:31-33` |
| `session/session-stats` | Whole-log turn/step counts projection | `src/index.ts:18` |
| `session/session-title` and LLM variants | Log-backed titles: every accepted revision is a log-only `session/title` event; `ctx.sessionTitle` | `session-title/src/index.ts:89-91` |
| `session/session-telemetry` + `-otel` | Telemetry backend seam `ctx.sessionTelemetry` + the OTLP/HTTP exporter (controlled by `DSH_TELEMETRY_MODE`, off by default) | `session-telemetry/src/index.ts:20-22`, `session-telemetry-otel/src/index.ts:301` |
| `session-query/session-query` | The query seam `ctx.sessionQuery`: live-preferred merged reads, three-surface event classification (`current`/`shadowed`/`log-only`) | `src/index.ts:69-71` |
| `session-query/session-query-sqlite` | FTS5 full-text search: literal phrases, predicate budgets, generation-bound cursors; `openAt: never` keeps SQLite unopened by default | group README and `src/index.ts:69-72` (launcher owns the path) |
| `session-query/session-log-export` | Browser `/export` command + `GET /api/session.export` ZIP streaming | group README |
| `session-query/tool-session-query` | Model-facing `session_search` and sibling tools: cwd-equality authorization, no cursors exposed | group README |

## 8. Human collaboration, orchestration, and self-modification

### 8.1 interaction/ (the human-collaboration plane)

| Package | Role | ctx key |
|---|---|---|
| `interaction/user-approval` | Channel-neutral one-shot approval seam: outcomes `allowed-once`/`rejected`/`cancelled`/`unavailable`, fail closed; `approval/request` waterfall + the `approval/asked`/`decided` audit pair | `ctx.approval` (`src/index.ts:18-20`) |
| `interaction/permission-presets` | User-facing permission presets (read-only / workspace-write / danger-full-access) bundling `sandbox/mode` with `approval/policy` | `ctx.permissionPresets` (`src/index.ts:37-39`) |
| `interaction/commands` | Plugin-owned human-command registry, dispatched without a model turn; scoped child injection shadows same-name commands | `ctx.commands` (`src/index.ts:91-93`) |
| `interaction/user-questions` | The service a tool/permission plugin uses to pause and ask the human | `ctx.userQuestions` (`src/index.ts:15-17`) |
| `interaction/tool-ask-user` | The model-facing `ask_user_question` tool | consumes `ctx.userQuestions` |

### 8.2 State and feedback

`plan/plan-mode` (logged plan state: the `plan/mode` log-only event, the `/plan` command, the reviewed `exit_plan_mode`; `ctx.planMode`, `src/index.ts:58-60`); `todo/tool-todo` (the whole-list-replace `todo_write` tool + `todo/write` snapshot events); `goal/` (`dsh-goal` same-session goals `ctx.goals`, `goal-round-driver` turning an armed goal into sequential goal rounds, `command-goal`'s `/goal`, `tool-goal`'s model tool); `schedule/schedule` (session-local durable reminders: `after_seconds`/`at`/`every_seconds`, installed only on later-created runtime root agents); `feedback/` (`command-feedback`'s immutable `/remark`; `message-feedback`'s per-assistant-message Like/Dislike + note, stored in a local sidecar, never entering the conversation).

### 8.3 guard/ and hooks/

`guard/repeat-tool-reminder`: an advisory loop breaker — it counts consecutive identical-argument tool runs and injects escalating reminders at configured thresholds (base config `[3,5,8]`), never vetoing. `guard/timeout-policy`: a zero-config single listener arming a cooperative deadline from each tool's declared `timeoutMs` and returning a structured `TOOL_TIMEOUT` on expiry.

`hooks/hook-protocol` is the shared library of the Claude Code / Codex hook wire protocol (matchers, exit-code/stdout codecs, `ctx.shell` execution, most-restrictive merge), not a Cordis plugin; `hooks-claude-code` and `hooks-codex` are the two dialect bridges wiring a user's existing hook configuration onto the harness interception points.

### 8.4 preset/ (per-session agent composition)

`preset/agent-presets`: a preset is a directory with one `agent.cordis.yml`; mounted under a standing scope, each joined session gets its own tools/prompt sections while sessions stay isolated (`ctx.agentPresets`, `src/index.ts:70-72`). The mechanism is exactly the chain from section 3.4: `dsh-tools`/`dsh-system-prompt` file registrations into the calling context's scope layer, and `agent → preset → global` resolves nearest-wins. `preset/persona` is the persona as a composable preset row. The web bundle disables most agent-plane tool rows and supplies them per session from the preset plane instead (`packages/bundle/web-app/cordis.patch.yml:293-424`), keeping cross-session registries (jobs/skill/goal/token-meter/subagent) on the host plane with the rationale inlined in the patch.

### 8.5 extensions/ (the agent modifies its own runtime)

The design record is `.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md`.

| Package | Role |
|---|---|
| `extensions/tool-cordis` | The model-facing self-modification toolset: `cordis_inspect` (inspect services/fibers/tools/dynamic packages and the `api`/`events`/`client` surfaces), `cordis_define` (record + syntax precheck, zero side effects), `cordis_run` (evaluate the host half + deliver the browser half), `cordis_stop`, `cordis_undefine` (`src/index.ts:26-35` plus `src/inspect.ts`, `src/providers.ts`) |
| `extensions/cordis-host-runner` | Dynamic-package host half: `define`/`undefine` only record (minting `dyn-<n>`); `run` evaluates host halves in a `vm` under the `cordis-dynamic` group fiber, and packages with a browser half trigger the answerable `cordis/request-run` round trip (human approval); `ctx.dynamicCordisRunner` (`src/index.ts:81-84`) |
| `extensions/cordis-client-runner` | Browser half: subscribes to announcements, evaluates with a guard facade over the real fiber ctx, seats plugins through `loader.create` |
| `extensions/ui-cordis` | The global overlay panel for run approvals + the durable definition card in the transcript |

### 8.6 boot/ (shared boot glue)

`boot/app-boot/src/index.ts` is the channel-neutral boot library (not a plugin): layered `.env` loading (`:177-198`), the fail-loud Loader guard `installFailLoud` (`:578-649`), the `cordis:include`/`cordis:group` builtins and root-include mount (`:486-529`), live user-patch reconciliation `watchUserPatches` (`:232-265`), post-settle audits (`:658-725`), the boot sequence `boot()` (`:757-802`), and the `harness:source` prompt section (`:805-829`). The profile machinery is in the same package at `src/profile.ts` (see section 3.2). `boot/cmdline` provides the launcher-to-app command-line handoff: `ctx.cmdlineArgs` and `ctx.appExit` (`src/index.ts:45-49`).

## 9. Remote and embedded surfaces

### 9.1 api/ and typert/ (remote BFF and typed RPC)

`api/gateway` is the Typert gateway: the host-side `TypertGatewayService` (`src/index.ts:90`, `inject = ['typert']`) binds dispatch to the transport's `/api` interception (`:104-111`), with endpoint claims from strict generated descriptors and SRC markers scanned off `ctx.reflect.props` (`:114-137`); the client half at `src/client/index.ts` installs the typed Client Remote (`ctx.remote`). `api/remotes` is the host Agent/Session lookup policy + the client-side Remote contribution assembly (commands, goals, dynamic packages, plugin inventory, message feedback); the forwarded-event allowlist `API_REMOTE_FORWARDED_EVENTS` carries a compile-time shape gate (`src/index.ts:33-41`). The `typert/` trio: `generator` (build-time artifact generation from source), `registry` (the runtime registry `ctx.typert`), `loader` (entry discovery and registration).

### 9.2 host/ and client/ (the two halves of the web GUI)

The host half is the API gateway + HTTP route server: `host/webserver` (`ctx.webServer`, `node:http` route registries + the single fallback seat, `src/index.ts:59-127`), `host/apiproxy` (the shared host API gateway, `ctx.apiProxy`), `host/frontend-static` (the SPA dist on the fallback seat), `host/directory-picker*` (the directory-picking seam + native/browse backends + adaptive composition), `host/plugin-inventory` (the read-only Loader-entry projection).

The client half is the browser shell: `client/web`'s `src/boot.tsx` is the kernel (parse `window.__DSH_BOOT__` → build the module system → prefetch → mount the vendored Cordis Loader → full fiber sweep → flip the settled signal); `client/modules` is dual-faced (the node half scans the tree to compose the boot manifest and serves `/plugins/<id>/client.js`); `client/connection` is the two-ended transport (the host half binds the gateway under `/api`; the browser half is the fetch/SSE client); `client/runtime` holds shared client services; roughly 35 `ui-*` packages are the React feature surfaces (the full roster is in `packages/bundle/web-app/cordis.patch.yml:174-274` and `packages/client/README.md`).

### 9.3 sdk/ and acp/ (out-of-process driving and automation)

`sdk/protocol` is the wire protocol: newline-delimited JSON-RPC over stdio (`src/transport.ts`) and the named requests/notifications (`initialize`, `session/prompt`, `shutdown`; notifications `session.event`, `session.status`, `subagent.*`). `sdk/server` is the Cordis plugin on stdio (`src/index.ts:20-46`; the event→notification mapping at `src/server.ts:71-102`). `sdk/client` is the TypeScript client: it owns the child process with an EOF → SIGTERM → SIGKILL disposal ladder.

`acp/acp` is the automation-only Agent Client Protocol server (JSON-RPC over stdio, depending on `@agentclientprotocol/sdk`; `src/index.ts:44-47`): programmatic clients create fresh agents, prompt, collect committed text, grant one-shot permissions, and cancel; the protocol surface includes `session/new`, `session/prompt` (one in flight per session, waiting for idle), and `session/request_permission`.

### 9.4 apps/ (entry points)

`apps/cli` is the `dsh` launcher: mode dispatch in `src/bin.ts`, flags in `src/args.ts`, profile composition and live reload in `src/profile-boot.ts` (see section 3.2), dumps in `src/dump-config.ts`. `apps/web` is the frontend Vite build entry; its `dist/` is served by the host's `frontend-static`. The three runnable examples live in `packages/examples/`: `agent-spine-demo` (the executor-less, UI-less agent spine as a single bundle plugin), `acp-demo`, and `jsonrpc-demo`.

## 10. Infrastructure workpieces

| Group | Contents |
|---|---|
| `identity/anonymous-user-id` | The random UUID at `$DSH_HOME/.anonymous-user-id` used as the telemetry identity (not an account); a library, not a plugin |
| `settings/` | `dsh-settings`, the namespace-resolution seam `ctx.settings` (`src/index.ts:132-134`) + `settings-file`, the file-backed provider (`$DSH_HOME/settings.yaml`, hot-reloaded, atomic writes) |
| `credentials/` | The credential-reference seam `ctx.credentials` + `credentials-local` (inherited env over the managed `.credentials.yaml` over `.env`, resolved per request, never materialized into process env) |
| `storage/` | The non-session storage hub `ctx.storage` + JSON/SQLite backends + the typed-domain facility `storage-domain` |
| `workspace/workspace` | Persistent workspaces: user directories + titles + ordered session membership, `ctx.workspaceRegistry` |
| `runtime-diagnostics/invariants` | The runtime-assertion registry `ctx.invariants` (`src/index.ts:69-71`) where each package's `/invariant` companion lands |
| `util/` | Zero-dependency primitives: `atomic-write`, `brand` (`Branded<B>`), `home-paths` (`$DSH_HOME` resolution), `launch-environment`, `native-command`, `output-retention`, `timeout` |
| `support/` | Test infrastructure (testkits, invariants, replay, Loader smokes) with lower compatibility expectations |

## 11. Where new behavior goes (quick reference)

| Goal | Mechanism | Related code |
|---|---|---|
| Add a model provider | register its adapter on `ctx.llm` | `packages/llm/llm/src/index.ts:338`; guide [cookbook/adding-an-llm-adapter.md](cookbook/adding-an-llm-adapter.md) |
| Add a model-facing capability | register on `ctx.tools`; its schema joins prompt assembly | `packages/core/tools/src/index.ts:1037`; guide [cookbook/adding-a-tool.md](cookbook/adding-a-tool.md) |
| Give one session a different capability set | compose an agent preset (a service row there needs an `isolate` realm) | `packages/preset/agent-presets/src/index.ts:70` |
| Add shell execution | register a `ctx.shell` backend | `packages/shell/shell/src/index.ts:65` |
| Add a human command | register on `ctx.commands`; it dispatches without a model turn | `packages/interaction/commands/src/index.ts:91` |
| Add background work | register on `ctx.jobs`; the `job_*` tools collect or stop it | `packages/jobs/jobs/src/index.ts:29` |
| Add filesystem access or policy | register a `ctx.fs` provider or listen to `fs/*` events | `packages/fs/fs/src/index.ts:44-77` |
| Confine spawned processes | use a `ctx.sandbox` backend; consumers wrap argv before spawning | `packages/sandbox/sandbox/src/index.ts:146` |
| Intercept a request, tool, or turn | use its `agent/*` or `tools/*` event; `agent/turn-stopping` stops a turn | `packages/core/agent/src/runtime-types.ts:146-292` |
| Add model-facing context | `agent.inject()`; it lands in the next admitted request | `packages/core/agent/src/runtime-types.ts` (`Agent.inject`) |
| Add UI or editor integration | drive `ctx.agents` and render from `session/event` | `packages/core/session/src/index.ts:76` |
| Add durable session state | extend `SessionEventMap`; render and replay from the log | `packages/core/session/src/types.ts:236` |
| Fork a live session | `ctx.sessions.fork(source, boundary?, childSessionId?)` | `packages/core/session/src/index.ts:1081` |
| Scope a registration to one agent | use that agent's `agent.ctx` | `packages/core/agent-loop/src/agent.ts:94` |

The complete feature-to-capability mapping and guide index is [cookbook/extension-cookbook.md](cookbook/extension-cookbook.md).

## 12. Engineering conventions and verification at a glance

Common commands (the full list is in the root [../AGENTS.md](../AGENTS.md) and [development.md](development.md)): `pnpm run test` (vitest unit tests), `test:coverage` (the CI coverage gate, 100% per file), `test:e2e` (real API, self-skips without `DEEPSEEK_API_KEY`), `test:snapshot` (keyless ACP/headless replay), `typecheck`, `lint`, `build`, `hygiene`, `doc-sync` (all documentation gates), and `pnpm dsh --profile headless "task"` (run one task from source).

The hard conventions that follow directly from "everything is a plugin" (full text in the root [../AGENTS.md](../AGENTS.md)):

- Registrations are effects; a registry's `register()` returns the disposer.
- Waterfall listeners must call `next()` to delegate.
- Switch on discriminant tags: closed unions end in `assertNever`.
- Opaque cross-boundary ids are branded with `Branded<B>` (`packages/util/brand`), never bare `string`.
- Explicit over implicit at package boundaries: defaulting is an explicit `resolve(request): Spec` step in the owning implementation (the template is `dsh-shell`).
- No hardcoded tunables in plugins: deployment-varying choices are validated `Config` fields changeable from cordis.yml.
- Misconfiguration fails loud: self-contained at load, otherwise at the earliest resolvable point; never silently skip a missing referent.
- Non-trivial changes require an Agent Note in the same PR (`.agents/notes/`); non-trivial model- or product-user-visible behavior changes add a keyless snapshot through a real runnable example in the same PR (policy in [testing.md](testing.md)).
- Defensive patterns for lifecycle/concurrency/subprocess/teardown are in [defensive-patterns.md](defensive-patterns.md).

## 13. Authoritative documentation index

| Document | Contents |
|---|---|
| [architecture.md](architecture.md) | The authoritative architecture map: composition, core packages, loop, seams, extension points |
| [cordis-primer.md](cordis-primer.md) / [cordis-tutorial/index.md](cordis-tutorial/index.md) | Cordis concepts and the hands-on tutorial |
| [subsystems/README.md](subsystems/README.md) | One page per subsystem: type definitions, semantics, the generated Cordis API |
| [capability-seams.md](capability-seams.md) / [event-producer-consumer.md](event-producer-consumer.md) | The seam panorama / the event producer-consumer panorama |
| [module-graph.md](module-graph.md) | The generated package dependency graph |
| [tool-catalog.md](tool-catalog.md) / [config-catalog.md](config-catalog.md) / [persistence-catalog.md](persistence-catalog.md) | Generated tool / config / persistence catalogs |
| [agent-lifecycle.md](agent-lifecycle.md) / [tool-execution-pipeline.md](tool-execution-pipeline.md) | Sequence diagrams and tool-pipeline detail |
| [cookbook/extension-cookbook.md](cookbook/extension-cookbook.md) | The extension cookbook and step-by-step guide index |
| [development.md](development.md) / [testing.md](testing.md) / [defensive-patterns.md](defensive-patterns.md) | Contributor workflow / testing policy / defensive patterns |
| [glossary.md](glossary.md) | Glossary (capability seam and other definitions) |
| [../AGENTS.md](../AGENTS.md) / [../packages/README.md](../packages/README.md) | Repo-level conventions / package groups and release expectations |
