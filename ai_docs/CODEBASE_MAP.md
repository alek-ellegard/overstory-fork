---
last_mapped: 2026-03-01T12:00:00Z
total_files: 244
total_tokens: 776014
---

# Codebase Map

## System Overview

Overstory is a multi-agent orchestration system for AI coding agents. It spawns workers in isolated git worktrees via tmux, coordinates them through a custom SQLite mail system, and merges their work back with 4-tier conflict resolution. A pluggable `AgentRuntime` interface supports Claude Code, Pi, GitHub Copilot, and OpenAI Codex.

```
                        ┌─────────────────────────────────────┐
                        │     Orchestrator (Claude Code)      │
                        │  Your session IS the orchestrator   │
                        │  CLAUDE.md + hooks + `ov` CLI       │
                        └────────┬──────────┬─────────────────┘
                                 │          │
                    ┌────────────┘          └────────────┐
                    ▼                                    ▼
         ┌──────────────────┐                 ┌──────────────────┐
         │   Coordinator    │                 │   Watchdog Tiers │
         │  (persistent,    │                 │  T0: daemon.ts   │
         │   root, no wt)   │                 │  T1: triage.ts   │
         │  Dispatches via  │                 │  T2: monitor.md  │
         │  ov sling        │                 └──────────────────┘
         └───┬─────┬────┬───┘
             │     │    │
     ┌───────┘     │    └───────┐
     ▼             ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Lead   │  │  Lead   │  │  Lead   │    (depth 1, own worktree)
│ (opus)  │  │ (opus)  │  │ (opus)  │
└──┬──┬───┘  └─────────┘  └─────────┘
   │  │
   │  └──────────┐
   ▼              ▼
┌────────┐  ┌──────────┐
│ Scout  │  │ Builder  │    (depth 2, own worktree)
│(haiku) │  │ (sonnet) │
└────────┘  └──────────┘

Data stores (all SQLite WAL):
  mail.db ──── Inter-agent messaging
  sessions.db ── Agent lifecycle + runs
  events.db ──── Tool events + timelines
  metrics.db ─── Token usage + costs
  merge-queue.db ─ FIFO merge queue
```

### Key Architectural Patterns

- **Two-layer agent definitions**: Base `.md` (HOW) + per-task overlay CLAUDE.md (WHAT)
- **Runtime-agnostic**: `AgentRuntime` interface abstracts spawn, config, guards, transcripts
- **SQLite everywhere**: WAL mode, `bun:sqlite` synchronous API, ~1-5ms per query
- **No YAML library**: Hand-rolled parsers in `config.ts` and `identity.ts`
- **DI over mocks**: `CoordinatorDeps`, `StopDeps`, `ConsistencyCheckDeps` avoid `mock.module()` leakage
- **Fire-and-forget hooks**: `ov log` never aborts Claude Code tool execution
- **Pending-nudge pattern**: File markers instead of direct tmux sendKeys to avoid I/O corruption

## Directory Structure

