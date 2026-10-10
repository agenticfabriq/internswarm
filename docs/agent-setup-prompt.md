# Set up InternSwarm with your coding agent

> Published from InternSwarm's code repository, which is not public yet, at 0e2bb50d. Anything about building from source applies there.

Copy everything below the line into Claude Code or Codex, running on the
machine where InternSwarm runs your agents. Your agent asks a few questions,
writes shell wrappers for each of your accounts, links the files those
accounts share, and checks the result. It backs up anything it replaces and
deletes nothing.

You need this only if you relaunch agents by hand in a session's terminal
(for example, switching accounts after a usage limit) or run several accounts
of one CLI. Sessions the app starts need none of it. See
[agent-onboarding.md](agent-onboarding.md) for what each piece buys, and for
what the app does not do yet for each agent.

---

You are setting up shell wrappers so that InternSwarm, an app that runs coding
agents in terminals it owns, keeps working when I start an agent **by hand**
in one of its session terminals: messaging between agents, conversation
tracking, the account name and usage in its panel, and rejoining the
conversation after switching accounts.

The full reference, with working code and the reason for every line, is
InternSwarm's `docs/multi-account.md`:
https://raw.githubusercontent.com/agenticfabriq/internswarm/main/docs/multi-account.md
Read it before writing anything. If you cannot fetch it, ask me to paste it.
Follow it where it gives code; this message lists what must be true, and the
traps that have broken real setups.

## 1. Ask me first

Ask these in one message, then wait for my answers:

1. Which CLIs do I use: Claude Code, Codex, or both?
2. For each, what are my accounts, and what label should each wrapper have?
   (Labels like `claude-j1`, `claude-j2`, `codex-j1`; each label becomes a
   command and a config directory, `~/.claude-j1` or `~/.codex-j1`.)
   Are any of them API-key accounts rather than logins?
3. Which account should a bare `claude` or `codex` mean?
4. Which file should the wrappers go in? (Default: `~/.zshrc`, or a file it
   sources.) The wrappers are written for zsh, macOS's default shell. If my
   shell is not zsh, tell me so and stop: the reference does not cover
   other shells.
5. Is the Codex or ChatGPT desktop app installed? It shares `~/.codex`.

Then show me the plan, the files you will create or change and what you will
back up, and wait for my go-ahead.

## 2. Build these

**Config directories and logins.** One config directory per account
(`CLAUDE_CONFIG_DIR` for Claude Code, `CODEX_HOME` for Codex). You cannot log
in for me: give me the exact commands to run once per account.

- Claude Code login: `CLAUDE_CONFIG_DIR=~/.claude-j1 claude`, then `/login`.
  An API-key account needs no directory: its wrapper sets
  `ANTHROPIC_API_KEY` from a private file (`chmod 600`, not in git).
- Codex login: `CODEX_HOME=~/.codex-j1 codex login`. A Codex API-key
  account gets a home of its own, logged in once with the key on stdin:
  `printenv OPENAI_API_KEY | CODEX_HOME=~/.codex-k1 codex login --with-api-key`.
  Never put a key on a command line.

**Shared stores, by symlink, into the default directory.** Use the
reference's idempotent scripts, which back up anything real they replace.

- Claude Code: `projects/`, `history.jsonl`, `plugins/`, `commands/`,
  `agents/`, `skills/`, `settings.json`, `CLAUDE.md`. `projects/` is the one
  InternSwarm depends on: it reads `~/.claude/projects` and cannot see which
  directory a wrapper chose. Never link `.claude.json`, which holds the login.
- Codex: `sessions/`, `archived_sessions/`, `shell_snapshots/`,
  `history.jsonl`, `plugins/`, `config.toml`. Never link `auth.json` or any
  `*.sqlite`. **Before linking:** ask me to quit every Codex process,
  including the desktop app and any editor's `codex app-server`, and check
  with `pgrep -il 'codex|chatgpt'`, which must print nothing. If an account
  already has sessions, merge them into `~/.codex` first with the
  reference's merge script; the linking script backs real folders up rather
  than merging them.

**The account announcement.** A helper that every wrapper runs the agent
through, writing `<label> <pid>` to
`~/.iswarm/agent-account/$ISWARM_SESSION_ID-$ISWARM_CHANNEL_ID` and removing
it afterwards. Copy the reference's `iswarm_with_account`; these properties
matter and each one fixed a real bug:

