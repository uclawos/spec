# UCP — U-ClawOS Protocol v0.2 (Draft)

> **Status**: Draft  ·  **Version**: 0.2.0-draft  ·  **Date**: 2026-04-30
> **License**: MIT (this spec)  ·  **Editors**: U-ClawOS Working Group
>
> **Chinese authoritative version**: [`docs/14-UCP协议规范.md`](../docs/14-UCP协议规范.md) (the Chinese doc is the working source; this English file is a synchronised abstract)

---

## 1. What is UCP

**UCP** (U-ClawOS Protocol) is a **desktop AI agent orchestration protocol**. It defines:

- How a single user machine (Windows / macOS / Linux) hosts multiple AI agent CLIs (OpenClaw, Hermes, Goose, Claude Code, Codex, OpenCode, …)
- How those agents are uniformly **installed, dispatched, coordinated, supervised, and isolated**
- How users and external entry points (IM, browser, CLI) submit tasks to the system

UCP is to AI agent CLIs what LSP is to language servers, or what MCP is to tool servers — but operating one layer up: **the agent itself, not the tool**.

The protocol is intentionally **process-centric, file-driven, and editor-neutral**. It is built on three plain TOML/JSON Schema documents and five runtime primitives.

## 2. Design principles

1. **Determinism over convenience** — install/uninstall scripts must be idempotent, observable, and reversible. No ambient state.
2. **Files as the source of truth** — `agent.toml` and `job.toml` are the contract. SDKs are conveniences; humans must be able to read and write the files directly.
3. **Process isolation by default** — every agent runs in its own process tree, scoped working dir, and explicit network allow-list. "Trust me bro" agents are not first-class.
4. **Open at every seam** — anyone can write a new agent (`agent.toml`), a new orchestrator (replacing Goose), or a new inbound (IM platform). No central registry blocks innovation.
5. **Backward compatibility within a major version** — v0.x is allowed to evolve, but a v0.2 SDK must accept v0.1 manifests (see §4.4 compatibility).

## 3. Three-layer architecture

```
┌─────────────────────────────────────────────────────────────┐
│  L3 — Inbound (where work originates)                       │
│        IM bridges, browser UI, CLI, schedulers, USB events  │
└────────────────────────┬────────────────────────────────────┘
                         │  job.toml
┌────────────────────────▼────────────────────────────────────┐
│  L2 — Orchestrator (the dispatcher)                         │
│        reads agent.toml + job.toml; assigns work; supervises│
│        Reference impl: Goose. Replaceable.                  │
└────────────────────────┬────────────────────────────────────┘
                         │  ACP / MCP / HTTP / stdio
┌────────────────────────▼────────────────────────────────────┐
│  L1 — Agent (the worker)                                    │
│        any process that ships an agent.toml + speaks one    │
│        of the supported runtime protocols                   │
└─────────────────────────────────────────────────────────────┘
```

## 4. `agent.toml` — Agent manifest

Every UCP-compliant agent ships an `agent.toml` declaring metadata, install, runtime, capabilities, sandbox, uninstall, and health checks.

### 4.1 Required sections

```toml
[meta]
id = "openclaw"           # globally unique; lowercase + hyphen
display = "OpenClaw"
ucp_version = "0.2"       # required; SDK enforces compatibility
license = "MIT"
description = "..."

[install]
strategy = "script"       # script | npm | pip | binary | docker
[install.windows]
cmd = "powershell -ExecutionPolicy Bypass -File install.ps1"
prerequisites = ["node>=20"]
mirror.npm = "https://registry.npmmirror.com"

[install.offline]         # optional but encouraged for heavy agents
bundle_url = "https://uclawos.org/bundles/openclaw-2026.3.13.tar.gz"
bundle_sha256 = "..."
fallback_mirrors = ["..."]

[runtime]
type = "http_mcp"         # http_mcp | acp | stdio_mcp | http_api | cli_oneshot | standalone
endpoint = "http://127.0.0.1:18788/mcp"
health_check = "http://127.0.0.1:18789/health"
startup_cmd = "openclaw gateway run --port 18789"
shutdown_cmd = "openclaw gateway stop"

[capabilities]
tags = ["code", "shell", "browser"]
languages = ["zh", "en"]
streaming = true

[sandbox]
isolation_level = "process"      # process | container | none
network_policy = "limited"       # full | limited | offline
allowed_domains = ["api.openai.com", "api.u-claw.org"]
filesystem_policy = "scoped"     # full | scoped | readonly
working_dir = "~/.uclaw/agents/openclaw/workspace"

[uninstall]
preserve_user_data = true
cleanup_dirs = ["~/.uclaw/agents/openclaw/workspace"]

[[health.checks]]
name = "version"
type = "command"            # command | tcp_port | http_get
cmd = "openclaw --version"
timeout_ms = 3000
```