```
overstory/
├── agents/                          # Canonical published agent definitions (npm package)
│   ├── builder.md                   #   Implementation agent (sonnet, read/write)
│   ├── coordinator.md               #   Top-level orchestrator (opus, read-only, spawns leads)
│   ├── lead.md                      #   Team lead (opus, read/write, spawns workers)
│   ├── merger.md                    #   Branch merge specialist (sonnet, read/write)
│   ├── monitor.md                   #   Tier 2 fleet patrol (sonnet, read-only)
│   ├── reviewer.md                  #   Validation agent (sonnet, read-only)
│   ├── scout.md                     #   Read-only reconnaissance (haiku)
│   └── supervisor.md                #   DEPRECATED: use lead instead
├── docs/
│   ├── runtime-abstraction.md       #   Design spec for multi-runtime adapter system
│   └── runtime-adapters.md          #   Contributor guide for new runtime adapters
├── scripts/
│   └── version-bump.ts              #   Bumps semver in package.json + src/index.ts
├── templates/
│   ├── CLAUDE.md.tmpl               #   Orchestrator CLAUDE.md template
│   ├── hooks.json.tmpl              #   Agent hooks template
│   └── overlay.md.tmpl              #   Per-worker overlay template
├── src/
│   ├── index.ts                     #   CLI entry point (Commander.js, 32 commands)
│   ├── types.ts                     #   ALL shared types and interfaces
│   ├── errors.ts                    #   Typed error hierarchy (extends OverstoryError)
│   ├── config.ts                    #   YAML config loader + hand-rolled parser
│   ├── json.ts                      #   JSON envelope helpers (jsonOutput/jsonError)
│   ├── test-helpers.ts              #   Real git repos, temp dirs, commit helpers
│   ├── agents/                      #   Agent lifecycle management
│   │   ├── manifest.ts              #     Agent registry loader + model resolution
│   │   ├── overlay.ts               #     Dynamic CLAUDE.md overlay generator
│   │   ├── hooks-deployer.ts        #     Guards: path boundary, capability, bash patterns
│   │   ├── guard-rules.ts           #     Pure data: tool lists, bash pattern constants
│   │   ├── identity.ts              #     Persistent agent CV (YAML in agents/{name}/)
│   │   ├── checkpoint.ts            #     Session checkpoint save/load/clear
│   │   └── lifecycle.ts             #     Handoff protocol (initiate/resume/complete)
│   ├── beads/                       #   Beads (bd) CLI wrapper
│   │   ├── client.ts                #     Typed wrapper for bd subcommands
│   │   └── molecules.ts             #     Multi-step workflow templates (planned API)
│   ├── commands/                    #   One file per CLI subcommand (32 commands)
│   │   ├── agents.ts                #     ov agents — discover active agents
│   │   ├── clean.ts                 #     ov clean — nuclear runtime cleanup
│   │   ├── completions.ts           #     ov completions — shell completion scripts
│   │   ├── coordinator.ts           #     ov coordinator start/stop/status
│   │   ├── costs.ts                 #     ov costs — token/cost analysis
│   │   ├── dashboard.ts             #     ov dashboard — live TUI (ANSI, multi-panel)
│   │   ├── doctor.ts                #     ov doctor — 11-category health checks
│   │   ├── ecosystem.ts             #     ov ecosystem — os-eco tool versions
│   │   ├── errors.ts                #     ov errors — aggregated error view
│   │   ├── feed.ts                  #     ov feed — unified real-time event stream
│   │   ├── group.ts                 #     ov group — batch task coordination
│   │   ├── hooks.ts                 #     ov hooks install/uninstall/status
│   │   ├── init.ts                  #     ov init — scaffold .overstory/ + bootstrap
│   │   ├── inspect.ts               #     ov inspect — deep single-agent view
│   │   ├── log.ts                   #     ov log — hook target (NDJSON + events.db)
│   │   ├── logs.ts                  #     ov logs — query raw NDJSON log files
│   │   ├── mail.ts                  #     ov mail send/check/list/read/reply/purge
│   │   ├── merge.ts                 #     ov merge — branch merging via queue + resolver
│   │   ├── metrics.ts               #     ov metrics — session duration/capability stats
│   │   ├── monitor.ts               #     ov monitor start/stop/status (Tier 2)
│   │   ├── nudge.ts                 #     ov nudge — tmux text nudge with debounce
│   │   ├── prime.ts                 #     ov prime — context injection for sessions
│   │   ├── replay.ts                #     ov replay — interleaved multi-agent replay
│   │   ├── run.ts                   #     ov run list/show/complete
│   │   ├── sling.ts                 #     ov sling — 14-step agent spawn pipeline
│   │   ├── spec.ts                  #     ov spec write — task specification files
│   │   ├── status.ts                #     ov status — fleet-wide state aggregation
│   │   ├── stop.ts                  #     ov stop — agent termination
│   │   ├── supervisor.ts            #     ov supervisor (DEPRECATED)
│   │   ├── trace.ts                 #     ov trace — chronological event timeline
│   │   ├── upgrade.ts               #     ov upgrade — npm version management
│   │   ├── watch.ts                 #     ov watch — Tier 0 watchdog daemon
│   │   └── worktree.ts              #     ov worktree list/clean
│   ├── doctor/                      #   11 modular health check categories
│   │   ├── types.ts                 #     DoctorCheck, DoctorCheckFn, DoctorCategory
│   │   ├── agents.ts                #     Manifest + identity validation
│   │   ├── config-check.ts          #     Config parseability + schema validation
│   │   ├── consistency.ts           #     Cross-subsystem state reconciliation
│   │   ├── databases.ts             #     SQLite schema + WAL mode validation
│   │   ├── dependencies.ts          #     External tool availability (git, tmux, bd/sd)
│   │   ├── ecosystem.ts             #     os-eco tool version validation
│   │   ├── logs.ts                  #     Log dir health, NDJSON validity, orphans
│   │   ├── merge-queue.ts           #     Merge queue schema + stale entry detection
│   │   ├── providers.ts             #     Provider config + gateway reachability
│   │   ├── structure.ts             #     .overstory/ directory tree validation
│   │   └── version.ts               #     Version sync (package.json vs index.ts)
│   ├── e2e/
│   │   └── init-sling-lifecycle.test.ts  # Full init→config→manifest→overlay pipeline
│   ├── events/                      #   Tool event tracking
│   │   ├── store.ts                 #     SQLite EventStore (insert, correlate, query)
│   │   └── tool-filter.ts           #     Smart arg filtering (20KB → 200 bytes)
│   ├── insights/
│   │   └── analyzer.ts              #     Post-session insight generation for mulch
│   ├── logging/                     #   Visual output layer
│   │   ├── color.ts                 #     ANSI primitives, quiet mode, chalk re-export
│   │   ├── theme.ts                 #     State colors, icons, separators, headers
│   │   ├── format.ts                #     Duration, timestamps, event lines, color maps
│   │   ├── logger.ts                #     Multi-format NDJSON + human logger
│   │   ├── reporter.ts              #     LogEvent → colored terminal line
│   │   ├── sanitizer.ts             #     Regex secret redaction (API keys, tokens)
│   │   └── theme.ts                 #     Canonical visual theme constants
│   ├── mail/                        #   Inter-agent messaging
│   │   ├── store.ts                 #     SQLite MailStore (CRUD, WAL, migrations)
│   │   ├── client.ts                #     Business logic (check, inject, reply routing)
│   │   └── broadcast.ts             #     @group address resolution (@all, @builders)
│   ├── merge/                       #   Branch integration
│   │   ├── queue.ts                 #     SQLite FIFO merge queue
│   │   └── resolver.ts              #     4-tier conflict resolution engine
│   ├── metrics/                     #   Token usage + cost tracking
│   │   ├── pricing.ts               #     Model pricing table (Claude, GPT, Gemini)
│   │   ├── transcript.ts            #     Claude JSONL transcript parser
│   │   ├── store.ts                 #     SQLite MetricsStore (sessions + snapshots)
│   │   └── summary.ts               #     Aggregation + console formatting
│   ├── mulch/
│   │   └── client.ts                #     Hybrid wrapper: programmatic API + CLI spawn
│   ├── runtimes/                    #   Pluggable agent runtime adapters
│   │   ├── types.ts                 #     AgentRuntime interface + supporting types
│   │   ├── registry.ts              #     Factory: getRuntime(name) → adapter
│   │   ├── claude.ts                #     Claude Code adapter (hooks-based guards)
│   │   ├── pi.ts                    #     Pi adapter (extension-based guards)
│   │   ├── pi-guards.ts             #     Pi guard extension code generator
│   │   ├── copilot.ts               #     GitHub Copilot adapter (--allow-all-tools)
│   │   └── codex.ts                 #     OpenAI Codex adapter (OS-level sandbox)
│   ├── sessions/                    #   Agent session lifecycle
│   │   ├── store.ts                 #     SQLite SessionStore + RunStore
│   │   └── compat.ts                #     Migration bridge (sessions.json → sessions.db)
│   ├── tracker/                     #   Pluggable task tracker
│   │   ├── types.ts                 #     TrackerClient interface (self-contained)
│   │   ├── factory.ts               #     Auto-detect backend (beads vs seeds)
│   │   ├── beads.ts                 #     Beads (bd) adapter
│   │   └── seeds.ts                 #     Seeds (sd) adapter
│   ├── watchdog/                    #   Tiered health monitoring
│   │   ├── health.ts                #     ZFC evaluation + state machine
│   │   ├── daemon.ts                #     Tier 0 polling daemon + escalation
│   │   └── triage.ts                #     Tier 1 AI-assisted log classification
│   └── worktree/                    #   Git worktree + tmux management
│       ├── manager.ts               #     Git worktree CRUD + branch management
│       └── tmux.ts                  #     Tmux session lifecycle + process tree cleanup
├── .overstory/                      #   Self-hosted orchestration state
│   ├── config.yaml                  #     Project config (25 agents max, 2s stagger)
│   ├── agent-manifest.json          #     7 agent types with models + capabilities
│   ├── agent-defs/                  #     Live agent definitions (template placeholders)
│   ├── hooks.json                   #     Orchestrator hook config
│   └── groups.json                  #     40+ historical task groups
├── .claude/                         #   Claude Code settings + slash commands
│   ├── settings.json                #     Experimental agent teams enabled
│   └── commands/                    #     issue-reviews, pr-reviews, prioritize, release
├── .mulch/                          #   Structured expertise (6 domains, 100 entry cap)
├── .seeds/                          #   Seeds issue tracker config
├── .pi/                             #   Pi runtime extensions (guards, mulch, status)
└── .github/                         #   CI/CD, templates, dependabot
    └── workflows/
        ├── ci.yml                   #     lint → typecheck → test
        └── publish.yml              #     Quality gates → npm publish → git tag → release
```

