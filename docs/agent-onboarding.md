# Supported coding agents

> Published from InternSwarm's code repository, which is not public yet, at 420ba314. Anything about building from source applies there.

InternSwarm runs any coding-agent CLI in a terminal it owns. A **verified**
agent gets more than a terminal: a status read from the structure of its
screen and its own records, notifications when it asks or finishes,
auto-answer, and mail typed straight into its input box. This page lists what
that takes, capability by capability: what the agent itself must provide, how
the app does it today, how it is measured, and who provides it on your
machine, the app alone, your dotfiles wrapper, or the agent's own settings
file, with what you lose without it.

It is written for two readers. A customer can use the table to see what each
agent does today and what a few minutes of setup buys. Someone adding an
agent can use the capability list as the definition of done.

**Verified today:** Claude Code and Codex. **Unverified:** Gemini CLI, Aider,
Goose and OpenCode. Antigravity (`agy`) is the next agent through this list:
its column below was measured on 2026-10-10, and nothing for it is built yet.
Statuses below were measured on Claude Code 2.1.288 and Codex 0.160.0.

## What "verified" switches on

`AgentType::tier()` in `src-tauri/src/models/agent.rs` returns `Verified` for
Claude Code and Codex only, and the client mirrors it in
`VERIFIED_AGENT_TYPES` (`src/lib/agentTier.ts`). Three things read it:

- **The screen watch.** A verified agent's channel is read by structure
  (`screen_structure::read`). Everything screen-based hangs off that: asks
  (a prompt the agent is waiting on), auto-answer, typed mail, waiting for
  the input box before typing, the unreadable-screen health marker, and
  Claude Code's end-of-turn finish.
- **Busy.** A verified agent's raw output never marks it busy; busy comes from
  its conversation region changing.
- **Notifications.** Only a verified agent's prompt becomes an "asks" notice.

An unverified agent still gets the terminal, a status from activity and
silence, messaging over MCP if it supports it, and mail delivered to its
inbox rather than typed into its prompt.

## At a glance

