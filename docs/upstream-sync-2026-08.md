# Upstream sync 2026-08 — seam map, commit classification, migration design

**Status:** knowledge capture only. Nothing was rebased, nothing was deleted, no
rev pin was changed, nothing was pushed. This document exists so the next sync
starts from measured facts instead of re-deriving them.

**Branch:** `overhaul/upstream-sync` (off `main`, doc-only commit).

| Fact | Value |
| --- | --- |
| Fork `main` | `3775aae0` |
| Merge base with upstream | `329c1f9bb` (2026-07-08) |
| Upstream ref used for verification | `1da014e2f` |
| Upstream `main` at time of writing | `7d84a39b8` (208 commits ahead of `1da014e2`) |
| Toolchain | `nightly-2026-07-05` (upstream pin; fork `main` still pins `nightly-2026-02-19`) |
| Fork's own patch, `src/` only | **3,668 insertions / 496 deletions across 20 files** |
| …of which `src/tools.rs` | **2,099 insertions — 57% of the entire fork patch** |

That last row is the whole argument. Until `tools.rs` stops being touched, every
upstream sync re-fights the same battle: upstream's `tools.rs` is now 20,761
lines and churned +10,452/−2,507 in this window alone.

---

## 0. Two verification results that change the plan

### 0.1 Upstream `main` does not currently build

`cargo check --all-targets` at upstream `7d84a39b8` fails, **EXIT=101**:

```
error[E0382]: borrow of moved value: `missing_reason_set`
  --> tests/bench_schema.rs:4933:32
```

The library compiles; only the `bench_schema` test target is broken. This is an
upstream defect, not ours. `1da014e2` is clean (`EXIT=0`), which independently
justifies pinning verification work to `1da014e2` rather than upstream HEAD.
**Do not start the real rebase on `7d84a39b8`** — either wait for upstream to fix
it, or land the one-line borrow fix upstream first.

### 0.2 "Cherry-picks cleanly" ≠ "builds" ≠ "tests pass"

The scouting pass verified the five upstream-PR candidates with cherry-pick
success only. Two of them do not survive a real build/test:

- `63fab1b0` compiles only after removing a `paused: None` line from
  `tests/compaction.rs` — a field that exists **only** because of the fork's
  `925d9d8d`. The commit silently carried a fork dependency into what was
  supposed to be a clean upstream patch.
- `372348cf` **deterministically breaks** `xdev::tests::one_liner_table_matches_live_descriptions`
  (verified: passes twice at base, fails twice on the branch). Upstream added an
  anti-drift gate in `src/xdev.rs` asserting each curated one-liner equals the
  live tool description's first sentence. The commit rewrites 8 tool
  descriptions and never updated the table. Both are fixed on the prepared
  branches below.

Take this as the general rule for the rest of the patch set: **symbol-match and
patch-apply are not evidence.** Section 2 applies the same standard and
overturns two more scouting verdicts.

---

## 1. Upstream seam map

Upstream 0.3.0 grew the exact extension points the fork's largest patches were
invented for. These are the targets for re-expressing fork behaviour.

### 1.1 `ToolApprovalHandler` — the approvals seam

```rust
// src/agent.rs:769
pub type ToolApprovalHandler =
    Arc<dyn Fn(ToolApprovalRequest) -> BoxFuture<'static, ToolApprovalDecision> + Send + Sync>;

// src/agent.rs:701 — wired on AgentConfig
pub tool_approval: Option<ToolApprovalHandler>,

// src/agent.rs:747 / :755
pub struct ToolApprovalRequest { pub tool_call_id: String, pub tool_name: String, pub arguments: Value }
pub enum ToolApprovalDecision { Allow, Deny { reason: String } }
```

### 1.2 `ToolFactory` — the tool-registry seam