## Module Guide

### src/ (Core)

**Purpose**: CLI entry point, shared types, configuration, error taxonomy, JSON conventions, test infrastructure.

| File | Key Exports | Role |
|------|-------------|------|
| `index.ts` | `VERSION` | Commander.js program with 32 subcommands, fuzzy suggestions |
| `types.ts` | ~50 interfaces/types | Single source of truth for all TypeScript types |
| `errors.ts` | `OverstoryError` + 9 subclasses | Typed error hierarchy with machine-readable `code` |
| `config.ts` | `loadConfig()`, `resolveProjectRoot()` | Hand-rolled YAML parser, deep merge, migration |
| `json.ts` | `jsonOutput()`, `jsonError()` | Ecosystem JSON envelope convention |
| `test-helpers.ts` | `createTempGitRepo()`, `commitFile()` | Real git repos in temp dirs for testing |

### src/agents/

**Purpose**: Agent lifecycle — manifest loading, overlay generation, guard deployment, identity, checkpoints, handoffs.

| File | Key Exports | Role |
|------|-------------|------|
| `manifest.ts` | `createManifestLoader()`, `resolveModel()` | Agent registry, model alias resolution |
| `overlay.ts` | `generateOverlay()`, `writeOverlay()` | Template substitution for per-task CLAUDE.md |
| `hooks-deployer.ts` | `deployHooks()` | Generates `.claude/settings.local.json` with guards |
| `guard-rules.ts` | `WRITE_TOOLS`, `DANGEROUS_BASH_PATTERNS` | Pure data constants for guard generation |
| `identity.ts` | `createIdentity()`, `loadIdentity()` | Persistent agent CV as YAML |
| `checkpoint.ts` | `saveCheckpoint()`, `loadCheckpoint()` | JSON checkpoint for compaction recovery |
| `lifecycle.ts` | `initiateHandoff()`, `resumeFromHandoff()` | Session handoff protocol |

