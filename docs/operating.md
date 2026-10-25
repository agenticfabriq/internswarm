# Operating InternSwarm

How the server, the desktop app and the hosted relay fit together, and how to
install, run and expose a server. The [README](../README.md) is the short
version.

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

### A server with no desktop, on the relay

The relay makes a server reachable from the internet, so relay setup refuses a server until it has remote access on and an admin password. **Sharing** sets both on any server the app is connected to, except where the server's `~/.config/iswarm/iswarm.env` sets them: Sharing shows those locked, and they are changed in that file. A server with no desktop, such as the packaged Linux server, is usually set up in `iswarm.env`:

1. In `~/.config/iswarm/iswarm.env`, set:

   ```bash
   ISWARM_REMOTE_ACCESS_ENABLED=true
   ISWARM_AUTH_PASSWORD=<a strong password of your own>
   ```

   **`change-me`, the example older packages wrote here, is not a password.** The server ignores it, and with remote access on and no other password it refuses to start until you set one. Once the server is on the relay, this password is what stands between the internet and a shell on this machine. The relay needs no change to `ISWARM_BIND_ADDRESS`: its tunnel connects outward from this machine. Leave `ISWARM_PUBLIC_ACCESS_POLICY` unset; setup switches the server to relay mode itself once its certificate is installed.
2. Restart the server: `systemctl --user restart iswarm` on Linux, `launchctl kickstart -k "gui/$(id -u)/ai.internswarm.server"` on macOS.
3. Reach the server from your desktop app. Forward a port with `ssh -N -L 17822:127.0.0.1:7822 <server>`, using the server's `ISWARM_PORT` in place of `7822` if you changed it, then add `http://127.0.0.1:17822` in **Connect to Servers** (a server you bound to a LAN or VPN address can be added by that address instead). Sign in as `admin`, or your `ISWARM_AUTH_USERNAME`, with the password above.

   Steps 1 and 2 can be done from the app instead when `iswarm.env` has no uncommented `ISWARM_REMOTE_ACCESS_ENABLED` line. Older packages wrote `ISWARM_REMOTE_ACCESS_ENABLED=false`, which counts: Sharing then shows remote access locked. Add the server without signing in, which it does not ask for while remote access is off. In **Sharing**, with this server selected, turn on remote access and set a password, unless `iswarm.env` sets one, then sign in with it.
4. Open **Server Admin** for that server, choose **Hosted relay**, and approve the code in the portal. Installing the certificate takes about two minutes. The server is then reachable from anywhere at the relay hostname Server Admin shows, behind its sign-in.

**Behind an HTTP proxy.** If this machine reaches the internet only through an HTTP proxy, add it to `iswarm.env` before step 2:

```bash
HTTPS_PROXY=http://proxy.example.com:3128
```

Relay setup and the relay tunnel both use it, and `NO_PROXY` lists hosts that bypass it. The tunnel takes its proxy only from environment variables, never from the operating system's proxy setting, so set `HTTPS_PROXY` here even on a machine that already has a system proxy. The tunnel asks the proxy for a `CONNECT` to the relay. It goes only through an `http://` proxy, with `user:password@` in the URL if the proxy asks for credentials; if the proxy for the relay is a `socks5://` or `https://` one, the tunnel logs a warning in the server log and dials the relay directly. The tunnel still checks the relay's own certificate against its built-in roots, so the proxy has to pass the relay's host through rather than re-sign it.

## Configuration UX

Desktop settings separate client reachability from server authority:

- **Connect to Servers** is the desktop client registry. It stores local/remote server URLs and sign-in state so the desktop can connect to those servers.
- **Server Admin** configures the selected authenticated `iswarm-server`: public access mode, native HTTPS, certificate state, relay account/hostnames, and `iswarm-tunnel` lifecycle.
- **Sharing** sets the selected server's bind and sign-in policy for localhost, LAN, VPN, and Tailscale use. Settings that server's `iswarm.env` sets are shown locked and are changed there.
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

**Claude may check with you before it answers a message.** A message is typed into the
agent's prompt, and Claude Code can show a longer one to Claude as pasted text. In the trials
behind this paragraph, with Claude Code 2.1.302, Claude answered on its own in a new
conversation, but in one where you had been telling it when to reply it kept asking you
about each new message. Telling it once, in that conversation, *you can reply if you don't
need me for decision or discussion* made it answer on its own, and it still asked you first
when another person's message wanted more than a reply, such as running commands and sending
back what they printed. All of that is Claude's own judgement, seen in a few trials of one
version; nothing in InternSwarm enforces it. InternSwarm decides only which mail reaches the
prompt, as [Held Mail](teammates.md#held-mail) and
[Letting mail skip Held Mail](teammates.md#letting-mail-skip-held-mail) describe. Mail that
reaches it is typed in like your own words, so keep the permission to replies, as that
wording does.

### Long commands: `iswarm_task`

A coding agent stops a command that runs past its own limit (ten minutes for a Claude Code
shell command), which cuts short a long build or a review-gated `git commit`. The
`iswarm_task` tool, on the same `iswarm` MCP server, runs the command in a terminal tab of the
agent's own session instead and answers at once. When the command ends, a one-line notice is
typed into the agent's prompt, so the agent can end its turn and carry on when told.

- Up to 4 tasks run at once in a session.
- The tab closes when the command succeeds, and stays, titled with its exit code, when it
  fails. Closing a running task's tab asks first, then stops the task and everything it
  started.
- The output goes to a temporary log that keeps the first 1 MB and about the last 10 MB. It
  goes with its session, or once 50 newer tasks in the session have ended, and the OS may
  clear it sooner.
- Quitting the app while tasks run always asks first, then stops them. The next start tells
  each agent its task was interrupted. After a crash, on macOS and Linux, the next start
  also stops what it can prove a task left running in its terminal session, retrying at
  each start until that is done; a process that started a session of its own (`setsid`)
  has left it and is not stopped.
- Only the session's own agent can use it, and only through the server's plain-HTTP
  listener, the one its agents reach at 127.0.0.1, whether remote access binds it to
  loopback or to the network. A call through the HTTPS listener, where the relay's traffic
  arrives, is refused. An agent that reaches a server through an ssh tunnel to that server's
  plain-HTTP port can run tasks on that server's machine, so wire one only to machines you
  control.
- Approve or allowlist it like `iswarm_message`: `mcp__iswarm__iswarm_task` for Claude Code.

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

## Installing the server

For the newest full release, one line picks the archive for your machine,
checks it against the release's checksums, and installs it:

```bash
curl -fsSL https://github.com/agenticfabriq/internswarm/releases/latest/download/install.sh | sh
```

For a release candidate, or to pick the archive yourself, download the
server archive for your machine from that release on the
[Releases](https://github.com/agenticfabriq/internswarm/releases) page, with
its `.install.sh`, and on that machine run:

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
