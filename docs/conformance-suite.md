# Conformance suite

> Published from InternSwarm's code repository, which is not public yet, at 420ba314. Anything about building from source applies there.

## What it is

A status reader is verified against the agent, not against a version
number: Claude Code and Codex change their screens and record files
without notice, and a version string says nothing about what changed. This
suite re-checks the assumptions the status readers rest on against the CLIs
installed today, and reports the version each result was taken from.

## Running it

```bash
cargo build --release --manifest-path src-tauri/Cargo.toml --no-default-features --features headless --bin iswarm-server
npm run conformance
```

The suite starts its own `iswarm-server` on a temp data dir and free
loopback ports, with every `CLAUDE*`, `ISWARM_*` and `CODEX_COMPANION*`
variable removed from its environment. It never talks to a server or app
you already run. It launches each CLI through an account wrapper, in a work
folder it keeps between runs, and drives a few one-shot turns: at most 8
short Claude turns and 5 Codex turns per run.

| Option | Default | |
|---|---|---|
| `--claude <wrapper>` | `claude-j1` | the command that starts Claude Code |
| `--codex <wrapper>` | `codex-j1` | the command that starts Codex |
| `--only claude\|codex` | both | run one agent's checks |
| `--server <path>` | `src-tauri/target/release/iswarm-server` | the server binary |
| `--claude-config <dir>` | `$HOME/.claude-j1` for `claude-j1` | where the wrapper keeps its transcripts |
| `--codex-home <dir>` | `$HOME/.codex-j1` for `codex-j1` | where the wrapper keeps its rollouts |
| `--report <file>` | `report.json` in the run folder | where the JSON report goes |