### src/commands/ (32 commands)

**Purpose**: One file per CLI subcommand. Each exports a Commander.js factory + programmatic entry point.

**Critical path commands**:
- `sling.ts`: 14-step agent spawn pipeline (the most complex command)
- `coordinator.ts`: Persistent orchestrator lifecycle with DI for testability
- `log.ts`: Hook target — writes NDJSON + events.db, fire-and-forget
- `mail.ts`: Inter-agent messaging with pending-nudge file pattern
- `merge.ts`: Branch merging via FIFO queue + 4-tier resolver
- `dashboard.ts`: Live TUI with multi-panel ANSI rendering

**Observability commands**: `status`, `inspect`, `trace`, `errors`, `feed`, `replay`, `logs`, `costs`, `metrics`

**Infrastructure commands**: `init`, `hooks`, `doctor`, `clean`, `watch`, `monitor`, `worktree`, `upgrade`, `ecosystem`, `completions`

### src/mail/

**Purpose**: Three-layer messaging — `store.ts` (SQLite CRUD), `client.ts` (business logic), `broadcast.ts` (group resolution).

- Messages sorted DESC for listing, ASC for unread inbox (FIFO processing)
- `check()`/`checkInject()` are destructive reads (mark as read)
- Schema migration runs on every `createMailStore()` call
- Payload stored as raw JSON TEXT, no runtime validation on parse
- Group addresses: `@all`, `@builders`, `@scouts`, etc. — case-sensitive capability matching