```rust
// src/sdk.rs:424
pub trait ToolFactory: Send + Sync {
    fn create_tool_registry(&self, enabled: &[&str], cwd: &Path, config: &Config) -> ToolRegistry;
}
// src/sdk.rs:323 — SessionOptions.tool_factory: Option<Arc<dyn ToolFactory>>
pub fn default_tool_registry(enabled: &[&str], cwd: &Path, config: &Config) -> ToolRegistry;
```

`ToolRegistry` now exposes public `push`, `extend`, `from_tools`, `into_tools`,
`get`. Upstream's own doc comment (sdk.rs:439–442) names the two intended uses:
"wrap each tool with an approval gate, or add a `Task` tool that spawns a nested
session" — i.e. upstream anticipated both of the fork's largest `tools.rs`
patches. **This is the seam that gets the fork out of `tools.rs`.**

### 1.3 `AskHandler` — the ask-user seam, and the reference implementation

```rust
// src/ask.rs:85
pub type AskHandler = Arc<dyn Fn(AskRequest) -> BoxFuture<'static, Result<AskResponse>> + Send + Sync>;
// src/ask.rs:383 / :394 / :449
impl AskTool { pub fn set_handler(&self, h: AskHandler); pub fn install_channel_ui(&self, sender: Sender<AskUiRequest>); pub fn respond_ui(&self, id: &str, response: AskResponse) -> bool; }
pub const ASK_UI_TIMEOUT_MS: u64 = 300_000; // src/ask.rs:246
```

`install_channel_ui` (ask.rs:394–447) is **a complete worked example of exactly
the pattern the desktop migration needs**: a pending-request map, a
`oneshot` channel per request, send-out / await-back, a wall-clock timeout, and
map cleanup on every failure path. Section 4 copies its shape rather than
inventing one.

### 1.4 Other replacements present upstream

`src/secrets.rs` (`SecretsMode`, `AgentConfig.secrets`, `mask_secrets_in_output`,
`restore_secrets_inbound`) · `tools.rs` `cached_tool_output`/`tool_cache_key` ·
`src/jobs.rs` (`JOB_SCHEMA = "pi.bash_job.v1"`, `bash {background:true}`) ·
`src/subagents.rs` `SubagentTool` · `sdk.rs:1488 set_max_tokens` ·
`sdk.rs:1457 set_session_name` · `SessionOptions.compaction_settings:
Option<ResolvedCompactionSettings>` + `ResolvedCompactionSettings::with_mode_applied`.

---

## 2. Classification of the 34 fork commits

### 2.1 Clean — retire upstream (5 branches prepared, local only)

Each branch is one commit on `1da014e2`, verified with `cargo check --all-targets`
**and** a test run on `nightly-2026-07-05`. **Never pushed** — hand-off for a human.

| Branch | Commit | check | tests |
| --- | --- | --- | --- |
| `upstream-pr/hashline-edit-default-op` | `d0cc1e2d` | EXIT=0 | at baseline |
| `upstream-pr/sdk-prompt-with-content` | `ff5cc1f6` | EXIT=0 | at baseline |
| `upstream-pr/sdk-dump-system-prompt-env` | `97ff49bd` | EXIT=0 | at baseline |
| `upstream-pr/builtin-usage-guidance` | `372348cf` | EXIT=0 | **fixed** — see 0.2, +`src/xdev.rs` table sync |
| `upstream-pr/compaction-summary-as-reference` | `63fab1b0` | EXIT=0 | **fixed** — see 0.2; `cargo test --test compaction` → 149 passed / 0 failed |

Baseline noise: `btw::tests::for_model_entry_rejects_credentialed_provider_without_key`
fails at `1da014e2` **before any of our changes** — it asserts `BtwClient::for_model_entry`
returns `None` without a credential, but this machine resolves a key from local
config, so it returns `Some`. Environment contamination, pre-existing, unrelated.
`extensions::tests::reactor::*`, `extensions::tests::core::*` and other
`extensions` tests are flaky run-to-run (different tests failed on different runs
of the same tree); treat them as noise, not regressions.

