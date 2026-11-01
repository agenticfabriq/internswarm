# InternSwarm

A console for coding agents on machines you own. Run Claude Code, Codex or any
other terminal agent in sessions a server owns, see at a glance which ones are
working, asking or finished, and answer them from your phone.

InternSwarm works with Claude Code, Codex and Antigravity; it is not made or
endorsed by Anthropic, OpenAI or Google.

This repository holds InternSwarm's releases and its issue tracker. It
contains no source code.

## Download

Every version is on the [Releases](https://github.com/agenticfabriq/internswarm/releases)
page: the desktop app for macOS on Apple silicon (`.dmg`), and the server for
macOS and Linux.

Release candidates are marked as pre-releases. The release named
`candidates` is not a version to install: it holds the list the desktop app
reads to offer the next release candidate.

## Documentation

- [Operating InternSwarm](docs/operating.md): installing the server, the
  desktop app, network access and the hosted relay.
- [Multiple accounts](docs/multi-account.md): more than one Claude Code or
  Codex account on a machine, with usage and messaging for each.
- [Set up InternSwarm with your coding agent](docs/agent-setup-prompt.md): a
  prompt that has your agent write that setup for you.
- [Messaging a teammate's agents](docs/teammates.md): reaching another
  person's agents through the hosted relay.
- [Relay trust model](docs/relay-trust-model.md): what the hosted relay can
  see and keeps, agent mail included, and what you still trust it with.
- [Supported coding agents](docs/supported-agents.md): what each agent does
  in InternSwarm, and the setup a few features take.

## Report a problem

Open an [issue](https://github.com/agenticfabriq/internswarm/issues/new/choose).
A security problem is reported privately instead: see [SECURITY.md](SECURITY.md).