### src/merge/

**Purpose**: FIFO merge queue + 4-tier conflict resolution.

- **Tier 1**: Clean `git merge --no-edit`
- **Tier 2**: Auto-resolve (keep-incoming or union strategy per file)
- **Tier 3**: AI-resolve via `runtime.buildPrintCommand()` (runtime-agnostic)
- **Tier 4**: Reimagine — `git merge --abort`, regenerate from both branch snapshots
- `dequeue()` physically deletes rows (crash = lost entry)
- Mulch-based historical learning for tier skipping (2-failure threshold)

### src/runtimes/

**Purpose**: Pluggable `AgentRuntime` interface with 4 adapters.

| Runtime | CLI | Guard Mechanism | Print Command | Beacon Verification |
|---------|-----|-----------------|---------------|---------------------|
| Claude Code | `claude` | `.claude/settings.local.json` hooks | `-p` flag | Yes (TUI swallows Enter) |
| Pi | `pi` | `.pi/extensions/` guard extension | Positional arg | No (indistinguishable states) |
| Copilot | `copilot` | `--allow-all-tools` (no guards) | N/A | Yes |
| Codex | `codex exec` | OS-level sandbox (Seatbelt/Landlock) | `--ephemeral` | No (headless) |

### src/watchdog/

**Purpose**: Three-tier health monitoring with progressive escalation.

- **health.ts**: ZFC evaluation — signal priority: tmux liveness > pid > recorded state. Forward-only transitions.
- **daemon.ts**: Tier 0 polling. Escalation: warn → nudge → AI triage → terminate. Full DI for testing.
- **triage.ts**: Tier 1 AI classification. Keyword-based: `retry` → recoverable, `terminate` → fatal, default → extend.

### src/worktree/

**Purpose**: Git worktree CRUD + tmux session lifecycle.

- `createWorktree()`: Branch convention `overstory/{agentName}/{taskId}`
- `killSession()`: Three-phase cleanup: get PID → SIGTERM tree → grace → SIGKILL → tmux kill-session
- `waitForTuiReady()`: Runtime-agnostic polling via `detectReady` callback
- `sendKeys()`: Flattens newlines to spaces (prevents premature Enter in TUI)

### src/doctor/

**Purpose**: 11 modular health check categories, each exporting `DoctorCheckFn`.

Categories: `agents`, `config`, `consistency`, `databases`, `dependencies`, `ecosystem`, `logs`, `merge-queue`, `providers`, `structure`, `version`. Each returns `DoctorCheck[]` with optional `fix()` closures.

### src/metrics/

**Purpose**: Token usage tracking, cost estimation, transcript parsing.

- `pricing.ts`: Model pricing table (Claude, GPT, Gemini) with substring matching
- `transcript.ts`: Claude JSONL parser capturing model from first assistant turn only
- `store.ts`: SQLite with `INSERT OR REPLACE` on `(agent_name, task_id)`
- `summary.ts`: In-memory aggregation (up to 10,000 sessions)

### src/tracker/

**Purpose**: Pluggable task tracker with auto-detection.

- `resolveBackend("auto")` checks `.seeds/` before `.beads/`, defaults to seeds
- Beads uses `issue_type` (normalized to `type`), Seeds uses JSON envelope format
- `types.ts` is self-contained (does not import from `src/types.ts`)