**Keep in fork, not PR-able:** `30abfca8` (brand-aware prompt via
`PI_AGENT_NAME` / `PI_AGENT_HIDE_PI_DOCS`) — LibertAI-specific, ~50 lines in
`app.rs`, applies clean, upstream would reject it.

### 2.2 "Superseded" — behavioural verdicts (NOTHING DELETED)

Verdicts are behavioural, per the gate. **Three of the twelve are not
superseded**, and one of those would have been a security regression.

| Commit | Upstream replacement | Verdict |
| --- | --- | --- |
| `c601fa17` bash command_wrapper | `config.shell_command_prefix` | 🔴 **NOT SUPERSEDED — security** |
| `d8c63fc8` + `1120076c` task tool | `src/subagents.rs` `SubagentTool` | 🔴 **NOT SUPERSEDED — compat** |
| `1d0bc449` per-project memory + git context | `src/memory.rs`, `src/context_files.rs` | 🔴 **NOT SUPERSEDED — wrong match** |
| `f76ac5c1` AutoCompactionEnd payload | `AutoCompactionEnd.result` | 🟡 needs-shim |
| `1f015bd0` redact secrets from reads | `src/secrets.rs` | 🟡 needs-verification |
| `f0e891a3` dedupe repeated reads | `cached_tool_output` | 🟢 safe-to-drop |
| `edc37524` + `c9058e45` background bash | `src/jobs.rs` | 🟢 safe-to-drop |
| `c86fa0f1` `set_max_tokens` | `sdk.rs:1488` | 🟢 safe-to-drop |
| `d7005c57` session rename | `sdk.rs:1457` | 🟢 safe-to-drop |
| `4574203e` compaction session overrides | `SessionOptions.compaction_settings` | 🟢 safe-to-drop |

#### 🔴 `c601fa17` — dropping this silently disables `--sandbox`

These are **different mechanisms**, not different spellings.

Upstream (`src/tools.rs:6503`) concatenates prefix text into the script body:

```rust
let command = command_prefix.filter(|p| !p.trim().is_empty())
    .map_or_else(|| command.to_string(), |prefix| format!("{prefix}\n{command}"));
// ... then: shell -c <command>
```

The documented example is `"set -euo pipefail"` (config.rs:2497). It runs
**inside the same unsandboxed shell** and provides **zero** isolation.

The fork replaces the executable itself — `bwrap …  /bin/bash -c "cmd"`:

```rust
let mut cmd = command_with_default_sigpipe_in_dir(&wrapper[0], cwd)?;
if wrapper.len() > 1 { c.args(&wrapper[1..]); }
```

A symbol-match retirement here would have removed libertai-cli's `--sandbox`
enforcement while leaving the flag in place — failing open, silently.

**Viable shim:** upstream's `shell_path` (`config.rs:165`) *is* used as the
executable (`command_with_default_sigpipe_in_dir(shell, cwd)` then `.arg("-c")`).
Point `shell_path` at a launcher script that execs `bwrap … /bin/bash "$@"` and
sandboxing is recovered with no `tools.rs` patch.
**Gap:** `shell_path` is Config-level only — there is **no** `SessionOptions`
override, whereas the fork added `SessionOptions::bash_command_wrapper`
specifically for per-session `--sandbox`. Also `Option<String>` vs the fork's
`Option<Vec<String>>` loses argv-boundary safety (spaces in paths). Either accept
process-global sandboxing, or land a small upstream PR adding an argv-shaped
override. Decide before dropping.

#### 🔴 `d8c63fc8` + `1120076c` — a compatibility surface, not a duplicate feature

| | fork `task` | upstream `subagent` |
| --- | --- | --- |
| name | `task` | `subagent` |
| required | `prompt` | `agent` **and** `task` (per task) |
| `additionalProperties` | permitted | **`false`** |
| child agent | anonymous, general-purpose | must resolve a **named definition file** in `$PI_CODING_AGENT_DIR/agents/*.md` or `.pi/agents/*.md` |