- It never overwrites a live announcement from another process (a one-shot
  `claude --version` in the same terminal would otherwise erase the running
  agent's).
- It removes the file only if it still holds exactly the line it wrote.
- A nested call (a wrapper run from inside another) leaves the file alone.
- Outside InternSwarm (`ISWARM_SESSION_ID` unset) it does nothing.
- If the helper's file fails to load, the wrappers must still run the agent:
  define a fallback that runs the command without announcing, and print once,
  at load, that announcements are off.

**One wrapper per account.**

- Claude Code: `env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j1"
  claude "$@"`. Use `env`, which finds the binary on `PATH`, not `command`
  inside `env` (that fails with exit 127 on most Linux). Unsetting the key
  keeps a key exported by my profile from silently switching a login account
  to per-token billing. An API-key account is a separate wrapper that sets
  the key explicitly.
- Codex: `env -u OPENAI_API_KEY CODEX_HOME="$HOME/.codex-j1" codex "$@"`.
  `OPENAI_API_KEY` is unset as a precaution.

**Inside InternSwarm only, the app's own launch flags.** Each guarded on its
variable, so nothing is added in an ordinary terminal:

- Claude Code: `--mcp-config "$ISWARM_MCP_CONFIG"` (messaging) and
  `--settings "$ISWARM_SETTINGS"` (conversation tracking, and what lets
  auto-answer work). Never `--strict-mcp-config`, which drops my other MCP
  servers. Never edit my `settings.json`; `--settings` adds to it.
- Codex: the two lines
  `-c "mcp_servers.iswarm.url=\"$ISWARM_MCP_URL\""` and
  `-c 'mcp_servers.iswarm.bearer_token_env_var="ISWARM_CHANNEL_TOKEN"'`.
  The inner double quotes are required. Do not install Codex hooks: a new
  hook makes Codex stop at a trust prompt before it starts.

**No flags in front of a Claude Code subcommand.** `claude-j1 stop <id>`,
`claude-j1 agents` or `claude-j1 mcp list` must run the subcommand. With the
flags above in front, a subcommand that takes an argument is not dispatched,
so `stop <id>` becomes a new, billed conversation handed the word "stop";
and `--mcp-config` reads the next word as a config file, so `agents` fails.
Appending the flags instead does not work either: `doctor` and `mcp list`
reject them as unknown options. When
the first argument is a bare word (not starting with `-`), ask the installed
binary: `claude <word> --help` prints the top-level usage line,
`Usage: claude [options] [command] [prompt]`, for anything that is not a
subcommand. If it is a subcommand, run it with the account's environment, without the flags, and
with `ISWARM_CHANNEL_TOKEN` unset, for every subcommand rather than by name.
`agents` starts sessions of its own, and if they inherit the terminal's
environment (not measured), the token would let each report itself as this
session's conversation and message as this session, even through a persisted
MCP entry. (So inside a session, `claude-j1 mcp list` may show a persisted
iswarm entry as failing; that is expected.) The complete
`claude-j1` under "Agent-to-agent messaging for hand-launched agents" in the
reference shows how. If you cannot tell, treat it as a subcommand: a session
without messaging is visible and free, a billed turn is not. Cache the answer
per CLI version if the 70 ms matters. Codex does not need this; `-c` in front
of its subcommands is fine.

**`--iresume`, as the first argument.** It relaunches on this session's own
conversation.

- Claude Code: copy the reference's `_iswarm_rejoin_args`. It asks the app
  (`GET <server>/api/agent/conversation`, with the channel token sent on
  stdin, never on the command line), and also check that the answer's
  `session_id` is `$ISWARM_SESSION_ID`, ignoring it otherwise. Then
  `--resume <conversation>` if its transcript exists, else
  `--resume $ISWARM_SESSION_ID`, else `--session-id $ISWARM_SESSION_ID`,
  saying on stderr which one it chose.
- Codex: ask the app exactly as for Claude Code (the channel token on
  stdin, the answer only if its `session_id` is `$ISWARM_SESSION_ID`), and
  use the id only if `rollout-*-<id>.jsonl` exists under this account's
  `sessions/`; then `codex resume <id>`. Otherwise run `codex resume` (Codex's
  picker) and say why on stderr. Never scan the rollout pool yourself: it
  holds Claude Code's Codex plugin sessions and the desktop app's too. If I
  typed anything after `--iresume`, do not ask the app: run
  `codex resume <what I typed>` as typed, and say so on stderr.
- Check for `--iresume` before adding any flag, since it reads the first
  argument.

**Bare names.** `claude() { claude-j1 "$@"; }` and `codex() { codex-j1 "$@"; }`
(my chosen defaults), so the "resume with `claude --resume <id>`" hint the
CLI prints on exit goes through the wrapper too.

**Approve the messaging tool once.** For Claude Code, add
`"mcp__iswarm__iswarm_message"` to `permissions.allow` in
`~/.claude/settings.json` (shared by every account through the link above).
For Codex, tell me to approve the `iswarm` tool the first time it asks.

## 3. Traps

- The InternSwarm Command field is one word, and the app adds its own flags
  after it. Never put anything of your own after `"$@"`, and never reject
  unknown arguments.
- In zsh, do not use `path` or `status` as local variable names: `path` is
  tied to `$PATH`, and `status` is read-only.
- In zsh, an array built from empty output has one empty element, which
  becomes a real empty argument. Build optional flag lists so that "none"
  means no elements at all.
- `[[` is not POSIX: scripts that use it start with `#!/usr/bin/env bash`.
- Never start InternSwarm or its server from inside a Claude Code session:
  it inherits markers that turn transcript saving off in the agents it starts.

## 4. Check, then report

Check each of these, and show me the output:

1. `type claude-j1` (and each wrapper) shows a function.
2. In an ordinary terminal, `claude-j1 --version` and `codex-j1 --version`
   work.
3. The subcommand guard, with InternSwarm's variables faked so the flags
   would be added:
   `ISWARM_MCP_CONFIG=/dev/null ISWARM_SETTINGS=/dev/null ISWARM_CHANNEL_TOKEN=x claude-j1 mcp list`
   lists my MCP servers instead of failing on an unknown option or an invalid
   config. Then read the wrapper and confirm its subcommand branch unsets
   `ISWARM_CHANNEL_TOKEN`.
4. `ls -la ~/.claude-j1/projects` (and each shared item, for each account)
   points at the default directory.
5. `CODEX_HOME=~/.codex-j1 codex login status` for each Codex home shows the
   login I expect.
6. Each wrapper file passes `zsh -n`.

Then tell me to do these inside an InternSwarm session terminal, since you
cannot:

7. Start `claude-j1` (or `codex-j1`), ask it to "list your iswarm peers", and
   check that it answers.
8. Open the usage panel and check that it names the account.
9. Quit the agent, type `claude-j2 --iresume`, and check that it says which
   conversation it is resuming.

Finish with a short report: what you created or changed, every backup you
made, the login commands I still need to run, and anything you could not do.