## Key Data Flows

### Agent Spawn (ov sling)

```
CLI args → loadConfig → validateHierarchy → manifestLoader.load()
  → resolveModel(config, manifest, role)
  → checkRunSessionLimit / checkParentAgentLimit / checkTaskLock
  → createWorktree(repoRoot, agentName, taskId)
  → generateOverlay(overlayConfig) → writeOverlay(worktreePath)
  → runtime.deployConfig(worktreePath, overlay, hooks)
  → mail.send(dispatch message, pre-send before tmux)
  → tracker.claim(taskId)
  → createIdentity(agentsDir, identity)
  → tmux.createSession(name, cwd, spawnCommand, env)
  → waitForTuiReady(name, runtime.detectReady)
  → sendKeys(name, beacon) → verify beacon delivery (5 retries)
  → sessionStore.upsert(session)
  → runStore.incrementAgentCount(runId)
```

### Hook Event Pipeline (ov log)

```
Claude Code hook fires → ov log <event> --agent <name> --stdin
  → Parse JSON from stdin (tool_name, tool_input, session_id)
  → filterToolArgs(toolName, toolInput) → compact representation
  → createLogger → write NDJSON to .overstory/logs/
  → createEventStore → insert to events.db
  → On tool-end: throttled token snapshot to metrics.db (30s interval)
  → On session-end:
      → transitionToCompleted (sessions.db)
      → updateIdentity (agent CV)
      → autoRecordExpertise (mulch)
      → send coordinator nudge (if lead)
```

### Mail Check (UserPromptSubmit hook)

```
ov mail check --inject --agent <name>
  → resolveProjectRoot() (worktree-aware)
  → readAndClearPendingNudge(agent) → file-based markers
  → client.checkInject(agent) → unread messages (marks read)
  → Format with priority banner if nudge was pending
  → Debounce tracking via mail-check-state.json
  → Output injected into agent prompt context
```

### Merge Pipeline

```
ov merge --branch <name>
  → Resolve target: --into > session-branch.txt > canonicalBranch
  → createMergeQueue → enqueue if not present
  → dequeue (FIFO) → createMergeResolver
  → Tier 1: git merge --no-edit
    → success: return clean-merge
    → fail: collect conflictFiles
  → [optional] mulch search for conflict history → skipTiers
  → Tier 2: auto-resolve (keep-incoming / union)
  → Tier 3: AI-resolve via runtime.buildPrintCommand
  → Tier 4: git merge --abort → reimagine from git show
  → recordConflictPattern to mulch (fire-and-forget)
```

## Conventions

### Code Style
- **Tab indentation**, 100-char line width (Biome enforced)
- **Strict TypeScript**: `noUncheckedIndexedAccess`, `noExplicitAny: error`, `useConst: error`
- All shared types in `src/types.ts`, all errors in `src/errors.ts`
- JSON errors go to stdout (ecosystem convention), not stderr

### Testing Philosophy
- **Real implementations over mocks**: real SQLite, real git repos, real filesystem
- **Only mock**: tmux (interferes with dev sessions), external AI services, network requests
- **DI over mock.module()**: `CoordinatorDeps`, `StopDeps`, etc. — avoids process-global leak (mulch record `mx-56558b`)
- Tests colocated: `{module}.test.ts` next to `{module}.ts`

### Subprocess Execution
- All external commands via `Bun.spawn` with stdout/stderr capture
- Check exit codes, throw typed errors on failure
- `PATH_PREFIX` prepends `~/.bun/bin` in hooks (Claude Code runs with minimal PATH)

### SQLite
- Always `bun:sqlite` synchronous API
- Always WAL mode + `busy_timeout=5000`
- `close()` attempts `PRAGMA wal_checkpoint(PASSIVE)` (best-effort)
- Migrations: `PRAGMA table_info` checks + `ALTER TABLE` or `CREATE TABLE IF NOT EXISTS`

### Agent Definitions
- Named failure modes in `ALL_CAPS` (in-context checklist)
- "Propulsion Principle" in all 7 agents: "Execute immediately. Do not ask for confirmation."
- Template placeholders (`{{TRACKER_CLI}}`, `{{QUALITY_GATE_INLINE}}`) in live `.overstory/agent-defs/`
- Hardcoded CLI names (`bd`, `mulch`) in published `agents/`