The full schema (machine-readable) lives at [`schemas/agent.schema.json`](./schemas/agent.schema.json) and is enforced by `sdk-rust` via `validate_agent()`.

### 4.2 Schema validation

Every `agent.toml` must pass `schemas/agent.schema.json` before it can be listed in any orchestrator. Validators:

- **Rust**: `ucp-cli validate ./agent.toml`
- **Web**: `https://uclawos.org/validator` (planned)

### 4.3 v0.2: `runtime.type = "standalone"` — software-store mode

Many AI tools in the wild (video generators, design suites, local-model runners) ship a complete UI of their own. They do **not** want to be dispatched by an orchestrator — users open them directly.

UCP v0.2 introduces `runtime.type = "standalone"` so these tools can still be **installed, uninstalled, and health-checked** by a UCP host without participating in the dispatch flow.

When `runtime.type = "standalone"`:

- The orchestrator **MUST NOT** include this agent in `dispatch()` candidates.
- The host UI **SHOULD** render an "Open" button using `runtime.web_url` or `runtime.native_entry`.
- At least one of `web_url` / `native_entry` is required (enforced by JSON Schema `if/then/anyOf`).

```toml
[runtime]
type = "standalone"
web_url = "http://localhost:8501"          # opened in user's default browser
# or
native_entry = "%LOCALAPPDATA%/foo/foo.exe"  # spawned via OS shell

[capabilities]
tags = ["video", "ai-app"]                  # use ai-app to mark non-worker
```

`install_agent` / `uninstall_agent` / `health_check_agent` work identically for standalone agents — only `dispatch()` is short-circuited.

### 4.4 Version compatibility

`uclawos-sdk` v0.2 enforces:

| `meta.ucp_version` declared by agent | Accepted by v0.2 SDK? |
|---|---|
| `0.1` | ✅ (backward-compatible) |
| `0.2` | ✅ (self) |
| `0.3` | ❌ (SDK too old; new fields unrecognised) |
| `1.0` | ❌ (major version mismatch) |

Rule: same major, declared minor ≤ SDK minor.

## 5. `job.toml` — task language

A job describes work to be done. The simplest form is a single intent string; the richest is a multi-step DAG with constraints, routing preferences, triggers, and an autonomy section for unattended execution.

```toml
[job]
id = "morning-standup-2026-04-30"
intent = "Summarise yesterday's git commits into a daily report"
mode = "auto"                          # interactive | semi-auto | auto

[job.constraints]
max_token_cost_cny = 1.0
max_runtime_minutes = 5
timeout_action = "abort"               # abort | notify | save-progress

[job.routing]
required_capabilities = ["code", "file-read"]
preferred_agent = "claude-code"        # hint, not requirement

[[job.steps]]                          # optional: multi-agent DAG
id = "gather"
agent = "openclaw"
prompt = "Find files changed this week under ~/Desktop/projects"
output_var = "changed_files"

[[job.steps]]
id = "analyze"
agent = "claude-code"
needs = ["gather"]
prompt = "Read git log of {{changed_files}} and group by project"
output_var = "summary"
```

See `docs/14` §5.1–5.4 (Chinese) for full syntax including triggers (manual/schedule/event/im_message) and autonomy bounds.

## 6. Five runtime primitives

Every UCP host (orchestrator) MUST implement:

| Primitive | Purpose | Failure mode |
|---|---|---|
| `install(agent_id)` | Run `[install]` script idempotently; stream logs | Roll back partial state, return error |
| `uninstall(agent_id)` | Run `[uninstall]` script; respect `preserve_user_data` | Best-effort; never destroy user data without consent |
| `health_check(agent_id)` | Run all `[[health.checks]]`; return aggregate report | Each check has its own timeout; one check failing ≠ agent down |
| `dispatch(job)` | Pick an agent matching `job.routing.required_capabilities`; run | If no agent matches, return `NoCandidate` |
| `supervise(job_id)` | Watch a running job; surface progress and faults | Faults trigger `timeout_action` |

### 6.1 Reference dispatch algorithm

```
input:  job J, list of installed agents A
output: an agent_id, or NoCandidate

1. filter A by health_check passing                        → A1
2. filter A1 where runtime.type ≠ "standalone"             → A2
3. filter A2 by capabilities.tags ⊇ J.routing.required_capabilities → A3
4. if J.routing.preferred_agent ∈ A3, return it
5. else return any agent in A3 (implementation-defined)
6. if A3 is empty, return NoCandidate
```

Step 2 is the v0.2 short-circuit for `standalone` agents.

## 7. Isolation model

Process-level isolation is the default and SHOULD be enforced by the host:

- **Process**: separate PID, scoped `working_dir`, separate stdout/stderr captured to `log_dir`
- **Network**: `network_policy = "limited"` enforces an allowlist of `allowed_domains`. The host MAY proxy or sniff to enforce; at minimum it documents the policy
- **Filesystem**: `filesystem_policy = "scoped"` constrains writes to `working_dir` plus user-granted directories

