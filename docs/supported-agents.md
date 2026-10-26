# Supported coding agents

InternSwarm runs any coding-agent CLI in a terminal it owns. A **verified**
agent gets more than a terminal: a status read from the structure of its
screen and its own records, notifications when it asks or finishes,
auto-answer, and mail typed straight into its input box.

**Verified today:** Claude Code, Codex and Antigravity (`agy`).
**Unverified:** Aider, Goose and OpenCode. Each verified agent is
re-checked every day, and in full whenever a new version of it comes out.

## What each agent supports

"With setup" means a one-time step on your machine, under
[Setting up your machine](#setting-up-your-machine); anything the app
launches needs nothing more.

| Feature | Claude Code | Codex | Antigravity | Aider, Goose, OpenCode |
|---|---|---|---|---|
| **Status:** working, waiting on you, idle | ✓ | ✓ | ✓ | from activity and silence only |
| **Asks:** a notice when it waits on your answer | ✓ | ✓ | ✓ | no |
| **Finished:** a notice when a turn ends | ✓ | ✓ | ✓ | no |
| **Usage limits:** a badge, with the reset time | ✓, with a notice | ✓, with a notice | with setup, a badge but no notice | no |
| **Auto-answer:** rules answer its prompts | ✓ | ✓ | ✓ | no |
| **Typed mail:** messages typed into its input box | ✓ | ✓ | ✓ | no: mail waits in its inbox |
| **Messaging:** it sends and waits for mail itself | ✓ | ✓ | with setup | where the CLI takes an MCP server, set up by hand |
| **Restart resumes** the conversation | ✓ | ✓ | ✓ | no: it starts fresh |
| **Model, effort and tokens** in the status bar | ✓ | ✓ | with setup | no |
| **Account and quota** in the usage panel | ✓; several accounts with wrappers | ✓; several accounts with wrappers | with setup, one account | no |

What you lose where a cell says no, or without the setup:

- **Antigravity's usage limit** shows as a badge in the usage panel, never as
  a notice: agy can carry a turn past an empty weekly pool on AI credits, so
  an empty pool does not prove it was refused. Without the status-line setup
  there is no badge, no model or tokens in the status bar, and no account in
  the usage panel.
- **Antigravity's messaging** needs agy's MCP entry, added once. Without it,
  agy cannot send mail, wait for it or list its peers; mail sent *to* it is
  still typed in.
- **Antigravity's accounts:** the panel shows the account agy is signed in
  to. No way to run a second agy account side by side was found.
- **An unverified agent** gets the terminal and a status from its activity.
  It never raises an ask, never gets a finished notice, is never
  auto-answered, and mail to it waits in its inbox for it to read.

## Setting up your machine

Most of this needs nothing from you: anything the app launches gets every
feature above. Setup matters when you **relaunch an agent by hand** in a
session's terminal, typically to switch to another account after a usage
limit, when you run **several accounts** of one CLI, and, for Antigravity,
once for messaging and usage.

- **Let your agent do it:** paste [agent-setup-prompt.md](agent-setup-prompt.md)
  into Claude Code or Codex on your machine. It asks a few questions, writes
  the wrappers, links the shared stores, and checks the result.
- **Do it yourself:** [multi-account.md](multi-account.md) explains every
  piece and why it is there.

What each piece gives you:

| Piece | Where | What it gives you |
|---|---|---|
| One config directory per account, logged in once | your home folder | several accounts of one CLI |
| Shared stores linked into each account (`projects/` for Claude Code; `sessions/` and friends for Codex) | your home folder | conversations, model and tokens, and resuming, across accounts |
| Account announcement in each wrapper | wrapper | the right account named in the usage panel |
| `--mcp-config "$ISWARM_MCP_CONFIG"`, or Codex's two `-c` lines | wrapper | messaging after you relaunch an agent by hand |
| `--settings "$ISWARM_SETTINGS"` | wrapper (Claude Code) | conversation tracking and auto-answer after a hand relaunch |
| `--iresume` | wrapper | rejoining the session's conversation on a hand relaunch |
| Approve `iswarm_message` once | agent settings | messaging in a session nobody is watching |
| Antigravity's MCP entry, added once | agent settings | Antigravity's messaging |
| Antigravity's status line, set once | agent settings | Antigravity's usage, model and account |
| Start the app from Finder, never from inside Claude Code or Codex | how you launch | Claude Code's records, and with them its model, tokens, limits and finished notices |

### Antigravity, once per machine

Messaging: add InternSwarm's bridge as an MCP server, then approve its tool the
first time agy asks, with option 3 ("always allow … Persist to
settings.json"):

```bash
agy mcp add iswarm -- sh -c 'exec sh "${ISWARM_MCP_STDIO:?not in an InternSwarm session}"'
```

Usage, model and account: add this entry inside the top-level
object of `~/.gemini/antigravity-cli/settings.json`, beside what is already
there. `/statusline` writes the command but cannot add `stack_with_default`,
which keeps agy's own footer below it:

```json
"statusLine": {
  "type": "command",
  "command": "sh -c '[ -n \"$ISWARM_AGY_STATUSLINE\" ] && exec sh \"$ISWARM_AGY_STATUSLINE\"; cat >/dev/null'",
  "stack_with_default": true
}
```

agy has one status-line command: **this entry replaces any status line of
your own**, and the two cannot both be set. Outside InternSwarm the command
reads what agy hands it and draws nothing, and agy's footer is unchanged;
the MCP entry shows as failed in agy's `/mcp`, with the reason.