The fork's schema documents `subagent_type` as "Optional routing hint for **Claude
Code-compatible callers**". A Claude-Code-compatible client emitting
`{"name":"task","input":{"prompt":…}}` gets tool-not-found against upstream, and
renaming does not rescue it: `prompt` is rejected by `additionalProperties:false`
and `agent` is required. This is wire compatibility, and upstream does not
provide it.

**Re-express, don't drop:** register a `task` tool from a `ToolFactory` impl in
libertai-cli. This is precisely upstream's suggested use and it removes ~412
lines from `tools.rs`.

#### 🔴 `1d0bc449` — the scouting match was simply wrong

Two claims fail on inspection:

- **`MemorySettings` does not exist** anywhere in upstream `src/memory.rs`.
- Upstream `src/memory.rs` is an **unrelated subsystem**: an agent-facing memory
  *tool* set (`MemoryStore`, `RetainTool`, `RecallTool`, `ReflectTool`,
  `MemoryEditTool`, schema `pi.memory.v1`). The fork's commit is 86 lines in
  `app.rs` that (a) load a per-project memory *file* keyed by encoded cwd and
  (b) inject a **git context block** (branch, dirty files, recent commits).
- **No git-context block exists upstream** — `grep` for `git status` / `Current
  branch` / `git_context` across `app.rs`, `sdk.rs`, `context_files.rs` returns
  nothing.
- `app::encode_project_cwd` / `app::load_project_memory` exist **only** in the
  fork, and libertai-cli's `/remember` calls them. Dropping breaks the CLI.

`context_files.rs` (AGENTS.md / CLAUDE.md) overlaps in spirit only — repo-local
convention files, not a cwd-keyed project store. **Keep.** Cheap to carry: 86
lines, `app.rs` only, never touches `tools.rs`.

#### 🟡 `f76ac5c1` — field naming/nesting differs

Upstream `AgentEvent::AutoCompactionEnd` (agent.rs:1101) carries
`result: Option<Value>`, `aborted`, `willRetry`, `errorMessage`. Post-compaction
token counts live **nested inside `result`, snake_case** (`{"tokens_before":…,
"tokens_after":…}`). The fork emits **top-level camelCase** `tokensAfter`, plus
`durationMs` and `trigger` ("threshold" vs "manual"), neither of which was found
upstream. Any consumer reading top-level `tokensAfter` breaks on drop. Confirm
`durationMs`/`trigger` have no upstream equivalent, then either shim in the
consumer or PR the two fields upstream.

#### 🟡 `1f015bd0` — opposite security posture

Upstream's vault is **reversible by design**: `restore_secrets_inbound`
(agent.rs:3960) puts real values *back* into tool-call arguments before
execution, and re-masks on the way out. Default mode is `Obfuscate`, not `Off`.
The fork's read-redaction is **one-way**. So under upstream, a secret read out of
a file and fed back through a later tool argument is restored to plaintext —
deliberate, and arguably better (the operator approves the real command), but it
is a different guarantee. Confirm this matches LibertAI's threat model before
dropping; the code is genuinely superseded, the *policy* may not be.

#### 🟢 Safe to drop (verified present and behaviourally equivalent)

`f0e891a3` → `tool_cache_key`/`cached_tool_output` with dependency tracking
(tools.rs:2141–2151, applied to `read` at :5965), strictly richer than the fork's.
`edc37524`+`c9058e45` → `src/jobs.rs` + `bash {background:true}` (tools.rs:7123)
+ jobs/wait/cancel tools. `c86fa0f1` → `sdk.rs:1488`, and upstream additionally
**seeds** `max_tokens` from the model registry (sdk.rs:1802), which fixes the
fork's actual motivating bug (4096-token truncation) more thoroughly than the
fork did. `d7005c57` → `sdk.rs:1457`. `4574203e` → `SessionOptions.compaction_settings`.