## Gotchas

### Critical Ordering Dependencies
1. **Session recorded BEFORE beacon sent** (`sling.ts`, `coordinator.ts`, `monitor.ts`): Hooks fire on tmux session start and need the session in `sessions.db` for `booting → working` transition.
2. **Dispatch mail sent BEFORE tmux session created** (`sling.ts`): Agents check mail on `SessionStart` hook.
3. **Synthetic session-end events written BEFORE databases wiped** (`clean.ts`): Event data needs session data to populate.

### Known Quirks
- **Two independent hand-rolled YAML parsers**: `config.ts` and `identity.ts` each have their own, with different semantics.
- **`logs.ts` vs `trace`/`errors`/`replay`**: `logs` reads raw NDJSON from disk; the others query `events.db`. Overlapping data, separate pipelines. NDJSON is ground truth.
- **`logs --limit N` returns most-recent N** (opposite of most tools — `slice(-limit)` not `slice(0, limit)`).
- **`mail check`/`checkInject` are destructive reads**: Messages marked read as side effect.
- **Chalk level set at import time**: Testing `NO_COLOR` requires subprocess spawn.
- **`molecules.ts` targets a planned `bd mol` API that doesn't exist yet**: Will fail in production.
- **Coordinator/monitor are `PERSISTENT_CAPABILITIES`**: Stop hook fires every turn, must NOT mark as completed.
- **macOS `/var` → `/private/var` symlink**: Tests use `realpathSync` to normalize paths.
- **`worktree clean` always uses `force: true` internally**: Deployed `.claude/` files create untracked files.
- **`merge queue.dequeue()` physically deletes rows**: Crash during merge = lost entry.
- **`resolveModel` maps provider-prefixed models to `sonnet` alias**: Real model injected via `ANTHROPIC_DEFAULT_SONNET_MODEL` env var (Claude Code gateway workaround).

### Security Notes
- Hook guards use `OVERSTORY_AGENT_NAME` env check — no-op for orchestrator's own session
- `sanitizer.ts` redacts: `sk-ant-*`, `github_pat_*`, `Bearer <token>`, `ghp_*`, `ANTHROPIC_API_KEY=*`
- Path boundary guards prevent agents from accessing files outside their worktree
- Non-implementation agents blocked from all file-write bash patterns

## Navigation Guide

**To add a new CLI command**: Create `src/commands/{name}.ts` with `create{Name}Command()` factory, register in `src/index.ts`, add to `COMMANDS` array in `src/commands/completions.ts` (static registry, count locked at 33 in tests).

**To add a new runtime adapter**: Implement `AgentRuntime` interface from `src/runtimes/types.ts`, register in `src/runtimes/registry.ts` factory map. See `docs/runtime-adapters.md` for the 5-step guide.

**To add a new agent type**: Add entry to `agents/{name}.md` (base definition), update `agent-manifest.json` with capabilities/model/tools, add to `src/agents/guard-rules.ts` if capability-specific guards needed.

**To add a new doctor check**: Create `src/doctor/{category}.ts` exporting `DoctorCheckFn`, add to `ALL_CHECKS` array in `src/commands/doctor.ts`.

**To add a new tracker backend**: Implement `TrackerClient` interface from `src/tracker/types.ts`, add adapter in `src/tracker/{name}.ts`, register in `src/tracker/factory.ts`.

**To understand the spawn pipeline**: Start at `src/commands/sling.ts:slingCommand()` — follows 14 sequential steps with extensive inline comments.

**To understand messaging**: `src/mail/store.ts` (SQLite) → `src/mail/client.ts` (logic) → `src/commands/mail.ts` (CLI). Broadcast: `src/mail/broadcast.ts`.

**To understand merge conflict resolution**: `src/merge/resolver.ts:resolve()` — 4 tiers with fall-through.

**To run quality gates**: `bun test && biome check . && tsc --noEmit` (or `bun run test && bun run lint && bun run typecheck`).