Container isolation (`isolation_level = "container"`) is reserved for v0.3.

## 8. Install robustness

§8 of the Chinese spec lays out three rules every install script MUST follow:

1. **Always have an exit path** — every failure mode reports a non-zero exit code with a stderr message; never hang.
2. **Progress must be observable** — write to stdout in real time (the SDK captures it as `InstallEvent::Log`).
3. **Offline-capable** — heavy agents (>5 MB compressed) SHOULD provide `[install.offline]` with `bundle_url` + `bundle_sha256` + `fallback_mirrors`. Lightweight agents (npm-only with <5 MB) MAY skip it; npm cache is sufficient.

## 9. Night shift (autonomous mode)

`[trigger.schedule]` with `only_when_idle = true` enables CPU-aware scheduling. `[job.autonomy]` enables unattended execution with hard safety bounds:

```toml
[job.autonomy]
allow_destructive = false              # rm -rf, git push --force etc.
allow_network = "egress-only"          # full | egress-only | none
allow_commit = false                   # only `git stage`, no commit
report_to = "feishu://#daily-ai"
escalate_to_user_if = ["test_failed", "ambiguous_intent"]
```

When `allow_destructive = false`, the orchestrator MUST refuse any `Step` whose `prompt` cannot be statically proven non-destructive. Reference implementations parse the prompt with the orchestrator's own LLM as a pre-check.

## 10. Open seams (extending UCP)

| Layer | What you write | Effort |
|---|---|---|
| New agent (lightest) | `agent.toml` + `install.{ps1,sh}` + `uninstall.{ps1,sh}` | hours |
| New inbound | A process that emits `job.toml` + POSTs to host's IM webhook | days |
| Replace orchestrator | Implement the 5 primitives above; speak the runtime types you support | weeks |

`uclawos/sdk` provides Rust functions for all five primitives. Other languages can either parse the TOML directly or call out to `ucp-cli`.

## 11. Relation to existing protocols

| Protocol | What it standardises | UCP relationship |
|---|---|---|
| **MCP** (Anthropic) | Tools an agent can call | Orthogonal — UCP agents may expose MCP servers, but UCP itself does not assume MCP |
| **ACP** (Zed Industries) | Editor ⇆ coding agent JSON-RPC over stdio/WS | UCP `runtime.type = "acp"` reuses ACP's wire format directly |
| **LSP** | Editor ⇆ language server | Different layer; LSP is per-file diagnostics, UCP is per-project agents |
| **OpenAI Chat Completions** | LLM API request/response | UCP doesn't constrain how an agent talks to its LLM; that's an agent-internal concern |

## 12. v0.2 scope and exclusions

In scope (v0.2 release):

- ✅ `agent.toml` schema (with `standalone` runtime type)
- ✅ `job.toml` schema (single-agent + DAG + autonomy)
- ✅ Five runtime primitives, reference Rust impl in `uclawos/sdk`
- ✅ Process isolation, network/filesystem policy, allowlist enforcement
- ✅ Backward-compat rule (v0.2 SDK ⊇ v0.1 agents)

Out of scope (deferred to v0.3+):

- Container isolation (`isolation_level = "container"`)
- Inter-agent message bus
- Federated multi-machine UCP
- Skill protocol (sub-agent capability composition)
- Formal capability ontology (currently `tags` are free-form strings)

## Appendix A: Reference adapters (v0.2 baseline)

In `uclawos/agents`:

| Agent | Runtime type | Status |
|---|---|---|
| `goose` | `acp` | Stable; reference orchestrator |
| `openclaw` | `http_mcp` | Stable |
| `hermes` | `http_mcp` | Reasoning worker |
| `claude-code` | `acp` | Wraps `@agentclientprotocol/claude-agent-acp` |
| `codex` | `acp` | Wraps `@zed-industries/codex-acp` |
| `opencode` | `acp` | Native `opencode acp` mode |

## Appendix B: Glossary

- **Agent** — a process that does work on the user's behalf via an LLM
- **Orchestrator** — the L2 dispatcher that reads `agent.toml` + `job.toml`
- **Inbound** — any source of jobs (IM, CLI, scheduler)
- **ACP** — Agent Client Protocol (Zed Industries); JSON-RPC 2.0 over stdio/WS
- **MCP** — Model Context Protocol (Anthropic); tool server protocol
- **Standalone agent** — v0.2: agent installed by host but launched by user (no dispatch)

## Appendix C: Reference implementation

- Spec & schemas: [`uclawos/spec`](https://github.com/uclawos/spec) (this repo)
- Rust SDK: [`uclawos/sdk`](https://github.com/uclawos/sdk) (`uclawos-sdk` crate, current `v0.2.0`)
- Reference adapters: [`uclawos/agents`](https://github.com/uclawos/agents)
- Reference desktop host: [`uclawos/desktop`](https://github.com/uclawos/desktop) (private)