### 2.3 Load-bearing — blocked on the cross-repo migration

`925d9d8d` (pause/resume primitive) changes `Tool::execute`'s return type from
`Result<ToolOutput>` to `Result<ToolExecution>` across ~46 builtin tools. It is
the single largest source of `tools.rs` conflict and the reason every future sync
hurts. It cannot be dropped from this repo alone (Section 4).

Moot the moment `925d9d8d` goes, meaningless to rebase before then:
`a0688156`, `8c581525`, `0a78a849`, `3775aae0` (test adaptations), plus
`b5ac64a7`/`f5425c7d`. `06ced4ef` is auto-dropped by git as patch-identical.

### 2.4 Residual — rebase normally, once 2.3 clears

Measured conflict load, small and tractable: `bc2b7663` keep_recent_tokens clamp
(1 hunk) · `730ef40f` fork_session (1) · `13a6411f` compact instructions (3) ·
`6fda82f8` (1) · `87367a84` live tool refresh (1) · `04e9a92b` live tool removal
(2) · `dba0ea25` stale writes (16, all `tools.rs`) · `28acb37a` compaction P2 (24
hunks, 6 files).

Move `dba0ea25` into a `ToolFactory` wrapper instead of resolving it in
`tools.rs`. Re-express `bc2b7663`/`28acb37a` on top of
`SessionOptions.compaction_settings` + `ResolvedCompactionSettings::with_mode_applied`
rather than adding new knobs.

### 2.5 Trial-rebase numbers (for planning)

Rebase onto `1da014e2` stopped at commit **5/34**, on `925d9d8d`: 8 conflict
hunks across `agent.rs` (2), `session.rs` (3), `tools.rs` (2), `extensions.rs` (1).
Independent per-commit cherry-pick survey: **11 clean / 24 conflicting, ~121
hunks** (an over-count — each was tested in isolation). Worst: `28acb37a` 24
hunks/6 files, `d8c63fc8` 18/5, `dba0ea25` 16 (tools.rs), `925d9d8d` 8,
`c601fa17` 8, `f76ac5c1` 8.

---

## 3. Resolved: when is `ToolApprovalHandler` invoked?

**The scout's open question. Answer: it depends on `approval_state`, and the
configuration we want is the trivial one.**

Read `Agent::request_tool_approval` (agent.rs:4493–4618), called unconditionally
from `execute_tool` (agent.rs:4337) before hooks and before execution:

1. **`config.approval_state == Some(_)`** → graduated gating runs *first*.
   `HardBlocked` returns a denial immediately; `AutoApproved` returns `None`
   (tool runs) — **neither ever reaches the handler**. Only
   `RequiresApproval` calls it (agent.rs:4539).
2. **`config.approval_state == None`** → the legacy path at agent.rs:4586 calls
   the handler on **every single tool call**, unfiltered.

**Consequence for `code_guardrail.rs` parity: leave `approval_state` as `None`.**
That is the default (`AgentConfig::default()`, agent.rs:781) and yields exactly
the total coverage the fork's `ToolExecution::Paused` guardrail has today. **No
`AlwaysAsk` configuration is required** — the concern the scout raised does not
arise, provided we do not opt into `approval_state`.

Two further facts that constrain the design:

- **The request is read-only.** `ToolApprovalRequest` carries
  `{tool_call_id, tool_name, arguments}` and the decision is `Allow` / `Deny{reason}`
  — there is **no** way to *rewrite* arguments through this seam. Any fork tool
  that mutates arguments on resume needs a `ToolFactory` wrapper instead.
