# Multiple accounts

> Published from InternSwarm's code repository, which is not public yet, at 0e2bb50d. Anything about building from source applies there.

Switch accounts the moment one hits its usage limit, without losing
agent-to-agent messaging or per-account usage numbers along the way.

The scenario this document is for: you run `claude-j1`, it hits its usage
limit mid-session, you quit the agent, and — in the same session terminal,
without touching the New Session dialog — **type `claude-j2` by hand**. The
app composed the *first* launch line; it had no part in the second.
Everything below exists to make that hand-typed relaunch work as well as the
app-composed one did.

## What you already have without any of this

- **App-launched sessions already get agent-to-agent messaging.** Pressing
  Start agent or Restart agent composes the launch line and appends the MCP
  flags itself — nothing here is needed for that first launch.
- **Single-account usage attribution already works.** If you only have one
  `~/.claude-*` directory, the app finds it and reads usage from it with no
  announcement required. This document is for people running more than one
  account.
- **Receiving a message never needs any setup.** Delivery is the server
  typing directly into the recipient's terminal — it does not go through
  MCP, so it does not matter what flags the recipient's launch line carries.
  Only *sending* a message, *waiting* for one, and *listing peers* are MCP
  tool calls, which is the entire reason the rest of this document exists.
- **A recipient is addressed by name, whichever account it runs as.** Every
  session has a name — `dotfiles#2` by default, editable, unique across the
  whole server — and `peers` lists the name to use for each session an agent
  can message. A whole session id still resolves; unlike a name, it does not
  change when somebody renames the session. Nothing about addressing depends
  on which config directory the agent picked, so a hand-typed `claude-j2` is
  reachable at exactly the same name as the `claude-j1` it replaced. Another
  *person* is addressed by their account instead — `alice@example.com`, which
  reaches every device of theirs you have verified — and that is unaffected by
  accounts in this document's sense, which are the CLI logins one human runs.

The rest of this is for the multi-account power user: comfortable editing
dotfiles, trading a few minutes of setup for messaging and usage tracking
that survive a hand-typed relaunch. To have your own coding agent do the
setup, paste `docs/agent-setup-prompt.md` from the InternSwarm repository into
it; this document is the reference it follows. Every wrapper function below goes in
your shell profile — `~/.zshrc`, on macOS's default shell — so it exists in
every terminal, not just ones iSwarm opens.

## Creating the accounts

Skip this if you already have several `~/.claude-*` directories; it is a
one-time setup, and the rest of this document starts from there.

Claude Code decides which account it is in two ways, and a multi-account setup
uses both:

- **`ANTHROPIC_API_KEY`.** When set, it is used and any stored OAuth login is
  ignored. An "API-key account" is therefore just running the CLI with a
  particular key exported for that one invocation — no files, no login.
- **`CLAUDE_CONFIG_DIR`.** The OAuth login lives in `<config dir>/.claude.json`
  (default `~/.claude`). Two OAuth accounts cannot share one config dir —
  logging into the second overwrites the first, which is the log-out/log-in
  dance this setup exists to end. Give each account its own directory and both
  stay logged in permanently.

Both gotchas that follow are why the wrappers below look the way they do: an
exported key **overrides** OAuth, so an OAuth account must run with the key
unset (`env -u ANTHROPIC_API_KEY …`); and a separate config dir isolates
*everything*, including history and plugins, which you do not want.

### What is shared, and what must not be

Symlink these from each `~/.claude-<name>` back to the default `~/.claude`:

- `projects/` — transcripts. **This one is load-bearing for InternSwarm**: the
  app reads `~/.claude/projects` and cannot see which config dir a wrapper
  chose, so an account writing elsewhere gets no model, effort, or tokens
  attributed. It is also what lets a conversation started under one account be
  resumed under another after a credit switch.
- `history.jsonl`, `plugins/`, `commands/`, `agents/`, `skills/`,
  `settings.json`, `CLAUDE.md` — history, plugins, and your customizations.

Leave these alone in each config dir — symlinking them defeats the purpose:

- `.claude.json` — the OAuth login and account identity
- session caches, telemetry, and policy-limit state

An idempotent script that creates the directories and wires the shared pieces,
backing up anything real it would replace:

```sh
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("$HOME/.claude-j1" "$HOME/.claude-j2" "$HOME/.claude-j3")

DEFAULT="$HOME/.claude"
SHARED_DIRS=(projects plugins commands agents skills)
SHARED_FILES=(history.jsonl settings.json CLAUDE.md)

for d in "${SHARED_DIRS[@]}"; do mkdir -p "$DEFAULT/$d"; done
for f in "${SHARED_FILES[@]}"; do
  [[ -e "$DEFAULT/$f" ]] && continue
  case "$f" in
    # An empty file is not valid JSON, and this one gets symlinked into every
    # account — seed it so a fresh machine starts parseable.
    *.json) printf '{}\n' > "$DEFAULT/$f" ;;
    *)      : > "$DEFAULT/$f" ;;
  esac
done

link() {  # link <source> into <target>, backing up anything real at target
  local source="$1" target="$2"
  if [[ -L "$target" ]]; then
    unlink "$target"
  elif [[ -e "$target" ]]; then
    mv "$target" "$target.backup.$(date +%Y%m%d%H%M%S)"
    echo "backed up existing $target"
  fi
  ln -s "$source" "$target"
}

for dir in "${ACCOUNTS[@]}"; do
  mkdir -p "$dir"
  for d in "${SHARED_DIRS[@]}";  do link "$DEFAULT/$d" "$dir/$d"; done
  for f in "${SHARED_FILES[@]}"; do link "$DEFAULT/$f" "$dir/$f"; done
done
```

Then log each OAuth account in once — `CLAUDE_CONFIG_DIR="$HOME/.claude-j1"
claude`, `/login`, exit, and repeat per directory. Because the logins live in
separate `.claude.json` files, they persist side by side and you never log in
again. Keep any API keys in a private file that is **not** tracked in git and
is `chmod 600`.

## Multiple Claude accounts

Claude Code picks its account from `CLAUDE_CONFIG_DIR`. Write one wrapper
function per account:

```sh
claude-j1() {
  env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j1" claude "$@"
}

claude-j2() {
  env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j2" claude "$@"
}
```

