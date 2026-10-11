# Operating InternSwarm

> Published from InternSwarm's code repository, which is not public yet, at ff8dce29. Anything about building from source applies there.

How the server, the desktop app and the hosted relay fit together, and how to
run, expose and install a server by hand. The [README](../README.md) is the
short version.

## Current Product Model

- **Local server:** the desktop app can start `iswarm` on `127.0.0.1:7822` while the app is open. Local desktop-owned sessions use `desktop_ephemeral` lifecycle cleanup.
- **Saved remote servers:** the desktop app can connect to one or more standalone `iswarm-server` instances on a trusted LAN/VPN/private network. Those servers own their projects, sessions, auth, and PTY runtime.
- **Standalone same-machine server:** `iswarm-server` can also run manually on the same machine for persistent local use. This is operationally the same server shape as a remote server, just addressed through localhost.
- **Web and mobile:** browser/tablet/mobile clients use the same HTTP/WS/SSE API and resolve one current server at a time.

## Network Exposure Modes

InternSwarm supports a private baseline and a hardened public server path.

For localhost, LAN, VPN, or Tailscale use, protected HTTP routes use bearer-token auth and stream endpoints may use bearer query tokens for compatibility.

For public direct HTTPS or hosted-relay use, app TLS terminates in `iswarm-server`. Public mode rejects query `auth_token`, uses secure `HttpOnly` session cookies for browser HTTP, requires CSRF headers for cookie-authenticated mutations, and authorizes WS/SSE with short-lived single-use stream tickets. Password hashes are Argon2id after automatic legacy migration. Mobile pairing tokens are short-lived and single-use.

Do not expose a plaintext `http://` public server. Public/mobile pairing URLs must use `https://`; private LAN and Tailscale deployments may continue to use private `http://` origins behind the trusted network boundary.

Native TLS settings live in `server-admin.json` beside `local-server.json`. Public direct HTTPS and relay modes require an enabled HTTPS listener, a primary hostname, a matching unexpired certificate in the server-local TLS store, and hardened auth settings. Managed-domain CSR generation creates the private key on the server machine and sends only the CSR to the control plane; certificate installation uses the stored pending private key.

## Hosted Relay (optional)

InternSwarm can reach a private server from anywhere through the **InternSwarm hosted relay** — a managed service that gives your server a stable public hostname without opening inbound ports. App TLS terminates in your `iswarm-server`, so the relay cannot read sessions, terminal output, prompts, files, cookies, pairing tokens or certificate private keys. It does see connection metadata: the SNI hostname, the client's IP address and port, the account, server, tunnel and region identifiers, connection timing, duration and byte counts, and tunnel health. The relay's operator controls DNS for your relay hostname; to remove that trust, publish with **My own domain** instead, which serves direct HTTPS with no relay. [The relay trust model](relay-trust-model.md) says what the relay sees and stores, agent mail included.

The relay is a paid hosted service and its server components are not part of this repository. This repository contains the **client** integration: `iswarm-server` owns public-access mode, HTTPS, certificates, relay account linkage, hostname selection, and tunnel lifecycle, and drives the bundled `iswarm-tunnel` sidecar. If the sidecar stops, relay reachability stops, but `iswarm-server` keeps running and direct localhost/LAN/VPN/Tailscale URLs continue to work.

## Configuration UX

Desktop settings separate client reachability from server authority:

- **Connect to Servers** is the desktop client registry. It stores local/remote server URLs and sign-in state so the desktop can connect to those servers.
- **Server Admin** configures the selected authenticated `iswarm-server`: public access mode, native HTTPS, certificate state, relay account/hostnames, and `iswarm-tunnel` lifecycle.
- **Share This Mac** remains the desktop-owned local server bind/auth policy for localhost, LAN, VPN, and Tailscale use.
- **Mobile Pairing** issues the pairing token on the selected server. When that server has relay enabled and a relay hostname, the QR URL prefers `https://<relay-hostname>`; direct private URLs remain usable when relay is disabled.