- **Approvals run concurrently and are cancellable.** `execute_tool_batch`
  (agent.rs:4064) drives the batch with
  `stream::iter(futures).buffer_unordered(parallelism)`, so up to `parallelism`
  handler futures are in flight **on a single task**, and the whole batch is
  raced against the abort signal via `select`. Therefore: the UI must **queue
  multiple simultaneous approval requests**, and every handler future must be
  **cancellation-safe** — if it is dropped mid-await (abort), the pending-map
  entry and any open modal must be cleaned up. `AskTool::install_channel_ui`
  already does this cleanup on every path; copy it.

---

## 4. Migration design — `ToolExecution::Paused` → `ToolApprovalHandler` + `AskHandler`

### 4.1 Corrected scope

The scouting estimate of "7 libertai-cli files" is wrong in both directions:

- **26 files** in libertai-cli reference `ToolExecution` (187 references).
- But only **3** use `ToolExecution::Paused`: `code_ask_user.rs`,
  `code_approvals.rs`, `code_guardrail.rs`. (`code_approval_ipc.rs` and
  `code_term.rs` match on the fork's own `PromptChoice::Paused` / `AskOutcome::Paused`,
  which are LibertAI types and stay.)
- There are **0** uses of `ToolExecution::Complete`; the variant is
  `ToolExecution::Done(ToolOutput)` and there is an `impl From<ToolOutput>`.

So the work splits cleanly:

- **23 files — mechanical.** Change the signature `PiResult<ToolExecution>` →
  `PiResult<ToolOutput>` and unwrap `Done(...)` / drop the `.into()`. Because of
  the blanket `From`, many sites are already `output.into()` and become just
  `output`. Low risk, compiler-driven.
- **3 files + desktop — architectural.** Below.

### 4.2 The real desktop blocker (bigger than `block_on`)

`libertai-code-desktop/src-tauri/src/session.rs` runs each session on **a
dedicated OS thread with a current-thread runtime**, driving a request loop where
"each request runs on the asupersync runtime via `block_on`" (module docs,
session.rs:1–7). The existing call is session.rs:1285:

```rust
if let Err(e) = runtime.block_on(handle.resume_paused_tools()) { … }
```

and its own comment states: *"Blocks the worker thread on the user's modal
answer; request-loop messages queue on the mpsc until resume settles."*

Today that is **sound**, because the fork's suspension is **return-based and
out-of-band**: the tool *returns* `Paused{request_id, kind, payload}`, the agent
turn **ends**, and the modal answer is delivered later by a separate
`approval_respond` Tauri command. The agent loop is not running while the user
thinks.

Upstream's approval is **in-band**: the agent loop is *parked inside the handler
future* awaiting the decision. So the turn and the response path must be able to
make progress **at the same time**. With the current one-thread/`block_on`
request loop and the answer arriving over the same mpsc, that is a **guaranteed
deadlock** — not merely the "don't `block_on` inside the handler" hazard, but the
structural fact that the turn monopolises the worker thread.

### 4.3 Required design

Model the handler on `AskTool::install_channel_ui` (ask.rs:394–447) verbatim in
shape:

1. **Shared pending map**, `Arc<Mutex<HashMap<String, oneshot::Sender<Decision>>>>`,
   owned outside the session thread.
2. **Handler future**: mint a request id, insert the `oneshot::Sender`, emit the
   Tauri `approval:request` event to the FE, then `await` the receiver under a
   wall-clock `timeout` (mirror `ASK_UI_TIMEOUT_MS = 300_000`). It must contain
   **no `block_on`** and no lock held across an `.await`.
3. **Response path must bypass the session request loop.** The
   `approval_respond` Tauri command resolves the `oneshot` **directly** through
   the shared map — it must *not* be queued on the session mpsc, or it will sit
   behind the very turn that is waiting for it.
4. **Cleanup on every exit path** — timeout, closed channel, and **future
   dropped by abort** (agent.rs:4083 races the batch against the abort signal).
   Removing the pending entry and dismissing the modal must happen in all four.