Everything lives under `$XDG_CACHE_HOME/iswarm-conformance` when that is
set, else `$HOME/Library/Caches/iswarm-conformance` on macOS and
`$HOME/.cache/iswarm-conformance` elsewhere: a work folder per pair of
installed versions, `work-<epoch>-claude-<version>-codex-<version>/`, and each
run in `runs/<time>/` with its `report.json` and `server.log`. A run's data dir
is removed when it ends. Claude's and Codex's trust screens show only on the
first run in a work folder, so a new version of either brings a new folder
and a real check of both, and nothing edits the accounts' own files to forget
a trust (maintainer's choice, 2026-10-04). The epoch is raised when results
recorded in older folders can no longer be carried, so the next run checks
afresh on the same versions.

The suite presses a key into an agent only on a screen it has just read
and recognised, and typed text gets its Enter only once it shows on the
input line. A permission prompt is answered only when the server has read
it, its own rows name what the turn asked for, and its option 1 is the
one-time yes. Anything else stops that agent's checks; it never guesses. Codex's update menu is one such screen: its
first option runs `brew upgrade`, so the suite answers nothing on it: the
Codex checks skip, marked untested, until you update Codex or choose Skip by
hand. The run then exits 2, or 1 if another check failed.

One run at a time: runs share the work folder, so a second run started while
one is going exits 2 and names the first one's pid. A lock left by a run that
was killed is taken over; if the system has since given that pid to another
process, delete `run.lock` in the cache folder.

Ctrl-C ends the agents and the server, then exits 130; a second Ctrl-C
kills the server outright. A suite killed any other way (SIGKILL, a crash)
leaves its server running: `pgrep -fl iswarm-server` finds it (`lsof`
cuts the command name short, so grepping its output for `iswarm-server`
never matches).

## Running it daily

`scripts/conformance-daily.sh` runs the suite against `origin/main`, never
a working copy. It keeps a clone of its own under the cache folder
(`iswarm-conformance/src`), resets it to `origin/main`, builds the server
there, runs the suite, and appends everything to
`iswarm-conformance/daily/<date>.log`, with each run's report beside it as
`daily/<time>-<pid>-report.json`. When a run does not pass, it shows a
macOS notification naming the exit code and the log. Its arguments go to
`scripts/conformance.js` (`--claude`, `--codex` and so on), and
`CONFORMANCE_REPO` points it at another remote, read on every run. The log
names the commit tested and the remote it came from.

On macOS a LaunchAgent runs it each day. Copy the script beside the clone
so the job starts from a file the update never rewrites, then load the
agent. The copy updates the clone, runs the clone's own version of the
script and waits for it. When main's version is missing, does not parse,
or stops before its end, the copy sends the notification itself. A run
never updates the copy: copy the script again when it changes.

```bash
mkdir -p "$HOME/Library/Caches/iswarm-conformance"
cp scripts/conformance-daily.sh "$HOME/Library/Caches/iswarm-conformance/conformance-daily.sh"
```

`$HOME/Library/LaunchAgents/com.iswarm.conformance.plist`, with `<you>`
replaced by your user name (launchd does not expand `$HOME`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.iswarm.conformance</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>/Users/<you>/Library/Caches/iswarm-conformance/conformance-daily.sh</string>
  </array>
  <!-- Daily at 09:30; a Mac asleep then runs it when it wakes. -->
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>30</integer></dict>
  <!-- launchd starts with a bare PATH and no SHELL. Put on the PATH the
       folders `command -v node cargo git lsof` shows in your terminal (nvm,
       fnm or volta keep node elsewhere; use their stable default, not a
       per-terminal path); the script names any tool it cannot find. -->
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key><string>/Users/<you>/.cargo/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    <key>SHELL</key><string>/bin/zsh</string>
  </dict>
  <key>StandardOutPath</key><string>/Users/<you>/Library/Caches/iswarm-conformance/daily/launchd.log</string>
  <key>StandardErrorPath</key><string>/Users/<you>/Library/Caches/iswarm-conformance/daily/launchd.log</string>
</dict>
</plist>
```

```bash
launchctl bootstrap "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.iswarm.conformance.plist"
launchctl kickstart "gui/$(id -u)/com.iswarm.conformance"   # one run now, to check it
launchctl bootout "gui/$(id -u)/com.iswarm.conformance"     # to stop it
```

A run uses about 8 short Claude turns and 4 Codex turns on the named
accounts. Only one run holds the work folder at a time, so a run you start
by hand while the daily one is going exits 2 and names it.

## Reading the result

Each check ends in one of five outcomes:

- `pass`
- `fail`
- `skip`, with a reason: limit reached, not observed this run, or not
  checkable daily
- `known`: a gap the maintainer decided to keep, reported with what was
  observed, never failing the run
- `unverifiable`: the check's condition cannot be produced on demand (a
  usage limit appears only when the account is limited), reported with when
  it last passed and on which version, from the reports of earlier runs;
  never failing the run, and not a skip, which reads as something going wrong

The exit code is 1 when any check failed. Otherwise it is 2 when the suite
itself could not run (no server binary, server never ready) or could not
test an agent at all (Codex showing its update menu), and 0 when nothing
failed. An account at its usage limit skips that agent's remaining checks
with a clear message; a skip is not a failure.

## The checks

Claude Code:

- `claude.trust-dialog`: the workspace trust dialog on a folder's first
  run reads as asking, though its options carry no numbers. Checked in the
  first run on each new Claude version, and failed if the dialog does not
  show in that new folder. After that, the newest verdict recorded on this
  version is carried, pass or fail, and the check is skipped when none is; a
  run that stopped at launch records a failure under this check but no
  verdict, so it is never carried;
  the server's reading of the dialog is tested on every push by
  `claudes_unnumbered_trust_dialog_reads_as_a_live_prompt`.
- `claude.idle-shape`: the idle screen (rules framing the `❯` line,
  chrome below) reads as idle on a screen the server can read.
- `claude.permission-prompt`: a real permission prompt replaces the input
  box, ends with its footer, and reads as needing input.
- `claude.hook-before-prompt`: the managed `PermissionRequest` hook runs
  before the permission prompt is read.
- `claude.call-after-answer`: the answered call and its result are in the
  transcript. Whether the call was already there before the answer is
  reported, not judged: it has been seen both ways.
- `claude.end-of-turn-line`: a turn end draws the end-of-turn line above
  the input box.
- `claude.mcp`: the MCP tool is reachable with `--mcp-config`, and its
  result renders collapsed.
- `claude.input-box-while-busy`: the input box stays drawn through a turn,
  the spinner keeps the conversation region moving, and no end-of-turn line
  is drawn before the end.
- `claude.typed-while-busy`: text typed while busy is recorded as the
  person's (`origin.kind: "human"`, or, when queued, an attachment with
  `commandMode: "prompt"`). Skipped when Claude's own record shows the
  turn had ended before the Enter: the run then typed an ordinary prompt.
- `claude.resize`: a resize changes no status.
- `claude.interrupt-row`: an Esc interrupt writes its interrupt row.
- `claude.background`: a background launch and its completion appear in
  the transcript, and the end-of-turn line carries the "still running"
  suffix while the shell runs.
- `claude.waiting`: the session reads Waiting while the background shell
  runs.
- `claude.turn-rows`: `end_turn` appears only on a message that ends a
  turn, and every turn end is followed by a `system` `turn_duration` row.
- `claude.finished-once`: at least one finished event, never more than the
  `turn_duration` rows, and never two with the same end time.
- `claude.usage`: while `claude` still runs, the session's agent-usage has
  the transcript's newest model and its tokens, each reply counted once at
  its newest row (Claude Code writes a row per content block, each repeating
  the reply's usage so far).
- `claude.limit-row`: Claude's limit row carries `error: "rate_limit"` and
  `quotaLimits.resetsAt`. Checked on this run's limit row, or else on the
  account's newest limit row of the last 7 days written by the installed
  version; unverifiable when there is none, skipped when the version is
  unknown.
- `claude.full-screen-child`: the process sampler sees a full-screen child
  take the foreground. Recorded `known`: the sampler names a group by its
  agent, so an editor the agent opens is not seen.
- `claude.sampler`: the process sampler names `claude`: from the moment
  Claude's screen is read until `/exit`, the session never reads no agent,
  and it reads no agent after `/exit`, by the sampler or the shell's exit
  report. A no-agent reading at launch, before Claude holds the foreground,
  does not count.

- `claude.restart-resume`: after Restart agent, the conversation the
  restarted process reports under its new token (the live binding, which the
  restart clears) is the one reported before.

Codex:

- `codex.trust-dialog`: the trust screen on a folder's first run reads as
  asking. Checked in the first run on each new Codex version (see the work
  folder above), and failed if the screen does not show in that new folder.
  After that, the newest verdict recorded on this version is carried, pass or
  fail, and the check is skipped when none is; as for Claude, a stop at launch
  is never carried; the server's reading of the screen is tested on every
  push by
  `codexs_trust_screen_and_update_menu_read_as_live_prompts`.
- `codex.idle-shape`: the idle screen (the `›` line, footer below) reads
  as idle.
- `codex.approval-prompt`: an approval prompt replaces the input line, ends
  with its footer, and reads as needing input.
- `codex.open-call`: the approval coincides with a tool call in the log
  that has no output yet.
- `codex.rollout-open`: the running process holds its `rollout-*.jsonl`
  open.
- `codex.mcp`: the MCP tool is reachable with the `-c mcp_servers.iswarm…`
  flags.
- `codex.background`: Codex, asked to leave `sleep 20; echo done` running
  and end its turn, ends the turn with the process running; the process's
  end is recorded for that turn in the 35 s that follow; no turn starts
  after the turn ended; and the session reads idle with one finished event
  for the turn. A Codex that resumes on its own after a background process
  would make "finished" fire mid-work. The command is matched whole. Fails
  when Codex ran it and ended its turn but recorded no end for it. Skipped,
  saying why, when Codex never ran it or it returned inside the turn (the
  skip says after how many seconds).
- `codex.restart-resume`: after Restart agent, the restarted `codex` holds
  the same rollout open as before (it reopens it at start-up, measured on
  0.160.0; the check sends one short turn only if it has not).
- `codex.panel`: while `codex` still runs, the session's agent-usage has the
  model and effort of the rollout's last `turn_context`, and the account is
  the announced Codex account with a quota reading from this run.
- `codex.turn-events`: the log carries `task_started` and `task_complete`
  with a `turn_id`, and `token_count` with `rate_limits`.
- `codex.finished-once`: at least one finished event, all from the session
  log, never more than the `task_complete` rows, and never two with the same
  end time.
- `codex.limit-shape`: a refused turn carries `usage_limit_exceeded` and a
  "try again at" time the reader parses. Checked on this run's refusal, or
  else on the account's newest refusal of the last 7 days written by the
  installed version; unverifiable when there is none, skipped when the
  version is unknown.
- `codex.sampler`: the process sampler names `codex`: from the moment
  Codex's screen is read until `/quit`, the session never reads no agent,
  and it reads no agent after `/quit`, by the sampler or the shell's exit
  report.
- `codex.inline-renderer`: after the last `/quit`, the suite types
  `<wrapper> --no-alt-screen` into the session's own shell, as a person
  relaunching by hand would, and Codex draws inline: the shell's launch line
  stays above Codex's header, which the default renderer's alternate screen
  never shows. When it does not, the three checks below are skipped: read on
  a screen that is not inline, they would pass as inline ones.
- `codex.inline-idle`: the inline idle screen reads as idle.
- `codex.inline-approval`: an approval drawn inline reads as needing input.
- `codex.inline-answer`: answering it with `1` runs the command, the turn
  finishes, and the session reads idle again.

A real rate limit cannot be provoked on demand, but an account in daily use
meets one often. So the limit checks read the newest limit record in the
account's own files of the last 7 days, written by the installed version,
when the run itself meets none. A record from an older version says nothing
about the installed one. Only
the lines carrying the limit marker are read, and the report shows the
verdict, not their content. When a run does meet a limit, its own record is
checked before the remaining checks are skipped.

## When a check fails

A failure means the CLI changed something a reader depends on.

1. Open `report.json` and `server.log` in the run folder. A failed screen
   check carries the screen it saw.
2. Record the new screen or record row as a fixture.
3. Fix the reader against that fixture.

Never loosen a check to make it pass: the check is the assumption, and a
loosened one stops telling you when the assumption breaks.