## When a Session's Shell Asks Something at Startup

A session's agent starts once its shell is ready to read a command, so a
startup question (oh-my-zsh's "Would you like to update? [Y/n]",
Powerlevel10k's first-run wizard) waits for you. After 3 s the session says
"Waiting for shell". Answer the question in the terminal, or on the phone with
the reply box, and the agent starts.

A shell is ready when its line editor turns bracketed paste on, which zsh,
fish and bash 5.1 or later do at every prompt, or when it prints a
shell-integration prompt marker (OSC 133). Other shells (macOS's `/bin/bash`
3.2, `sh`, `dash`), and every shell on Windows, get the launch typed at once,
as before.

A shell that does neither (bracketed paste switched off and no prompt marker)
never says it is ready, so its session shows the waiting state. **Start
anyway** types the launch at once, into whatever is reading the terminal: use
it for such a shell, not while a startup question is showing, or the question
reads the launch line. A program running in that shell (one you start, such
as an agent or an editor, or a picker your startup files draw) never receives
the waiting launch: it goes out only while the shell itself is in front, and
otherwise keeps waiting. A startup
question that uses the line editor itself (bash's `read -e`, zsh's `vared`)
looks ready, and the launch is typed into it.

## Agent-to-Agent Messaging

Messaging is automatic for sessions you start from InternSwarm: Start agent and Restart agent add
the MCP connection to the launch line themselves, so agents in your sessions can message each
other over the built-in `/mcp` endpoint with nothing to paste. `iswarm` injects
`ISWARM_CHANNEL_TOKEN` and `ISWARM_MCP_URL` into every session it launches; a session started
outside InternSwarm has neither, so the messaging tool simply reports itself unavailable and
nothing else about that session changes. Running more than one account, or relaunching an
agent by hand mid-session? The app-appended flags do not cover that — see
[multi-account.md](multi-account.md) for the wrapper that does.

A recipient is addressed by its **name** — the label on the session, `dotfiles#2` by default
and editable, unique across the whole server. `peers` lists every session an agent can
message together with the name to address it by, and an incoming message is prefixed with the
sender's name, so a reply goes straight back to it. A whole session id still resolves too (as
do its first 8 characters, when they are unambiguous); unlike a name, an id never changes when
somebody renames the session.

Another **person** is addressed by their account — `alice@example.com` — and the message goes
to every device of theirs this server has verified, one sealed envelope each. Verification is
a deliberate step: you and they compare a fingerprint out of band, and it gates the two
directions differently. **Mail you send to a device that is not ready is not queued for
later** — that device is skipped, and a send that reaches none of them is an error rather
than a receipt. The receipt lists every device that got no envelope from this send, and why.
Sending the same message again is not a retry of the skipped devices — it is a second send to
all of them, so anyone reached the first time receives it twice.
**Mail arriving from an unverified device waits** in Held Mail until a person decides,
which is what keeps a stranger's text out of an agent's context. Setting this up with a
teammate, from the invite through the eight words to Held Mail and standing grants, is walked
through in [teammates.md](teammates.md), which the app also shows from Peers and Held Mail.

`wait` times out after 30 seconds and returns no messages rather than failing — call it
again, that is the normal way to keep waiting. Passing `timeout_ms` (a JSON number, not a
quoted one) moves that, up to a server cap of 45 seconds. The cap was set from a measurement:
Claude Code 2.1.220 kills an MCP tool call at roughly 55 seconds, so 45 leaves margin and a
wait ends by returning an empty list rather than by being killed.

**45 seconds is the ceiling for a single call, whatever your client is set to** — the server
clamps what you ask for, so nothing raises it. `MCP_TOOL_TIMEOUT` (milliseconds, set in your
environment) moves your *client's* kill point instead, for every MCP tool in the session. It
helps only if your client gives up sooner than 45 seconds. Claude Code's default is about 55,
so by default it changes nothing — but it is the same knob in both directions: set below the
wait's own length, it kills the call instead of letting it return empty. A wait ends as soon
as a message arrives, so that shows up as waits failing only when nothing was there — busy
channels look fine, quiet ones break. To keep waiting past the ceiling, call `wait` again.