5. **Concurrent requests**: `buffer_unordered` means several approvals can be
   open at once. The FE needs a queue/stack of modals keyed by request id, not a
   single global modal.
6. **`resume_paused_tools()` disappears**, along with the "re-fire tools that
   paused across an app exit" behaviour at session.rs:1285. In-band approvals
   cannot survive a process exit — a pending future dies with the process. If
   cross-restart resumption is a product requirement, it must be rebuilt on top
   of persisted session state, and that is **new work, not a migration**. Flag
   this to product before starting.

### 4.4 Per-file plan

**libertai-cli**

| File | Action |
| --- | --- |
| `code_approvals.rs` | Becomes the `ToolApprovalHandler` factory. `PromptChoice::{Allow,AllowSession,Prefix,GrantRoot,Domain,AlwaysAllow}` → `Allow`; `Deny` → `Deny{reason}`. The `Paused` arm is deleted; suspension is now the future itself. Rule persistence ("always allow in this directory") is unaffected. |
| `code_ask_user.rs` | Retarget onto `AskTool::set_handler` / `install_channel_ui` + `respond_ui` rather than the approval seam — this is a question, not a gate, and upstream already models it. Map `AskOutcome::Paused` onto the awaited handler; `dismissed: true` for cancel. |
| `code_guardrail.rs` | `ToolExecution::Paused{..} => None` arm (line 202) is removed. Coverage is preserved **only** by leaving `approval_state = None` (Section 3) — add a regression test asserting the handler observes a tool the guardrail must gate. |
| 23 other files | Mechanical signature change (4.1). |
| `code_term.rs`, `code_approval_ipc.rs` | Keep their own `PromptChoice::Paused` / `AskOutcome::Paused`; these are LibertAI enums. The terminal UI blocks to answer and never pauses (code_term.rs:191), so it maps to the handler with no behaviour change. |

**libertai-code-desktop**

| File | Action |
| --- | --- |
| `src-tauri/src/session.rs` | Install the handler at `AgentConfig.tool_approval`. Delete the `resume_paused_tools` call (1285) and the `PromptChoice::Paused` arms (3559, 3619). Restructure so the turn does not monopolise the worker thread, and route `approval_respond` to the pending map directly (4.3 §3). |
| FE modal counterpart | Key modals by request id and support a queue; handle server-side dismissal on timeout/abort. |

### 4.5 Ordering

The fork, both consumers, and both rev pins (`libertai-cli/Cargo.toml:124`,
`libertai-code-desktop/src-tauri/Cargo.toml:39`, both `rev = "3775aae0"`) must
move **atomically**. A rebase changes every SHA, so nothing lands until the
consumer migration compiles. Bump the pins last, in the same change.

Reference the July sync's actual conflict resolutions rather than re-deriving
them: `origin/backup/main-pre-upstream-sync-20260715` (`5bfa4a07`) is the
pre-rebase snapshot of a sync that **succeeded**, and `main` is its result. Diff
the two to recover how ~800 commits' worth of conflicts were resolved by hand.

---

## 5. Recommended next steps

1. **Human pushes the five `upstream-pr/*` branches** and opens PRs. Every clean
   commit accepted upstream is one that never conflicts again. Consider also
   sending the `tests/bench_schema.rs` borrow fix (0.1).
2. **Decide the three 🔴 verdicts** — sandbox mechanism (`c601fa17`), `task`-tool
   wire compatibility (`d8c63fc8`), project memory + git context (`1d0bc449`).
   None may be dropped as-is.
3. **Get the migration design in 4.3 human-approved**, especially the desktop
   threading restructure and the loss of cross-restart resume (4.3 §6).
4. **Only then** rebase, on a green upstream base, with all three repos moving
   together.
5. **Cadence.** Upstream moved 1,068 commits / 0.1.21 → 0.3.0 / 104 new source
   files in one window. A thin patch set stays thin only with monthly syncs; a
   quarterly cadence reproduces exactly this situation.