"App" means nothing to set up. "Wrapper" means your shell wrapper for the CLI
(see [Setting up your machine](#setting-up-your-machine)). "Agent settings"
means the CLI's own configuration.

| | Capability | Claude Code | Codex | Antigravity | Provided by |
|---|---|---|---|---|---|
| C1 | Registration and tier | ✓ | ✓ | ✗, runs as Gemini | App |
| C2 | Launch from a typed line, with the app's flags | ✓ | ✓ | ✓ | App, or wrapper |
| C3 | The process sampler names the agent | ✓ | ✓ | measured | App |
| C4 | The record read is this session's | ✓ | ✓ | measured | App; wrapper for a hand relaunch of Claude Code |
| C5 | Restart resumes the conversation | ✓ | ✓ | measured | App |
| C6 | Turns: start, end, finished or interrupted, length | ✓ | ✓ | measured, no turn end | App |
| C7 | A usage limit, with its reset time | ✓ | ✓ | – | App |
| C8 | Background work | ✓ Waiting | by design | – | App |
| C9 | Screen reader, per renderer | ✓ classic, partial full-screen | ✓ default and inline | measured, inline | App |
| C10 | Prompt options and the key that answers | ✓ | ✓ | measured | App |
| C11 | Auto-answer confirms a real tool call | ✓ | ✓ | – | App; wrapper for a hand relaunch of Claude Code |
| C12 | Notifications | ✓ | ✓ | ✗ | App |
| C13 | Typed mail | ✓ | ✓ | ✗, inbox only | App |
| C14 | Relayed text cannot forge the reader's marks | ✓ | ✓ | – | App |
| C15 | MCP messaging | ✓ | ✓ | partial | App; wrapper or agent settings for a hand relaunch; agent settings to approve the tool |
| C16 | Environment and the agent's own markers | partial | ✓ | measured, nested untested | How you start the app |
| C17 | Unreadable-screen health marker | ✓ | ✓ | – | App |
| C18 | Re-checked on every CLI update | ✓, with gaps | ✓, with gaps | ✗ | App repo |
| P1 | Model, effort, permission mode, tokens | ✓ | ✓ | measured | App, plus a shared store for wrappers |
| P2 | Account label, quota, several accounts | ✓ | ✓ | partial | Wrapper |

P1 and P2 are parity rows: nothing in the verified tier depends on them, but
a customer notices them first.

The Antigravity column is a measurement, not support. **measured** means
what the agent offers for that capability was observed and recorded, not
that it is enough, and the app uses none of it yet; ✗ means the app gives it
nothing today; – was not measured. The details are under
[Antigravity, measured](#antigravity-measured).

## The capabilities

Each entry says what the agent must provide, how Claude Code and Codex do it,
what measures it (unit tests in the app repo, and checks in the daily
[conformance suite](conformance-suite.md)), what passing means, and what you
lose without it.

### C1. Registration and tier

- **Needs from the agent:** nothing. Every per-agent `match` in the app
  names each variant and has no catch-all arm, so a new agent fails to
  compile until each one is decided.
- **Measured by:** `exactly_the_verified_agents_have_a_reader`,
  `agentTier.test.ts`, `only_a_verified_agents_prompt_asks`.
- **Pass:** the Rust tier, the client list and the screen-reader arms name
  the same agents.
- **Flip the tier last.** An agent marked verified with no screen reader is
  read as unrecognised on every look: it never reads busy, never asks, its
  typed mail waits for an input box that never appears, and the `?` marker
  shows after five minutes.

### C2. Launch from a typed shell line, with the app's flags

- **Needs from the agent:** an interactive terminal UI started from a line
  typed into the user's shell (typed, not exec'd, so a shell function
  resolves), accepting flags after the command word and in any position.
- **Claude Code / Codex:** the app types the command from Settings → Agent
  → Command, a single word, and appends its own flags.
- **Measured by:** unit tests in `channel_runtime_service.rs`; the
  conformance suite launches every session through an account wrapper.
- **Provided by:** the app, or your wrapper named in the Command field.
- **Without a wrapper:** one account per agent. The Environment tab can pick
  an account per session instead.

### C3. The process sampler names the agent

- **Needs from the agent:** a recognisable foreground process: its
  executable's name, its process name, or a `<agent>/versions/<version>`
  path. A CLI that updates itself while it runs can rename its own
  executable (Antigravity's became `antigravity.<number>.old`), so match
  the process name.
- **Claude Code / Codex:** `classify_agent_name` knows `claude`,
  `claude-code` and `codex`, each with or without `.exe`: Claude Code from
  npm runs as `bin/claude.exe`. Unix only; on Windows the sampler does nothing.
- **Measured by:** `classifies_known_agent_processes`,
  `classifies_claude_installed_by_npm_which_runs_as_claude_exe`; conformance
  `claude.sampler` and `codex.sampler`.
- **Pass:** from the first readable screen until the agent exits, the
  session never reads "no agent", and it does after the exit.
- **Without it:** the session reads "no agent": typed mail is refused,
  prompts never go live, and auto-answer never fires.

### C4. The record read is this session's, now

- **Needs from the agent:** a structured record per conversation, and a way
  to tie it to this terminal: either the app names the id at launch and is
  told about switches, or the record can be found from the process writing
  it.
- **Claude Code:** the app launches with `--session-id <id>`, or `--resume
  <id>` once a transcript exists, and a SessionStart hook in the app's
  `--settings` file reports the conversation after every `/resume`,
  `/clear` and `/compact`.
- **Codex:** the `rollout-*.jsonl` file a `codex` process in the session's
  foreground holds open; of several (Codex keeps earlier conversations open
  after `/new` and `/resume`), the most recently written that is not a
  subagent's. It works for any `CODEX_HOME`.
- **Measured by:** unit tests in `agent.rs`, `channel_runtime_service.rs`,
  `conversation_hook_service.rs` and `codex_log_service.rs`; conformance
  `codex.rollout-open`. No daily check reports a Claude Code conversation
  switch.
- **Pass:** two sessions in one folder each read only their own record, and
  after `/clear` or `/resume` the record followed is the new conversation.
- **Provided by:** the app for anything it launches. A Claude Code started
  by hand needs your wrapper to pass `--settings "$ISWARM_SETTINGS"`, and a
  wrapper that picks its own `CLAUDE_CONFIG_DIR` needs that directory's
  `projects/` linked to `~/.claude/projects`.
- **Without it:** on a hand relaunch without `--settings`, the panel speaks
  for the old conversation for up to an hour, then model, effort and tokens
  go blank, and the new conversation gets no finished or limit notices.
  Without the linked `projects/`, there is no record at all.

### C5. Restart resumes the conversation

- **Needs from the agent:** a resume-by-id flag, and C4's id.
- **Claude Code:** Restart agent and an app restart resume the conversation
  the session last reported, or say which fallback they took.
- **Codex:** Restart agent and an app restart resume the thread the app
  recorded from the rollout Codex holds open (`codex resume <id>`); with
  nothing recorded, or its rollout gone, they start fresh and the panel says
  so. On a hand relaunch, a wrapper's `--iresume` asks the app for the same
  thread (see [multi-account.md](multi-account.md#codex-session-tracking)).
- **Measured by:** unit tests such as
  `a_restart_resumes_the_tracked_conversation_when_its_transcript_exists` and
  `a_codex_restart_resumes_its_recorded_thread_before_the_messaging_flags`;
  conformance `claude.restart-resume` and `codex.restart-resume`.

### C6. Turns: start, end, finished or interrupted, length

- **Needs from the agent:** record rows written as they happen: turn start,
  turn end, an interrupt distinct from a clean end, and a turn length or the
  time of the person's prompt. A finished notice needs at least 5 seconds.
- **Claude Code:** `end_turn`, confirmed by `turn_duration`; interrupt rows.
  **Codex:** `task_started`, `task_complete` and `turn_aborted`, each with a
  `turn_id`.
- **Measured by:** recorded fixtures under `src-tauri/tests/fixtures/records`;
  conformance `claude.turn-rows`, `claude.finished-once`,
  `claude.interrupt-row`, `codex.turn-events`, `codex.finished-once`.
- **Pass:** one finished notice per clean end, never for an interrupt.
- **Without it:** no finished notice for Codex; for Claude Code, finished
  only from the end-of-turn line on screen.

### C7. A usage limit, with its reset time

- **Needs from the agent:** a structured refusal row, not words, carrying or
  implying when the limit resets.
- **Claude Code:** a `<synthetic>` row with `error: "rate_limit"` and
  `quotaLimits.resetsAt`. **Codex:** a `task_complete` whose
  `codex_error_info` is `usage_limit_exceeded`, with the time parsed from
  "try again at …".
- **Measured by:** unit tests; conformance `claude.limit-row` and
  `codex.limit-shape`, which read the account's own newest limit record when
  a run meets none, and report "unverifiable" with their last pass when there
  is none from the installed version.
- **Pass:** a refused turn reads rate-limited until its reset; a window at
  100% use alone is not a limit.
- **Without it:** no rate-limited badge or notice, and mail is not held for a
  limited agent.

### C8. Background work

- **Claude Code** resumes on its own when background work ends, so a turn
  that ends with work outstanding reads **Waiting**, and the finished notice
  waits until the work is done.
- **Codex** does not resume on its own: a turn that ends with a process
  still running is over, so it reads idle and the finished notice fires.
  The daily check `codex.background` fails the day a Codex starts resuming
  on its own, because "finished" would then fire mid-work.
- **Measured by:** conformance `claude.background`, `claude.waiting` and
  `codex.background`.

### C9. Screen reader, per renderer

- **Needs from the agent:** a stable screen shape that tells apart the input
  region, a prompt drawn in its place, and a conversation region that changes
  while the agent works, for every renderer the CLI offers.
- **Claude Code:** two rules framing a `❯` line, or a last rule with an "Esc
  to cancel" footer. **Codex:** the last `›` line with at most four footer
  rows below it, or a prompt footer.
- **Measured by:** recorded screens under `src-tauri/tests/fixtures/screens`
  (Claude Code classic and full-screen, Codex default and `--no-alt-screen`);
  conformance `*.idle-shape`, `*.approval-prompt`/`*.permission-prompt`,
  `claude.input-box-while-busy`, `claude.resize`. The daily run checks
  whichever renderer the test account uses, so only one per CLI.
- **Status:** Claude Code classic ✓, full-screen partial (no recorded busy
  screen); Codex default ✓, inline ✓ (seven recorded screens: the same
  shapes as the default renderer).
- **Provided by:** the app. Which renderer you use is the agent's own
  setting (`/tui fullscreen`, `codex --no-alt-screen`); nothing needs setting
  up.
- **Without it:** a `?` after five minutes, no asks, typed mail goes to the
  inbox, and no busy from the screen.

### C10. Prompt options and the key that answers

- **Needs from the agent:** prompts with at least two options numbered from
  1 (or an unnumbered menu, which is never answered automatically), a title,
  an "esc to …" hint, and a digit that selects and confirms.
- **Measured by:** parser tests built from captured screens; conformance
  `claude.trust-dialog` and `codex.trust-dialog`, checked once per CLI
  version.
- **Without it:** a prompt that does not parse is not an ask and is never
  auto-answered.

### C11. Auto-answer confirms a real tool call

- **Needs from the agent:** a signal, apart from the screen, that a real tool
  call is waiting for its answer, so a menu drawn on screen alone never gets
  keys.
- **Claude Code:** a PermissionRequest hook in the app's `--settings` file
  tells the server when Claude Code opens a permission dialog. **Codex:** a
  tool call in the rollout with no output yet.
- **Measured by:** conformance `claude.hook-before-prompt`,
  `claude.call-after-answer` and `codex.open-call`.
- **Provided by:** the app for anything it launches. A Claude Code started by
  hand needs your wrapper to pass `--settings "$ISWARM_SETTINGS"`.
- **Without it:** a hand-relaunched Claude Code is never auto-answered; the
  automation audit records `skipped_unconfirmed` with `no_call`.

### C12. Notifications

- **Needs from the agent:** C6, C7 and C9.
- **Pass:** asks, finished and limited each notify once, and never for an
  unverified agent.

### C13. Typed mail

- **Needs from the agent:** an input region the reader sees (C9), bracketed
  paste or a joined single line, a single Enter that submits, and the typed
  text shown on the input line.
- **Measured by:** unit tests; conformance `claude.typed-while-busy`. Codex
  has no daily typed-mail check.
- **Pass:** a multi-line message shows on the input line and is submitted
  once, never while a prompt or an unsent draft stands.
- **Without it:** mail goes only to the inbox, and the sender is told the
  agent is unverified.

### C14. Relayed text cannot forge the reader's marks

- **Needs from the agent:** nothing; every mark a new reader keys on must be
  broken in relayed text before it is typed.
- **Measured by:** `a_relayed_body_cannot_frame_an_input_box`,
  `a_relayed_body_cannot_end_a_codex_prompt` and their siblings.
- **Without it:** a message could forge a prompt or a false "finished".

### C15. MCP messaging

- **Needs from the agent:** a per-launch MCP server definition that adds to
  the user's own servers rather than replacing them, a bearer token read from
  the environment at connect time, a model that calls the tool when asked,
  and a tool approval that can be granted once.
- **Claude Code:** `--mcp-config <file>` (never `--strict-mcp-config`).
  **Codex:** two `-c mcp_servers.iswarm…` flags.
- **Measured by:** conformance `claude.mcp` and `codex.mcp`.
- **Provided by:** the app for sessions it starts while "Automatically add
  messaging to sessions I start" is on. A hand relaunch needs your wrapper's
  messaging line, or the server added to the agent's own settings
  ([operating.md](operating.md#other-clis-or-by-hand)). Approving the tool is the agent's own
  permission setting.
- **Without it:** a hand-relaunched agent cannot send, wait or list peers
  (receiving still works), and an unapproved tool fails at once in an
  unattended session.

### C16. Environment and the agent's own markers

- **Needs from the agent:** it inherits the terminal's environment, including
  the `ISWARM_*` variables, so a hand relaunch keeps its identity; and any
  marker it sets for its children does not switch off a nested instance's
  record.
- **Claude Code:** measured: when the app itself is started from inside a
  Claude Code session, it inherits that session's `CLAUDE_CODE_*` markers,
  and they turn transcript saving off in the Claude Code sessions it then
  starts. The app does not strip them. **Codex:** measured on 0.160.1: the
  commands it runs see the terminal's `ISWARM_*` variables, plus markers it
  sets (`CODEX_SANDBOX`, `CODEX_SANDBOX_NETWORK_DISABLED`, `CODEX_CI`,
  `CODEX_PERMISSION_PROFILE`, `CODEX_THREAD_ID`, `CODEX_SESSION_ID`). An app
  started under those markers still runs Codex sessions that start their own
  thread and keep their record. Not measured: whether an inherited
  `CODEX_SANDBOX` changes how a nested Codex sandboxes its own commands.
  **Antigravity:** measured on 1.2.12: the commands it runs see the
  terminal's `ISWARM_*` variables, its own markers (`ANTIGRAVITY_AGENT`,
  `ANTIGRAVITY_CONVERSATION_ID`, `ANTIGRAVITY_TRAJECTORY_ID`,
  `ANTIGRAVITY_PROJECT_ID` and six more) and `GIT_PAGER`. A stdio MCP server
  it starts sees the `ISWARM_*` variables and none of its markers. Not
  measured: what an inherited marker does to a nested Antigravity.
- **Provided by:** how you start the app. **Start InternSwarm from Finder,
  the Dock or a plain terminal, never from inside a Claude Code or Codex
  session.**
- **Without it:** no Claude Code transcript: no model, effort or tokens, no
  Waiting, no limit, and finished only from the screen.

### C17. Unreadable-screen health marker

- **Needs from the agent:** nothing beyond C9.
- **Pass:** a `?` shows on desktop and phone only while the agent is present
  and its screen shape has been lost for five minutes.

### C18. Re-checked on every CLI update

- **Needs:** recorded screens and records per renderer and record kind, tied
  to versions, and a daily run against the installed CLI. See
  [conformance-suite.md](conformance-suite.md).
- **Not checked daily yet:** Claude Code's conversation switches, the second
  renderer of each CLI, Codex interrupt, resize and typed
  mail, the turn-length fields, the account readers, and Linux (the daily job
  runs on macOS).
- **A CLI that updates itself** must have its version recorded on every run.
  Antigravity replaces its executable while it runs, and the next start is
  the new version: 1.2.11 became 1.3.3 on macOS and 1.2.12 became 1.3.3 on
  Linux, on 2026-10-10.

### P1. Model, effort, permission mode, tokens

- **Claude Code:** read from the transcript, subagents included. **Codex:**
  read from the rollout its own process holds open: model, effort,
  permission mode (sandbox and approval, left out at Codex's default) and
  tokens; blank while no Codex runs in the session.
- **Provided by:** the app, plus the linked `projects/` store for any wrapper
  that picks its own `CLAUDE_CONFIG_DIR`. Codex needs nothing extra.
- **Without it:** the status bar and usage panel stay blank.

### P2. Account label, quota, several accounts

- **Label:** your wrapper announces which account it launched, in
  `~/.iswarm/agent-account/<session>-<channel>`; the app checks the
  announcement against the terminal's live shell. Without one, the app infers
  the account for a single Claude Code account, and uses the session's
  `CODEX_HOME` (else `codex`) for Codex.
- **Quota:** for Claude Code, from the account's own usage cache, or live
  from Anthropic when "Fetch live usage" is on in the usage panel (on macOS
  that reads the login from the Keychain, which may ask for permission). For
  Codex, the newest reading the app saw that account's Codex write into its
  rollout, kept per account; it moves only on turns the app watched.
- **Without it:** "Account: unknown" when you have more than one Claude Code
  account, and a Codex reading credited to the session's account rather than
  the one you typed by hand.

## Setting up your machine

Most of this needs nothing from you: anything the app launches gets every
capability above. Setup matters when you **relaunch an agent by hand** in a
session's terminal, typically to switch to another account after a usage
limit, and when you run **several accounts** of one CLI.

- **Let your agent do it:** paste [agent-setup-prompt.md](agent-setup-prompt.md)
  into Claude Code or Codex on your machine. It asks a few questions, writes
  the wrappers, links the shared stores, and checks the result.
- **Do it yourself:** [multi-account.md](multi-account.md) explains every
  piece and why it is there.

What each piece buys, from the table above:

| Piece | Where | Capabilities |
|---|---|---|
| One config directory per account, logged in once | your home folder | P2 |
| Shared stores linked into each account (`projects/` for Claude Code; `sessions/` and friends for Codex) | your home folder | C4, P1, resuming across accounts |
| Account announcement in each wrapper | wrapper | P2 |
| `--mcp-config "$ISWARM_MCP_CONFIG"`, or Codex's two `-c` lines | wrapper | C15 on a hand relaunch |
| `--settings "$ISWARM_SETTINGS"` | wrapper (Claude Code) | C4, C11 on a hand relaunch |
| `--iresume` | wrapper | rejoining the session's conversation on a hand relaunch |
| Approve `iswarm_message` once | agent settings | C15 unattended |
| Start the app from Finder, never from inside Claude Code or Codex | how you launch | C16 |

## Antigravity, measured

Measured on 2026-10-10 on Linux arm64 (1.2.12, then 1.3.3 once it had
updated itself), run as the Gemini type with Command `agy`, against step 1 of
[Adding an agent](#adding-an-agent). Nothing below is built.

- **C3:** the process is named `agy`. Its executable path is not stable (see
  C3), and the session reads "no agent" whenever its output is quiet.
- **C4, C6:** it holds `~/.gemini/antigravity-cli/conversations/<id>.db` and
  `presence/<id>.lock` open, so the conversation can be found from the
  process, as Codex's rollout is. Each conversation also writes
  `brain/<id>/.system_generated/logs/transcript.jsonl`, one row per step
  (`step_index`, `source`, `type`, `status`, `created_at`). A turn is a
  `USER_INPUT` row followed by `PLANNER_RESPONSE` rows; no row marks its end.
  The model and the token counts are kept only in protobuf blobs in the
  conversation database, which has no published schema.
- **C5:** `agy --conversation <id>` resumes a conversation, and `/exit` prints
  that line.
- **C9:** the sign-in and folder-trust screens use the alternate screen;
  after them it draws inline on the main screen, each frame inside
  synchronized output, with the kitty keyboard protocol, no mouse reporting
  and no OSC sequences. The input region is a `>` line between two
  full-width rules, with `? for shortcuts` and the model below it.
- **C10:** three prompt kinds: folder trust (↑/↓, then Enter), a command
  permission (four numbered options) and an MCP tool approval (six). A bare
  digit answers the numbered ones, with no Enter.
- **C15:** no per-launch flag. `agy mcp add` writes one file for every
  session, and it expands neither `${VAR}` nor `$VAR`, so a saved HTTP entry
  would carry one session's token into all of them. With literal values the
  iswarm tool works. A stdio server it starts inherits the session's
  `ISWARM_*` variables, so a stdio-to-HTTP bridge started through `sh -c` is
  the one-time setup still to build.
- **P1, P2:** the footer shows the model and effort, and the banner the
  account and its quota plan. `ANTIGRAVITY_APP_DATA_DIR` does not select
  another account's folder, and no other way was found.
- **Not measured:** C7 (a limit needs an exhausted quota), C8 and C11.

## Adding an agent

Measure before building, and flip the tier last.

1. **Measure the agent.** Run it as an unverified type that adds no flags,
   with the CLI as its Command (Antigravity ran as Gemini). Then measure its
   foreground process name and executable path (C3); whether it writes a
   per-conversation record, and how it can be tied to one terminal (C4, C6,
   C7); every renderer, captured with
   `ISWARM_SCREEN_CAPTURE_DIR` (C9), and a capture of every prompt kind with
   the key that answers it (C10); how it takes an MCP server per launch
   (C15); and which environment variables it sets for its children (C16).
   The app has no capture viewer: replay a capture through
   `screen_capture::replay` in a test, or decode its `out` frames into one
   file and `cat` it in a terminal of the recorded size.
2. **Build the readers against the captures.** Add the agent to every
   per-agent `match` (C1), the process classifier (C3), the record reader
   (C4, C6, C7, C8), the screen reader and its fixtures (C9), the relayed-text
   sanitiser for its marks (C14), and the launch flags (C2, C15).
3. **Add a conformance file** (`scripts/conformance-<agent>.js`) covering C3,
   C4, C6, C7, C9 to C11, C13 and C15, and run it daily.
4. **Flip the tier**, in `AgentType::tier()` and `VERIFIED_AGENT_TYPES`
   together, only when everything above passes.
5. **Update this page:** its column in the table, and what its wrapper needs.