In a non-interactive session, an MCP tool you have not approved fails instantly with no
prompt to approve it — approve `iswarm_message` once (or allowlist it) so an unattended
session can actually use the feature. Both of the above apply whether messaging reached the
session automatically or through a snippet pasted by hand below.

### Other CLIs, or by hand

Setup is one-time per machine: paste the snippet for your CLI into its config once, and the
same text works for every session and every project after that. The desktop app's **Share
This Mac** settings has a copyable "Agent messaging" section with the same two snippets,
behind an "Other CLIs, or by hand" disclosure.

**Claude Code** — add this entry to the `mcpServers` object in Claude Code's user-scope
config (`~/.claude.json`, or `claude mcp add-json --scope user`). It reads the port and token
from the session environment, so the same text works on any machine, even on a non-default
port:

```json
"iswarm": {
  "type": "http",
  "url": "${ISWARM_MCP_URL:-http://127.0.0.1:7822/mcp}",
  "headers": { "Authorization": "Bearer ${ISWARM_CHANNEL_TOKEN:-}" }
}
```

Confirm it registered: `claude mcp list` should show `iswarm`.

**Codex** — add this to `~/.codex/config.toml`. Codex has no `${VAR}` expansion for `url`, so
it is literal; edit it if this server is not on port 7822. The token is still read from the
environment, by name:

```toml
[mcp_servers.iswarm]
url = "http://127.0.0.1:7822/mcp"
bearer_token_env_var = "ISWARM_CHANNEL_TOKEN"
```

The `:-` defaults in the Claude Code snippet are load-bearing: without them, an unset
variable is passed through as the literal string `${ISWARM_CHANNEL_TOKEN}`, and `/mcp`
rejects that with its own 401 message — so a session launched outside InternSwarm fails cleanly
instead of mysteriously.

## Building from source

### Desktop With Local Server

```bash
npm install
npm run build
npm run tauri:dev
```

The desktop app starts the local server when `Start local server with desktop` is enabled in Settings.

#### Inspecting a release build

`npm run tauri:dev` already gives you a WebKit inspector — it builds with
`debug_assertions`, and tauri-runtime-wry enables devtools under
`any(debug_assertions, feature = "devtools")`. What has never been inspectable
is a RELEASE build, which is the one that reproduces slow-burn behaviour like
memory growth over hours.

```bash
npm run tauri:build:devtools
```

That adds the opt-in `devtools` cargo feature. Attach from Safari:
**Develop → [your Mac] → tauri://localhost**.

Three things to know before you use it.

**It is deliberately not the default.** An inspectable production app is a
different thing to ship, so this is a build you make on purpose.

**The artifact is indistinguishable from a shippable one.** It writes to the
same `target/release/bundle/` paths under the same product name, version and
identifier as a plain `tauri build`. Nothing in the filename says "inspectable".
Delete it when you are done, or you will eventually ship it by accident. The
discriminators are Safari's Develop menu seeing it, or
`strings <binary> | grep developerExtrasEnabled`.

**It changes what you are measuring.** wry sets `developerExtrasEnabled` for the
webview's whole lifetime, and an attached Web Inspector retains JS objects and
adds instrumentation of its own. Dirty-page numbers from this build are not
directly comparable to the shipped one, and nobody has quantified the gap — so
use it to find a retainer, not to certify a floor.

On the App Store: `setInspectable(true)` is public API (macOS 13.3+), but wry
also sets `developerExtrasEnabled`, a private KVC key on `WKPreferences`, on
every macOS version whenever the feature is on. A binary built this way must not
go to the App Store. For a DMG it is fine, which is how this ships.

### Standalone Server From Source

Use this for a persistent server on the same machine or another private-network host:

```bash
npm install
npm run build
cargo run --manifest-path src-tauri/Cargo.toml --bin iswarm-server --no-default-features --features headless
```