Bare `claude` after `env` is safe even though the function and the binary
share a name: `env` is an external program that resolves `claude` through
`PATH` and knows nothing of shell functions, so the wrapper cannot call
itself. (Do not write `command claude` there — inside `env` that word must be
a binary, which macOS ships and most Linux does not, so it dies with exit 127
on a Linux server.) `"$@"` forwards whatever arrives after the command word
untouched; see [What the app appends to your
command](#what-the-app-appends-to-your-command) below for what that actually
is.

`env -u ANTHROPIC_API_KEY` because the CLI prefers an API key over the config
directory's login: a key exported anywhere in your profile would silently
switch every launch from the subscription you selected to per-token API
billing. Unsetting it here makes the wrapper mean what it says. If per-token
billing is what you want, make that its own wrapper that sets the key
explicitly.

Set **Command** to `claude-j1` in a session's **Settings… → Agent** tab, or
type it by hand into a running terminal to switch accounts mid-session.

### Making it the whole repository's default

Setting it per session means setting it again on every new one. A repository
can carry the answer instead: **Default command** in the repository's
**Settings… → Agent** tab, beside **Default agent for new sessions**. New
sessions created in that repository start with it, and the New Session
dialog shows it as what an empty Command field will run — so the account a
session bills is visible before it is created.

Two things it deliberately does not do. It applies only to sessions running
the repository's **default agent**, because a wrapper is written for one CLI
and `claude-j1` cannot launch Codex; a Codex session in a Claude-defaulting
repository gets `codex`. And it does not touch **existing** sessions, which
keep the command they were created with — changing a repository default
never re-points a running agent.

Changing the default agent clears the default command with it, for the same
reason: a wrapper for the old agent is not a command for the new one.

## Multiple Codex accounts

Same idea, a different variable: Codex picks its account from `CODEX_HOME`.

```sh
codex-j1() {
  CODEX_HOME="$HOME/.codex-j1" command codex "$@"
}

codex-w1() {
  CODEX_HOME="$HOME/.codex-w1" command codex "$@"
}
```

### What Codex shares, and what must not be

**This part is load-bearing**, in the same way `projects/` is on the Claude
side. `CODEX_HOME` isolates *everything*, so without the symlinks below each
account keeps its own session history — and a session started on `codex-j1`
becomes unreachable from `codex-w1`, which is the entire point of the second
account.

Symlink these from each `~/.codex-<name>` back to the default `~/.codex`:

- `sessions/`, `archived_sessions/` — the rollout files. Restart agent and
  `--iresume` both resume a conversation from here, and this is what lets a
  session started before a credit switch continue after it. The app looks in
  the `CODEX_HOME` the Environment tab sets, if any, else in
  `~/.codex`; it cannot see a home a shell wrapper picks, so such a home must
  link `sessions/` here, or Restart starts fresh there, and says so.
- `shell_snapshots/` — captured shell state, referenced by rollouts.
- `history.jsonl` — the input history, so what you typed follows the switch.
  Codex appends in place and its `message-history` writer takes a lock, so two
  accounts sharing one file is safe.
- `plugins/` — installed plugin content, worth sharing because it is large and
  identical.
- `config.toml` — model, effort, marketplaces, enabled plugins. Sharing it
  assumes your accounts do not need to differ here. If one does — a work
  account with its own MCP servers, say — give that account a real
  `config.toml` instead of the link, or layer a `$CODEX_HOME/<name>.config.toml`
  and pass `-p <name>`; profiles add to the base rather than replacing it. Note
  that `codex mcp add` writes through the symlink into the shared file, so an
  MCP server added under one account appears in both.

Leave these alone in each account home — sharing them defeats the purpose or
corrupts state:

- `auth.json` — the login and account identity, the Codex counterpart of
  `.claude.json`
- `state_5.sqlite`, `thread_history_1.sqlite`, `logs_*`, `goals_*`,
  `memories_*` and their `-wal`/`-shm` siblings — per-account SQLite, unsafe to
  reach through two paths at once
- `session_index.jsonl`, `installation_id`

`~/.codex` is also the ChatGPT desktop app's home. It writes its own sessions
there (`originator: "Codex Desktop"`), which is why sharing into it rather than
into a fourth directory is the simpler arrangement: one pool, not two.

That overlap is deliberate and useful — start a conversation in the desktop app
and continue it from the CLI, or the reverse. It does mean the pool holds
conversations other programs wrote, some in your project's own folder: Claude
Code's Codex plugin (`/codex:review` and friends, `originator: "Claude Code"`;
in one measured pool, 385 of the newest 400 rollouts were the plugin's) and the
desktop app (`originator: "Codex Desktop"`). None of them is ever taken for
your session's conversation: the app records the one your session's own
`codex` process holds open, and `--iresume` asks the app for that. To continue
a desktop conversation instead, use Codex's own `codex resume` picker.

**Quit every Codex process before running either script below** — the desktop
app, any `codex` TUI, and any `codex app-server` daemon an editor plugin left
running. That daemon is the one that catches people out: it is long-lived, it
holds rollout files open for writing, and nothing about it is visible in a
terminal.

```sh
pgrep -il 'codex|chatgpt'   # must come back empty
```

Not `pgrep -x codex`: `-x` matches the exact name, and the desktop app's
processes are `Codex` and `ChatGPT`, so the check would pass while the very
process the sentence names first is still running.

Relinking a tree a live process is appending to is the one way this loses data:
the copy takes a stale prefix, the original is moved aside, and the process goes
on writing to the old inode.

**If the accounts already have history, merge it into `~/.codex` first.** The
script below *backs up* anything real it would replace rather than merging it,
so running it on populated homes leaves an empty hub and your sessions orphaned
in `sessions.backup.<stamp>` — technically recoverable, silently wrong. The same
applies to `plugins/`: an account that installed one the default home lacks
would lose it to a backup, so the merge covers it too. Session ids are UUIDs, so
the session trees cannot collide on a filename:

```bash
#!/usr/bin/env bash
set -euo pipefail
# bash, not sh: `[[` below is not POSIX, and under `sh` every test exits 127,
# nothing is copied, and the wiring script then backs the real trees away.
for home in "$HOME/.codex-j1" "$HOME/.codex-w1"; do
  for d in sessions archived_sessions shell_snapshots plugins; do
    [[ -d "$home/$d" && ! -L "$home/$d" ]] || continue
    mkdir -p "$HOME/.codex/$d"
    cp -Rp "$home/$d"/. "$HOME/.codex/$d"/     # cp, not mv: date dirs collide
  done
  # Append rather than replace, oldest first.
  [[ -f "$home/history.jsonl" && ! -L "$home/history.jsonl" ]] &&
    cat "$home/history.jsonl" >> "$HOME/.codex/history.jsonl"
done
```

Then the wiring itself, idempotent, backing up anything real it would replace:

```sh
#!/usr/bin/env bash
set -euo pipefail

ACCOUNTS=("$HOME/.codex-j1" "$HOME/.codex-w1")

DEFAULT="$HOME/.codex"
SHARED_DIRS=(sessions archived_sessions shell_snapshots plugins)
SHARED_FILES=(history.jsonl config.toml)

for d in "${SHARED_DIRS[@]}"; do mkdir -p "$DEFAULT/$d"; done
for f in "${SHARED_FILES[@]}"; do [[ -e "$DEFAULT/$f" ]] || : > "$DEFAULT/$f"; done

link() {  # link <source> into <target>, backing up anything real at target
  local source="$1" target="$2"
  if [[ -L "$target" ]]; then
    unlink "$target"
  elif [[ -e "$target" ]]; then
    mv "$target" "$target.backup.$(date +%Y%m%d%H%M%S)"
    echo "backed up existing $target"
  fi
  ln -s "$source" "$target"
}

for dir in "${ACCOUNTS[@]}"; do
  mkdir -p "$dir"
  for d in "${SHARED_DIRS[@]}";  do link "$DEFAULT/$d" "$dir/$d"; done
  for f in "${SHARED_FILES[@]}"; do link "$DEFAULT/$f" "$dir/$f"; done
done
```

Then log each account in once with `CODEX_HOME="$HOME/.codex-j1" codex login`,
repeating per directory; the logins live in separate `auth.json` files and
persist side by side. An API-key account is a home of its own too, logged in
once with the key on stdin, never on the command line:

```sh
printenv OPENAI_API_KEY | CODEX_HOME="$HOME/.codex-k1" codex login --with-api-key
```

`codex login status` under each `CODEX_HOME` shows which kind each home holds.

### What the app appends to your command

A command set in **Settings… → Agent** is typed into the session's shell,
not exec'd — that is why a shell function resolves at all, and why it must
be a single word with no arguments of your own: the app appends flags after
it.

- **Claude**: `--session-id <id>` for a session with no transcript yet, or
  `--resume <id>` once one exists — never both, and the CLI refuses
  `--session-id` for an id it already holds a transcript for. Then, when
  **Automatically add messaging to sessions I start** is on (the default),
  `--mcp-config <path>`. Always, `--settings <path>` — the app-owned settings
  file that registers a SessionStart hook so the usage panel follows a
  `/resume` or `/clear` (see [Conversation tracking](#conversation-tracking)
  below). `--settings` loads *additional* settings that merge with your own;
  your `settings.json` is never edited.
- **Codex**: no session-binding flag. Codex does accept `codex resume
  <SESSION_ID>`, but the app does not put it on the launch line, because
  nothing tells the app which Codex session a channel is in: the Claude
  binding comes from a `SessionStart` hook reporting itself, and a hook the
  app installs is not something Codex will run unattended (see [Codex
  session tracking](#codex-session-tracking) below). The wrapper's
  `--iresume` resolves the id instead. When **Automatically add messaging to
  sessions I start** is on, the app appends the two
  `-c mcp_servers.iswarm...` flags described under [Agent-to-agent messaging
  for hand-launched
  agents](#agent-to-agent-messaging-for-hand-launched-agents) below.

None of that breaks a wrapper that just forwards `"$@"` in place: Claude
accepts flags in any position, so reordering them costs nothing.

**Except in front of a subcommand.** A wrapper that adds `--mcp-config` or
`--settings` itself, as the ones under [Agent-to-agent messaging for
hand-launched agents](#agent-to-agent-messaging-for-hand-launched-agents) do,
puts them in front of whatever you type. Measured on 2.1.278, two things go
wrong. A subcommand that takes an argument is not dispatched when any option
is in front of it, so `claude-j1 stop <id>` becomes a **new conversation**
handed the words "stop <id>", a billed turn; the same goes for `attach`,
`logs` and the others that name a session. And `--mcp-config` takes several
values, so the word after it is read as a config file: `claude-j1 agents`
fails with "Invalid MCP configuration".
Appending the flags instead does not help: `claude doctor` and `claude mcp
list` reject them as unknown options. So when the first argument is a bare
word, ask the installed binary whether it is a subcommand —
`claude <word> --help` prints the top-level `Usage: claude [options]
[command] [prompt]` line for anything that is not — and run a subcommand with
the account's environment, none of the flags, and without the terminal's
`ISWARM_CHANNEL_TOKEN`. `agents` starts sessions of its own, and if they
inherit this terminal's environment (not measured), the token would let each
report itself as this session's conversation and message as this session,
through the flags or through an iswarm entry in your persisted MCP config. The
token is withheld from every subcommand rather than from `agents` by name, so
one added upstream is covered too. One visible cost: inside a session,
`claude-j1 mcp list` may show a persisted iswarm entry as failing, because it
gets no token. That is the wrapper, not a broken setup; check a running agent
with `/mcp` instead (see [Verify](#verify)). When the answer is unclear,
treat the word as a subcommand: a session started without messaging is
visible and free, a billed turn is neither. Codex has no such problem: its
`-c` flags work in front of every subcommand. What
actually breaks a wrapper is hardcoding something of your own *after*
`"$@"` — `claude "$@" -p "prompt"` runs `-p` after whatever the app
appended, and `-p` switches Claude out of the interactive mode iSwarm's
terminal expects into a one-shot print-and-exit run — or validating `"$@"`
at all: rejecting an unrecognized flag, or a nonzero argument count, rejects
the exact flags the app is trying to hand you.

## Making the usage panel name your account

iSwarm's usage panel can show which account a session is running under —
`claude-j1`, `claude-j3`, and so on — along with that account's usage. The
numbers come from the account's own usage cache, which Claude Code refreshes
when you run `/usage`; turn on **Fetch live usage** in the panel to have the
app ask Anthropic instead (on macOS this reads the account's login from the
Keychain, which may ask for permission).

It cannot work this out on its own. The agent runs inside a server-owned
PTY, and **macOS does not let a process read a child's environment**, so the
app cannot see which account the CLI selected. The launching command has to
say so.

### The announcement

A wrapper writes one file when it launches an agent, and removes it when the
agent exits:

```
path     ~/.iswarm/agent-account/$ISWARM_SESSION_ID-$ISWARM_CHANNEL_ID
content  <label> <pid>
```

- **`ISWARM_SESSION_ID` and `ISWARM_CHANNEL_ID`** are exported into every
  session shell. Both are needed: one session has several channels, so the
  session id alone lets two agents overwrite each other, and every session's
  agent channel is *named* `agent`, so the channel id alone gives the whole
  machine one shared file.
- **`<pid>`** is the announcing shell's. The app checks it against the
  channel's live shell, so a file left behind by a killed shell is ignored
  rather than reported as the running account. An unknown account is a
  supported state; a confidently wrong one is not.
- **`<label>` must match a config directory.** Announcing `claude-j1` makes
  the app read usage from `~/.claude-j1`. A label with no matching directory
  shows a name and no numbers — which looks like a bug and is really a
  naming mistake.

### A wrapper

This extends the plain per-account wrapper from [Multiple Claude
accounts](#multiple-claude-accounts) — swap that definition for this one
once you also want the panel to name the account:

```sh
_iswarm_account_signal_path() {
  [ -n "$ISWARM_SESSION_ID" ] || return 1
  local key="$ISWARM_SESSION_ID"
  [ -n "$ISWARM_CHANNEL_ID" ] && key="$ISWARM_SESSION_ID-$ISWARM_CHANNEL_ID"
  printf '%s/.iswarm/agent-account/%s\n' "$HOME" "$key"
}

# $1 = label, e.g. claude-j1. The rest is the command to run.
iswarm_with_account() {
  local label="$1" signal owner ours="" exit_status
  shift

  # Nested inside another announced command in THIS shell. Several shells in
  # one channel resolve the SAME path, and in zsh a `( … )` subshell keeps the
  # parent's `$$`, so ownership cannot be told apart by file contents here.
  # Depth is the discriminator: the outer call owns the announcement and the
  # inner one leaves it alone entirely.
  if [ -n "${_ISWARM_ACCOUNT_ACTIVE-}" ]; then
    "$@"
    return $?
  fi
  # `local` is dynamically scoped in zsh, so this is visible to everything the
  # call runs and is restored on return.
  local _ISWARM_ACCOUNT_ACTIVE=1

  if signal="$(_iswarm_account_signal_path)"; then
    mkdir -p "$(dirname "$signal")" 2>/dev/null
    # Never displace a LIVE announcement from another shell. Without this, a
    # wrapped one-shot run inside a session terminal (`claude --version`, a
    # script) overwrites the running agent's announcement — the panel then
    # loses the account name and quota while model and tokens keep working,
    # which reads like an app bug and is really this line. A recorded pid that
    # is gone is fair game: a killed agent should not hold the channel.
    owner="$(cut -d' ' -f2 "$signal" 2>/dev/null)"
    if [ -z "$owner" ] || [ "$owner" = "$$" ] || ! kill -0 "$owner" 2>/dev/null; then
      printf '%s %s\n' "$label" "$$" >| "$signal" 2>/dev/null
    fi
    # What we actually own — read back rather than assumed, since the write
    # above may have declined.
    [ "$(cat "$signal" 2>/dev/null)" = "$label $$" ] && ours="$label $$"
  fi

  "$@"
  exit_status=$?

  # An idle channel must stop claiming an account — but clear ONLY our own
  # line. An unconditional `rm` is the same bug from the other end. Compare the
  # whole line, not the pid: a nested command shares `$$` and differs only by
  # label, so a pid-only check still deletes the agent's announcement.
  if [ -n "$ours" ] && [ "$(cat "$signal" 2>/dev/null)" = "$ours" ]; then
    rm -f "$signal" 2>/dev/null
  fi
  return $exit_status
}

# Outside InternSwarm, ISWARM_SESSION_ID is unset and this is a no-op.
claude-j1() {
  iswarm_with_account claude-j1 \
    env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j1" claude "$@"
}
```

Then set **Command** to `claude-j1` in a session's **Settings… → Agent** tab.

A Codex account announces the same way. Codex's own home goes in `CODEX_HOME`,
and the label names it:

```sh
codex-j1() {
  iswarm_with_account codex-j1 \
    env -u OPENAI_API_KEY CODEX_HOME="$HOME/.codex-j1" codex "$@"
}
```

`OPENAI_API_KEY` is unset as a precaution, for the same reason as
`ANTHROPIC_API_KEY` above; whether Codex prefers an exported key over its
login has not been measured. If the file defining `iswarm_with_account` might
fail to load (a dotfiles path that does not resolve on another machine), keep
the wrappers working without it, and say so once:

```zsh
(( $+functions[iswarm_with_account] )) || {
  print -u2 "iswarm: the account helper did not load; announcements are off"
  iswarm_with_account() { shift; "$@"; }
}
```

`zsh` note: avoid the local names `path` and `status` inside these
functions. `path` is tied to `$PATH` — assigning a string to it breaks
command lookup for the rest of the function — and `status` is a read-only
alias for `?`.

### Conversation tracking

The app knows which Claude conversation a session is in, and keeps up when
that changes. It installs a **SessionStart hook** — a small script Claude runs
at the start of every session, including after `/resume`, `/clear`,
`/compact`, and `/fork` — that reports the current conversation id back to the
server. So the usage panel and status badge follow the conversation you are
actually in, not the one the app first launched.

The hook rides a managed settings file the app hands Claude with `--settings`,
which *adds* settings and **never edits your `settings.json`** (so a symlinked
dotfile is safe). An **app-launched** session gets it automatically. A
**hand-typed** relaunch gets it too **when your wrapper re-passes it** — the
per-account wrappers in this repo already do, via `$ISWARM_SETTINGS`. If you
maintain your own wrapper, add one line before running the binary, right
beside the messaging line:

```zsh
[[ -n $ISWARM_SETTINGS ]] && set -- --settings "$ISWARM_SETTINGS" "$@"
```

The hook is inert outside InternSwarm: with no channel token in the
environment it exits at once and makes no network call, so a plain `claude` in
an ordinary terminal is never touched.

The same file carries a second hook, **PermissionRequest**, which tells the
server when Claude opens a permission dialog. Auto-answer relies on it: it
presses keys for Claude only when that hook has reported the dialog on screen,
because Claude's transcript cannot be relied on to show the pending tool call
before the dialog is answered. So a Claude launched **without** the managed settings (a
wrapper that does not re-pass `$ISWARM_SETTINGS`) is never auto-answered, and
the automation audit records a `skipped_unconfirmed` row naming `no_call` for
a rule that would have fired. Sessions started before this hook existed carry
the old settings file until they are relaunched. Like SessionStart, it is inert
outside InternSwarm, and it only reports the dialog: it never answers one
itself.

With the hook in place, a hand relaunch is tracked **whatever conversation it
lands on** — a fresh one, or one you resume with `claude-j1 --resume <id>`.
You no longer need `--iresume` for the panel to keep working; it is now only a
convenience for continuing *this session's own* conversation instead of
starting fresh.

### Codex session tracking

Codex works the other way round: nothing reports itself, so the app
**observes** which conversation a session is in instead.

Codex has the same hook events as Claude (`SessionStart` with `startup`,
`resume`, `clear`, `compact`), and its payload is richer. It is still not
usable as an install step, because a hook the app writes cannot run
unattended — verified against Codex 0.151.0 on 2026-08-31, a new or changed
`hooks.json` makes Codex stop at an interactive prompt before the session
starts:

> Hooks need review — 3 hooks are new or changed. Hooks can run outside the
> sandbox after you trust them.

Trust is pinned to a hash of each entry and recorded in `config.toml`, so
every update to a shipped hook would prompt again, and the prompt blocks
startup inside the PTY the app is driving. Hence the observation.

**Where the id comes from — the file Codex holds open.** Codex writes one JSONL
rollout per conversation under `$CODEX_HOME/sessions/YYYY/MM/DD/`, named
`rollout-<timestamp>-<UUID>.jsonl`, and keeps it open while it runs. Every 2
seconds the app looks at the files the session's own `codex` process holds
open, skips subagents' threads (their first line says
`"thread_source":"subagent"`), takes the most recently written, and records its
UUID as the session's conversation. Codex keeps an earlier conversation's file
open after `/new` or `/resume`, which is why it is the most recently written
file and not the oldest.

**What that buys you.** Restart agent and an app restart resume that
conversation (`codex resume <id>`), and `codex-<acct> --iresume` asks the app
for it. That covers the credit switch: run out of credit on `codex-j1`, quit,
and relaunch `codex-w1 --iresume` in the same session terminal, and it
continues the conversation j1 was in, provided both accounts link the shared
`sessions/`. Two sessions on one worktree each have their own, and `/new` and
`/resume` inside Codex are followed within about 2 seconds.

**Limits worth knowing:**

- **Nothing is recorded before Codex's first turn.** A session restarted
  before then starts fresh, and the panel says so.
- **A home a wrapper picks is invisible to the app unless it links
  `sessions/`.** The app reads the Environment tab's `CODEX_HOME`, else
  `~/.codex`; for an unlinked wrapper home, Restart starts fresh, labelled.
- **An earlier conversation can briefly look current.** A background process
  left running in it that ends just after `/new` writes to its file, so the
  app points back at it until the new conversation writes again.

### Codex usage numbers

The panel shows a Codex session's model, effort, permission mode and tokens,
and its account's 5-hour and weekly quota, read from the rollout the
session's own Codex holds open.

- **The panel** speaks only while Codex runs in the session; between quitting
  one account and starting the next it shows "—".
- **Quota** is the newest `rate_limits` reading (its `codex` bucket) the app
  saw that account's Codex write, kept per account and shown with its age
  ("as of 14:02"). It moves only on turns the app watched: a Codex run in a
  plain terminal, or by Claude Code's Codex plugin, does not update it.
- **Which account** a reading belongs to is the one in effect when Codex wrote
  it: your wrapper's announcement, else the session's `CODEX_HOME`, else
  `codex` (`~/.codex`). A Codex you type by hand with a different
  `CODEX_HOME` and no wrapper is credited to the session's account, so launch
  it through the wrapper.
- **The Usage tab** lists `~/.codex` and every `~/.codex-*` that has a
  `session_index.jsonl`, plus any account with a reading. A plan with no
  usage limit (a business plan, which reports no windows) reads "no usage
  limit (business)".

One thing not to misread: the `N% left` in Codex's status line is the
**context window**, not your quota. Recent versions relabel it `Context 100%
left · 260K window`, which makes that clearer.

### Rejoining the app's conversation

`--iresume` relaunches on the conversation this session is **actually in** —
the one you were just looking at, not merely the one the app started it with.
Tracking (above) follows you either way now; this is about *which* conversation
you continue, not whether the panel keeps working.

It adds the same session binding the app itself composes: `--resume <tracked
conversation>` when the app knows one and its transcript is here, else
`--resume <session id>`, else `--session-id <session id>` for a first launch.
It reads `ISWARM_SESSION_ID` and `ISWARM_TRANSCRIPT_DIR` to find the store, and
`ISWARM_MCP_URL` with `ISWARM_CHANNEL_TOKEN` to ask the app which conversation
that is — all four exported into every session shell. With the messaging and settings lines,
a hand-typed `claude-j1 --iresume` is wired like the app's own launch:
session, messaging, tracking, account.

#### Prerequisite: one transcript store

Set up once in [Creating the accounts](#creating-the-accounts) — every
`~/.claude-*/projects` symlinked to `~/.claude/projects`. It matters here
because the app reads that default store and cannot see which
`CLAUDE_CONFIG_DIR` a wrapper chose (the child's environment, which macOS does
not let a parent read), so an account writing elsewhere gets no model, effort
or tokens attributed, `--iresume` or not. It is also what makes the
credit-switch continuation possible at all: a conversation started under
`claude-j1` is only there for `claude-j2 --iresume` to resume if both write to
the same directory.

#### The flag

```sh
# The binding flags the app itself would append — flag AND id, because they
# are not always the session's own id.
#
# Ask the app which conversation this channel is in rather than deriving it.
# The app resumes the TRACKED conversation and falls back to the bound id, and
# a wrapper that re-derives that rule drifts from it: this one did, within a
# day of being written, and every relaunch afterwards quietly landed somewhere
# other than where the user was.
_iswarm_rejoin_args() {
  if [ -z "${ISWARM_SESSION_ID-}" ] || [ -z "${ISWARM_TRANSCRIPT_DIR-}" ]; then
    echo "--iresume: not inside an InternSwarm session" >&2
    return 1
  fi

  # `local`, because this gets pasted into an interactive profile and `base` in
  # particular is a name an operator is likely to already have. It is not in
  # POSIX, but every shell this doc targets implements it — zsh, bash, and dash
  # included. (What dash does reject is the hyphen in `claude-j1`: a function
  # name has to be a POSIX identifier. That has always been true of this file,
  # which is written for `~/.zshrc`.)
  local tracked base body answered
  tracked=
  if [ -n "${ISWARM_MCP_URL-}" ] && [ -n "${ISWARM_CHANNEL_TOKEN-}" ]; then
    base=${ISWARM_MCP_URL%/}; base=${base%/mcp}
    body=$(printf 'header = "authorization: Bearer %s"\n' "$ISWARM_CHANNEL_TOKEN" |
      curl -fsS -m 3 --config - "$base/api/agent/conversation" 2>/dev/null)
    # Use the answer only if it is about THIS session: the token already
    # scopes it, so a mismatch means something is wrong enough not to guess.
    answered=$(printf '%s' "$body" |
      sed -n 's/.*"session_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')
    if [ -n "$body" ] && [ "$answered" = "$ISWARM_SESSION_ID" ]; then
      tracked=$(printf '%s' "$body" |
        sed -n 's/.*"conversation_id"[[:space:]]*:[[:space:]]*"\([0-9a-fA-F-]*\)".*/\1/p')
    elif [ -n "$body" ]; then
      echo "--iresume: the server answered for ${answered:-no session}, not $ISWARM_SESSION_ID — ignoring it" >&2
    fi
  fi

  # Its transcript has to exist in the store this account reads, exactly as the
  # app checks: `--resume` on an id with no transcript finds no conversation
  # and dies at the prompt rather than falling back.
  if [ -n "$tracked" ] && [ -f "$ISWARM_TRANSCRIPT_DIR/$tracked.jsonl" ]; then
    echo "--resume $tracked"
  elif [ -f "$ISWARM_TRANSCRIPT_DIR/$ISWARM_SESSION_ID.jsonl" ]; then
    # Reached three ways — the app could not be asked, it said nothing is
    # tracked, or the tracked id has no transcript in this store — and only the
    # last of those puts you in an OLDER conversation than the one you left. The
    # app labels that substitution rather than performing it silently; so should
    # this.
    [ -n "$tracked" ] &&
      echo "--iresume: $tracked has no transcript here — resuming the bound conversation, which is OLDER" >&2
    echo "--resume $ISWARM_SESSION_ID"
  else
    # This CREATES a conversation rather than resuming one. Say so: silence
    # here is what makes a relaunch look like it worked when it did not.
    echo "--iresume: nothing to resume — starting a NEW conversation" >&2
    echo "--session-id $ISWARM_SESSION_ID"
  fi
}

# The plain per-account wrapper from earlier, with the flags. `${1-}` rather
# than `$1` so a bare `claude-j1` survives `set -u`.
claude-j1() {
  if [ "${1-}" = --iresume ]; then
    shift
    local args flag id
    args=$(_iswarm_rejoin_args) || return 1
    # Split by parameter expansion, NOT by leaving `$args` unquoted: zsh does
    # not word-split unquoted parameters, so `set -- $args` there passes one
    # argv entry — `--resume <uuid>` as a single option — and the launch dies
    # on an unrecognised flag. This form is correct in both shells.
    flag=${args%% *}
    id=${args#* }
    set -- "$flag" "$id" "$@"
  fi
  env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j1" claude "$@"
}
```

Then `claude-j1 --iresume` (with anything else after it — `claude-j1
--iresume --model opus`) rejoins the session's own conversation, tracked;
`claude-j1` on its own still starts a fresh one; outside InternSwarm
`--iresume` fails loudly instead of guessing. Two sessions on one worktree
resolve correctly: each shell carries its own id.

An explicit flag rather than a guard on "no arguments", so that it never
fires by surprise: a plain `claude-j1` means what it always meant. Check for
it before anything else in the wrapper rewrites `$@` — it reads `$1`, and the
messaging line in the next section puts `--mcp-config` in front of your
arguments.

`claude-j1 --resume` with a conversation picked from the CLI's list keeps
*its own* id — a different conversation from the one the app launched. That
used to leave the panel to go quiet after an hour; with the tracking hook it
no longer does — the hook reports the picked id and the panel follows it. And
because `--iresume` now asks for that same reported id, a later credit-switch
returns you to the picked conversation rather than to the one the app bound at
launch. What `--iresume` gives you is a way back to where you were without
picking from a list. (Forking a picked conversation into
the app's id — `--resume <picked> --fork-session --session-id
"$ISWARM_SESSION_ID"` — is a copy, not a continuation, and only works while
that id holds no transcript, which after any app launch it does.)

This used to have an honest limit, and it is worth knowing what it was because
the shape of the bug is general. `--iresume` rejoined the conversation the app
**bound at launch**, not the one you were just in: after a `/clear`, or a resume
from the picker, you landed in the *pre-clear* conversation — or, if the bound
id had never been written at all, in a brand-new one with nothing saying so.

The cause was not the check; it was that the check lived in two places. The
wrapper mirrored the app's launch rule, the app improved its rule the next day,
and nothing kept the copy honest. Asking the app removes the copy. If you write
your own wrapper, ask rather than re-derive — the endpoint exists for exactly
that.

### Wrap the bare name too

When Claude exits it prints a helpful line:

```
Resume this session with:
claude --resume <id>
```

That command names the **naked binary** — copied verbatim it launches with no
messaging config, no tracking hook, and no account announcement, at exactly
the moment (a credit switch) you most want all three. The fix: alias the bare
name to your default account, and the same for Codex —

```zsh
claude() { claude-j1 "$@"; }
codex()  { codex-j1 "$@"; }
```

Neither calls itself: the wrappers reach the binary through `env` or
`command`, and both find it on `PATH` rather than the function.

Now the printed hint goes through the wrapper, and so does an app auto-start
that runs plain `claude` (which otherwise shows "Account: unknown" in the
panel). Outside InternSwarm every iswarm flag is env-gated away, so this is
just Claude pinned to one account with `ANTHROPIC_API_KEY` unset — which also
means an API key exported by your profile cannot silently switch a casual
`claude` to per-token billing. Pick an OAuth account as the target, and
`command claude` still reaches the raw binary.

One deliberate consequence: this **overrides an Environment-tab
`CLAUDE_CONFIG_DIR`** — the wrapper pins its own directory, so typing
`claude` in a session whose Environment tab selected another account
re-pins that launch to the default. If you rely on Environment-tab
selection on a machine, skip the alias there (the per-account wrappers and
`command claude` are unaffected); on a remote server the question does not
arise, because shell functions from your dotfiles are not installed there
anyway.

### What survives an app restart

Your place in the conversation. Tracking follows every switch — `/clear`,
`/resume`, a credit switch — while the app runs, and the last conversation
each session reported is stored durably, so **restarting the app relaunches
each session on the conversation you were actually in**, whichever account
launched it. That is the point of the shared transcript store: a conversation
started under `claude-j1` and continued under `claude-j2` resumes correctly
under whichever account the session runs next.

The one condition is that the conversation's transcript still exists in the
store the session launches against. When it does not, the session falls back
— to its own launch-bound conversation, or to a fresh one if that transcript
is gone too — and **the status bar says which**, rather than letting a
restart quietly masquerade as the conversation it failed to bring back:

- `Conv: baa106 · resumed · last active Aug 11` (the end of the conversation id) — it could not reach the
  conversation you were in and resumed an older one. The date is there so a
  long-dead conversation is visible *before* you spend more usage on it;
  Claude Code will not warn you, because a `--resume` simply loads.
- `started fresh — previous conversation unavailable` — nothing could be
  resumed. Your earlier conversation is still on disk; the resume picker's
  newest entry for that directory is where you were.

Either label sticks until you actually leave the conversation it explains —
resuming something else, `/clear`ing, or terminating the agent — not until
the CLI's next `SessionStart` report. A `--resume`d launch reports its own
startup under `source: "resume"`, not `"startup"`, so the label would
disappear within seconds of appearing if it were keyed off that; it is keyed
off the reported conversation id instead.

### Caveats

- **A command is machine-local.** A shell function does not exist on a
  remote server. To select an account there, set `CLAUDE_CONFIG_DIR` (or
  `CODEX_HOME`) in the **Environment** tab instead — that works on any
  server. The panel still needs an announcement to name a Codex account.
- **Never start InternSwarm, or its server, from inside a Claude Code
  session.** It inherits that session's `CLAUDE_CODE_*` markers, which turn
  transcript saving off in every Claude Code session it then starts: no
  model, effort or tokens, no Waiting and no limit. Start it from Finder, the
  Dock, or an ordinary terminal.
- **A command is typed into the session's shell, not exec'd.** That is why a
  shell function works at all — see [What the app appends to your
  command](#what-the-app-appends-to-your-command) for exactly what lands
  after it, and what breaks a wrapper.
- **Your shell startup runs after the app's environment.** If a profile
  unconditionally exports something the app set or removed, the profile
  wins.

## Agent-to-agent messaging for hand-launched agents

Add one line to a wrapper before it runs the real binary. This is the piece
that makes a hand-typed relaunch, not just an app-launched one, capable of
sending and receiving:

```zsh
# claude wrappers — before running the binary:
[[ -n $ISWARM_MCP_CONFIG ]] && set -- --mcp-config "$ISWARM_MCP_CONFIG" "$@"

# codex wrappers:
[[ -n $ISWARM_MCP_URL ]] && set -- \
  -c "mcp_servers.iswarm.url=\"$ISWARM_MCP_URL\"" \
  -c 'mcp_servers.iswarm.bearer_token_env_var="ISWARM_CHANNEL_TOKEN"' "$@"
```

Folded into the announcing `claude-j1` wrapper from the previous section,
with `--iresume` from [Rejoining the app's
conversation](#rejoining-the-apps-conversation), the whole thing looks like
this. Keep the subcommand and `--iresume` checks **first**: both read `$1`,
and the messaging line puts `--mcp-config` in front of your arguments. The
subcommand check is the one described under [What the app appends to your
command](#what-the-app-appends-to-your-command):

```zsh
# Whether a bare first word is a Claude Code subcommand (`stop <id>`,
# `agents`, `mcp list`). Asks the installed binary, so a subcommand added
# upstream is known without editing this. When the answer is unclear it says
# yes: a session started without the flags is visible and free, a word sent
# to the model as a prompt is a billed turn.
_iswarm_claude_is_subcommand() {
  [[ -n ${1-} && ${1-} != -* ]] || return 1
  local help
  help=$(command claude "$1" --help 2>/dev/null)
  [[ -n $help ]] || return 0
  [[ $help != *'Usage: claude [options] [command] [prompt]'* ]]
}

claude-j1() {
  # A subcommand runs as the account, announced, with none of the flags
  # below (in front of it they make `stop <id>` a new conversation) and
  # without this terminal's channel token. `agents` starts sessions of its
  # own; if they inherit this environment (not measured), the token would let
  # each report itself as this session's conversation and message as this
  # session. Withheld from every subcommand, so one added upstream is covered
  # too. The cost: inside a session, `claude-j1 mcp list` may show a
  # persisted iswarm entry as failing, since it gets no token.
  if _iswarm_claude_is_subcommand "${1-}"; then
    iswarm_with_account claude-j1 \
      env -u ANTHROPIC_API_KEY -u ISWARM_CHANNEL_TOKEN \
      CLAUDE_CONFIG_DIR="$HOME/.claude-j1" claude "$@"
    return
  fi

  # `claude-j1 --iresume`: the session binding the app would append.
  # Before the messaging line — this reads $1.
  if [ "${1-}" = --iresume ]; then
    shift
    local args flag id
    args=$(_iswarm_rejoin_args) || return 1
    # Split by expansion, not by leaving `$args` unquoted: zsh does not
    # word-split unquoted parameters, and `set -- $args` there would pass
    # `--resume <uuid>` as one argv entry.
    flag=${args%% *}
    id=${args#* }
    set -- "$flag" "$id" "$@"
  fi

  # Messaging, on every launch inside InternSwarm.
  [[ -n $ISWARM_MCP_CONFIG ]] && set -- --mcp-config "$ISWARM_MCP_CONFIG" "$@"

  # Conversation tracking, on every launch inside InternSwarm — WITHOUT this
  # line a hand relaunch is not tracked and the panel goes blank on /resume,
  # which is the core thing this fixes.
  [[ -n $ISWARM_SETTINGS ]] && set -- --settings "$ISWARM_SETTINGS" "$@"

  iswarm_with_account claude-j1 \
    env -u ANTHROPIC_API_KEY CLAUDE_CONFIG_DIR="$HOME/.claude-j1" claude "$@"
}
```

With that, a hand-typed `claude-j1 --iresume` is wired like the app's own
launch line — session, messaging, tracking, account — which is the whole point of the
credit-switch flow: hit the limit on `claude-j1`, quit, type
`claude-j2 --iresume`, and the same conversation continues under the other
account with the panel, badge and messaging all still working (given the
[shared transcript store](#prerequisite-one-transcript-store)). One
difference from the app's own line: the wrapper adds messaging whenever the
variable is set, whether or not **Automatically add messaging to sessions I
start** is on — see below.

Or, without the account-announcement layer, just the plain per-account
wrapper from earlier plus messaging — and the same `--iresume` for Codex. It
asks the app which conversation this session is in, exactly as the Claude Code
one does (see [Codex session tracking](#codex-session-tracking)), checks that
the conversation is in this account's sessions, and runs `codex resume <id>`.
When the app has nothing it can use, it opens Codex's own picker and says
why:

```zsh
codex-j1() {
  # `codex-j1 --iresume`: rejoin the conversation the app recorded for this
  # session. First, for the same reason as above: it reads $1.
  if [ "${1-}" = --iresume ] && [ "$#" -eq 1 ]; then
    shift
    local base body answered id
    if [ -n "${ISWARM_MCP_URL-}" ] && [ -n "${ISWARM_CHANNEL_TOKEN-}" ]; then
      base=${ISWARM_MCP_URL%/}; base=${base%/mcp}
      body=$(printf 'header = "authorization: Bearer %s"\n' "$ISWARM_CHANNEL_TOKEN" |
        curl -fsS -m 3 --config - "$base/api/agent/conversation" 2>/dev/null)
    fi
    answered=$(printf '%s' "$body" |
      sed -n 's/.*"session_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')
    if [ -z "$body" ]; then
      print -u2 -- "--iresume: could not ask the app — opening the picker"
    elif [ "$answered" != "${ISWARM_SESSION_ID-}" ]; then
      print -u2 -- "--iresume: the server answered for ${answered:-no session}, not ${ISWARM_SESSION_ID-} — opening the picker"
    else
      id=$(printf '%s' "$body" |
        sed -n 's/.*"conversation_id"[[:space:]]*:[[:space:]]*"\([0-9a-fA-F-]*\)".*/\1/p')
      if [ -z "$id" ]; then
        print -u2 -- "--iresume: the app has no Codex conversation recorded for this session — opening the picker"
      elif [ -z "$(print -r -- $HOME/.codex-j1/sessions/**/rollout-*-$id.jsonl(N.))" ]; then
        print -u2 -- "--iresume: $id is not in this account's sessions — opening the picker"
        id=""
      else
        print -u2 -- "--iresume: rejoining $id"
      fi
    fi
    set -- resume ${id:+"$id"}
  elif [ "${1-}" = --iresume ]; then
    shift
    # `codex resume` is [OPTIONS] [SESSION_ID] [PROMPT]: an id of ours in
    # front of yours would put yours in the prompt slot.
    print -u2 -- "--iresume: arguments given, so the app was not asked"
    set -- resume "$@"
  fi
  # `-c` overrides before the `resume` subcommand are accepted.
  [[ -n $ISWARM_MCP_URL ]] && set -- \
    -c "mcp_servers.iswarm.url=\"$ISWARM_MCP_URL\"" \
    -c 'mcp_servers.iswarm.bearer_token_env_var="ISWARM_CHANNEL_TOKEN"' "$@"
  CODEX_HOME="$HOME/.codex-j1" command codex "$@"
}
```

Facts behind those two guard lines, all live-verified against real binaries:

- `--mcp-config` can be passed more than once, and every file loads — your
  own config is never displaced. Plain `--mcp-config` **merges** with
  whatever else you have configured; `--strict-mcp-config` **replaces** it.
  Never reach for strict — it silently drops every other MCP server you
  have.
- Codex's `-c` can define an entire MCP server inline, and
  `bearer_token_env_var` tells Codex to read the token from the environment
  at connect time rather than taking it as a literal. The inner double
  quotes are load-bearing: Codex parses the value half of `-c key=value` as
  TOML, and without them the value is not a string and the server never
  registers.
- `$ISWARM_MCP_URL` is written into the command line as a literal rather
  than staying a variable reference because Codex has no `${VAR}` expansion
  for `url`. The token stays indirect — read by name at connect time —
  because that is the only way a token that exists solely inside that PTY's
  environment can reach Codex.
- Both lines guard on the env var, so **outside iSwarm they add nothing at
  all**: no failed `iswarm` entry shows up in an ordinary terminal session,
  because `$ISWARM_MCP_CONFIG` / `$ISWARM_MCP_URL` are simply unset there.

### "Automatically add messaging to sessions I start" does not cover this

iSwarm's setting named exactly that (`agent_messaging_enabled`, on by
default) gates one thing: the flags the *app itself* appends when it
composes a launch line for Start agent or Restart agent. It is read fresh
every time one of those runs, so flipping it takes effect on the **next**
Start agent or Restart agent — never on a session that is already running.

"Sessions I start" means launches the app types into the terminal itself,
not the session as a whole — a hand-typed relaunch, the entire scenario this
document is for, is not one of those, even inside a session the app
originally started. The setting also does not touch the `/mcp` endpoint,
token minting, or delivery, which are per-request rather than per-launch.
That is why the wrapper lines above work whether the setting is on or off:
`$ISWARM_MCP_CONFIG` and `$ISWARM_MCP_URL` are exported into every iSwarm
session regardless, because a hand-launched agent was always meant to work
independently of it.

### Approve the tool once

MCP tools require permission. In a non-interactive session, a tool you have
not approved fails instantly with no prompt to approve it — which silently
breaks unattended use. Approve `iswarm_message` once (or allowlist it) so an
unattended session can actually use it. For Claude Code, allowlist it in
`~/.claude/settings.json`, which every account shares through the link above:

```json
{ "permissions": { "allow": ["mcp__iswarm__iswarm_message"] } }
```

Merge that into the file's existing `permissions.allow` rather than replacing
it. For Codex, approve it the first time Codex asks.

`wait` returns no messages rather than failing after 30 seconds — that is
normal, call it again. `timeout_ms` moves that, and the server caps it at 45
seconds; nothing raises the cap. `MCP_TOOL_TIMEOUT` (milliseconds) sets your
*client's* kill point instead, for *every* MCP tool in that session, not just
this one, which is why iSwarm does not set it for you.

## Verify

- Ask the agent to use the tool — e.g. "list your iswarm peers", or send a
  test message to another session by the name `peers` gives for it. This is
  the check that works for both
  CLIs and both the automatic and hand-launched paths: a live server
  answers, and a session with no channel token reports the tool
  unavailable rather than erroring, so either way a real reply (or a clean
  "unavailable") is the positive proof.
- `claude mcp list` only reads *persisted* config, so it reports nothing
  for a connection supplied per-invocation via `--mcp-config` — which is
  how every path in this document reaches Claude Code. Do not use it to
  verify a hand-launched agent; it will show `iswarm` as absent even when
  it is working.
- **Claude Code only:** `/mcp` inside the running session lists the
  servers actually connected *for that session*, including ones loaded via
  `--mcp-config` — unlike `claude mcp list`, which only reads persisted
  config. (`claude mcp list` is still the right check for the one-time
  manual paste into user-scope config described in the app's Share This
  Mac settings — that entry *is* persisted config.)
- The usage panel names the account, once the wrapper has written its
  announcement.
