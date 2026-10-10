# Relay Trust Model

> Published from InternSwarm's code repository, which is not public yet, at 0e2bb50d. Anything about building from source applies there.

The InternSwarm hosted relay gives your server a public HTTPS address without
opening an inbound port on your machine, and carries agent mail between
machines. It routes your traffic and cannot read it. This page says what the
relay sees, what it stores, and what you still trust its operator with.

## Where TLS Terminates

For a relay address such as `https://server.relay.example.com`, TLS terminates
inside your `iswarm-server`. The relay reads the hostname a client asks for
(the TLS SNI) to route the connection, then forwards the encrypted bytes to
your outbound `iswarm-tunnel` sidecar.

So the relay does not see HTTP routes, WebSocket frames, server events, bearer
tokens, cookies, pairing codes, terminal output, file contents, diffs, prompts,
or API payloads.

## What The Relay Sees About Connections

- The SNI hostname.
- The client's IP address and port.
- Account, server, tunnel and relay region identifiers.
- Connection timing, duration, byte counts, close reasons and error
  categories.
- Tunnel health, reconnects, and quota or plan status.

This is enough to operate the service, enforce abuse limits, debug
reachability and bill for relay use. It is not enough to read your traffic
while the TLS key stays on your server.

## Your Hostname And Its Certificate

The relay gives your server a hostname under its own domain. Your server
generates the certificate's private key itself, sends only a certificate
signing request, and stores the certificate it gets back next to that key. The
relay never receives the key, so it cannot passively read your traffic.

**What you still trust the operator with:** it controls DNS for the relay's
domain, so it could obtain a second certificate for your hostname and stand in
for your server. That is an active attack, not passive reading, but nothing in
the relay design prevents it. If you cannot accept that, do not publish
through the relay: Server Admin's **My own domain** serves public HTTPS
directly, on a domain and certificate you control, with no relay involved.

## Agent Mail

Mail between sessions on one server never leaves the machine. Mail to another
of your machines, or to another person, travels through the relay, sealed.

**Sealed on the sending machine.** Each message is encrypted to the receiving
machine's encryption key (HPKE, RFC 9180: X25519, HKDF-SHA256,
ChaCha20-Poly1305) and signed with the sending machine's Ed25519 key. The
relay cannot read a message or change one without the change being caught: a
receiving machine refuses an envelope whose signature fails or whose recipient
fields do not match it, before opening it.

**The relay hands out keys and vouches for none of them.** You verify a
teammate's machine by comparing eight words over a channel you already trust;
your own machines attached to one app verify each other. A machine whose key
changes gets none of your mail until you verify it again. Mail from another
person waits in **Held Mail** for someone on the receiving side to decide
about, unless it is a reply or you have granted it a way through; see
[Messaging a teammate's agents](teammates.md).

**What is sealed:** the message text and the name of the session that sent it
(such as `api#3`).

**What the relay sees of each envelope:**

- What kind of envelope it is: a message, a receipt (below), or one of the
  notices machines send each other about a message: that it was let in,
  answered on another machine, or withdrawn.
- The sending and receiving account's email address.
- The sending and receiving machine's name.
- The sending session's id and, for mail addressed to one session, the
  receiving session's id. These are opaque ids, never session names. An empty
  receiving session shows the relay that the mail was addressed to a person
  rather than to one of their sessions.
- The channel names, which key the message was sealed to, a message id, a
  per-machine sequence number, when it was sent and when it expires, its size,
  and when it passed through.

Mail addressed to a person goes to each of their machines as a separate copy.
The copies carry the same sequence number, so the relay can tell they are one
message.

**Receipts.** The receiving machine tells the sending machine what became of
each message: stored, held for a person, let in, declined, or refused. A
receipt is sealed and signed like a message when the receiving machine holds
a usable encryption key for the sender. When it holds none, or sealing fails, the
receipt travels signed but readable: the relay then sees which
message it answers, both machines, the time and the outcome.

**What the relay keeps about your team:** each member's email address, display
name and role, and each member's machines with their hostnames, regions and
public keys. This directory is how machines find each other.

**Mail for a machine that is not connected waits on the relay**, still sealed,
with its recipient, its sending machine, its message id, its size and when it
arrived. A machine can have up to 256 messages waiting, each up to 240 KiB.
Every message expires at most 24 hours after it was sent, and the receiving
machine refuses it after that. The relay deletes waiting mail when the machine
collects it, when the machine is revoked (along with mail that machine sent
and that is still waiting), and when its owner is removed from the team.

## The Tunnel Sidecar

`iswarm-tunnel` is a separate process from `iswarm-server`. It receives the
relay URL, the server id, the relay hostnames, the local HTTPS address it
forwards to, the certificate fingerprint, and this machine's identity key
pair. That key proves the machine to the relay, and it is also the key that
signs this machine's agent mail.

It does not receive the key that opens mail sealed to this machine, so mail
passes through it sealed. Nor does it receive workspace paths, app cookies,
app auth tokens, pairing codes, terminal state, file contents, git data,
prompts, diffs, or the TLS private key. If the sidecar stops, relay
reachability stops, but `iswarm-server` and its sessions keep running.

## When Not To Use The Relay

The relay is optional. Localhost, LAN, VPN, Tailscale and direct HTTPS on your
own domain all work without it, when your server settings allow them. Use
them if your environment does not allow the metadata above to leave it, if you
cannot accept the DNS trust described above, if a full-tunnel VPN blocks relay
traffic, or if your organization requires its own network access controls.
Mail to another person, or to another of your machines, travels only through
the relay today.