Then open `http://127.0.0.1:7822/` locally, or configure `ISWARM_REMOTE_ACCESS_ENABLED=true`, a bind address, and an auth password before exposing it on a LAN/VPN address.

For direct public HTTPS access, configure `/api/server/settings`, generate a CSR with `/api/server/tls/csr`, install the returned certificate with `/api/server/tls/certificate`, and verify `/api/server/tls/status` before advertising the public URL.

### Packaged Server Install

Use `./scripts/install-iswarm.sh` only when you intentionally want to build from a checked-out repo on the destination machine. For normal remote deployment, use the packaged install path because the destination machine does not need Node, Rust, Cargo, or Tauri tooling.

Build an installable archive on a machine with Rust and Node tooling:

```bash
npm install
npm run package:server:check -- --platform macos-arm64
npm run package:server -- --platform macos-arm64 --flavor release
```

For Linux packages from macOS, install the prerequisites printed by the check command, then package the target architecture:

```bash
npm run package:server:check -- --platform linux-x86_64
npm run package:server -- --platform linux-x86_64 --flavor release
```

Packaging emits archives under `artifacts/` plus checksum, install-script, metadata JSON, and manifest sidecars. The destination machine does not need a checked-out repo or build tools.

On the destination machine:

```bash
chmod +x <archive-name>.install.sh
./<archive-name>.install.sh
~/.local/share/iswarm/app/current/scripts/verify-install.sh
```

The packaged installer creates a user service through `launchd` on macOS or `systemd --user` on Linux, stores mutable config in `~/.config/iswarm/iswarm.env`, and stores persistent data in `~/.local/share/iswarm/data`.

### When the server runs

**Linux:** the server is a `systemd --user` service. systemd stops a user's
services when that user logs out, and starts them only at a login, unless the
user *lingers*. So the installer turns linger on (`loginctl enable-linger`),
and `verify-install.sh` fails while it is off. With linger on, the server
starts at boot and keeps running after you log out of the SSH session that
installed it. Some distributions let only an administrator turn linger on;
the installer then says so, and you run `sudo loginctl enable-linger $USER`
once. Uninstalling leaves linger on, in case something else needs it:
`loginctl disable-linger $USER` turns it off.

**macOS:** the server is a LaunchAgent, which starts when you log in to the
Mac's desktop. After a restart it stays down until someone does. With
FileVault on, the disk must be unlocked first, and unlocking it over SSH
leaves the Mac at the login window, where the server does not start. Plan on
logging in at the Mac after each restart.

## Runtime Architecture

- One backend contract powers desktop, browser, tablet, and mobile clients: REST for CRUD/settings, WebSocket for PTY channels, and SSE for session/runtime events.
- Each session has one tracked channel, usually `agent`, plus optional helper terminal channels.
- The tracked channel drives sidebar status, notifications, mobile quick replies, and default focus.
- Client disconnects do not kill a server-owned PTY while the owning server keeps running.
- Each server launches its channels locally with `portable-pty`. To work on another machine, run `iswarm-server` there and connect the app to it; the app shows every connected server in one view.
- SQLite stores projects, hosts, sessions, channels, terminals, repo settings, and prompt templates.

## Project Layout

```text
intern-swarm/
├── src-tauri/
│   ├── src/api/         axum routes, PTY WebSocket, SSE events, static UI hosting
│   ├── src/services/    PTY runtime, auth, session lifecycle, git/files/worktree services
│   ├── src/commands/    thin Tauri IPC wrappers for desktop-owned local operations
│   ├── src/db/          SQLite accessors
│   ├── src/models/      serde models shared by API and services
│   ├── src/tunnel/      hosted-relay client + tunnel sidecar protocol
│   ├── migrations/      SQLite schema migrations
│   └── src/bin/         standalone `iswarm-server` and `iswarm-tunnel` entrypoints
├── src/                 React + Vite frontend
├── scripts/             server packaging, install, and guide generators
└── tests/ui/            Playwright browser workflow coverage
```

## Verification

See CONTRIBUTING.md for the checks a change runs.
