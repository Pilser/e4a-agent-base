# E4A Agent Base — Future Plan (no code changes yet)

> Source: fork snapshot of `zerostack` (minimal Rust coding agent, ~30k LoC core, 26MB binary,
> ~16MB RAM). Cloned from `https://github.com/Pilser/zerostack`, package renamed
> `zerostack` -> `e4a-agent-base`. Upstream history detached (`.git` removed); this directory
> is now the single base repo for both agent shapes below.
> License: **GPL-3.0-only** (inherited — see §5, it constrains how we link this code).

## 1. One repo, two build shapes (future work, NOT implemented)

```
e4a-agent-base/
├─ src/...            # upstream code, untouched for now
├─ Cargo.toml         # renamed package; feature split comes later
└─ E4A-AGENT-PLAN.md  # this file
```

| Shape | What it is | Where it runs | Terminal? | FS? |
|---|---|---|---|---|
| **Embed lib** (`e4a-agent-base` as rlib, minimal features) | Stripped agent core for simple non-FS tasks (prompt build, tool-schema, quiz/chat orchestration). Linked in-process by `e4a-engine` / `e4a-agent` crate. | In-process (native + WASM) | No | No — all FS/tool modules compiled out |
| **Full runner** (`e4a-agent-runner` binary, full features) | Dedicated agent with full filesystem inside its workspace dir. Driven by engine, streams events back. | Docker/Swarm container per active task (§5 of arch doc) | No terminal (headless `--headless`/RPC mode; TUI stays for dev only) | Yes, jailed to `/var/e4a/workspaces/{school}/{user}/` via `e4a-sandbox` + `e4a-workspace` caps |

Both shapes stream to `e4a-engine`, which forwards to frontends:
`runner/lib -> engine.events() {JobProgress, Toast, ChatDelta} -> shell-slint set_property / server SSE -> Slint + React`.

## 2. Strip list for the embed lib (later)

Compile out via features / `cfg`: `src/agent/tools/{bash,edit,write,read,grep,find_files,list_dir}` (all FS),
`src/fs.rs`, `src/sandbox.rs`, TUI (`src/ui`, `crossterm`), `git-worktree`, `acp`, `hooks`, `lsp`,
`multimodal`, `pdf`, `memory`, `advisor`, `export`. Keep: agent runner core, prompt system,
permission/type layer (reused as capability tiers), retry, provider client, MCP client, session types.
Target: small crate (`no_std`-friendly where practical, `wasm32` check in `cargo xtask wasm`).

## 3. Full runner deltas (later)

* Default-on headless mode: no TUI, JSON/event stream on stdout or socket (`status-signals` pattern).
* Workspace jail: cwd forced under assigned workspace dir; `AuthorizedWorkspace` relative-path validation.
* Role flag: `--role {tutor,teacher,career,analyst}` mapping to `e4a-sandbox` tier allowlists.
* Engine client: MCP/JSON-RPC with scoped token `{user, role, school, exp}`; only allowlisted `CommandId`s dispatchable.

## 4. Non-goals / sequencing

1. Now: rename + this doc only. Upstream code untouched, no feature edits, no builds.
2. Next: decide lib API surface (`AgentHandle::{prompt,stream,cancel}`) against `e4a-agent` crate needs.
3. Then: feature-gate the strip list; add `embed` vs `full` CI jobs.
4. Last: Dockerfile + `swarm-stack.yml` wiring via `e4a-sandbox`.

## 5. Licensing constraint (read before linking)

Upstream is `GPL-3.0-only`. A renamed package does not change that: any binary statically
linking this code (including an in-process embed lib inside `e4a-engine`) must comply with GPL-3.0
(copyleft covers the combined work on distribution). Options to decide later: (a) keep the agent
always as a separate process (MCP/RPC boundary = no linking, engine stays permissive-licensed),
(b) relicense negotiations upstream, or (c) accept GPL for the whole stack. Do NOT embed before this
is decided. The full-runner-in-Docker shape (separate process) is license-clean by design.

## 6. Adopted structure (server-side embed allowed, shipped clients clean)

Deployment reality: GPL-covered code runs ONLY on our servers; users receive frontend shells only.
Under GPL-3.0 this is private/SaaS use — no conveyance, no source-release duty. This holds ONLY if
no GPL-linked binary is ever distributed (no agent code in shipped Slint apps, WASM bundles, Docker
images given to third parties, or contractor copies outside the org). Rules:

1. `e4a-agent` (thin, permissive) defines an `AgentHandle` trait (`prompt`, `stream`, `cancel`) and
   knows nothing of zerostack internals. The future clean-room rewrite implements this same trait —
   the swap is one `impl`, not a restructure.
2. `e4a-agent-base` embed is behind an explicit Cargo feature (`embed`, default OFF). `e4a-engine`
   exposes matching optional feature `agent-embed`, enabled ONLY for `e4a-server` builds.
3. Shipped clients (`e4a-app`, `e4a-web`, React bundle) MUST NOT enable it — enforced in CI
   (`cargo xtask layers` asserts no app target depends on `e4a-agent-base`; `cargo tree` audit).
4. Past distributions can't be uncured by a later rewrite, so rule 3 holds from the first build.
5. If distribution needs ever change (on-premise school servers, app-store binaries with agent
   inside), revisit §5 BEFORE shipping — at that point choose relicense or clean-room early.
