# Security

## Reporting a vulnerability

Report it privately, through GitHub's private vulnerability reporting:
this repository's **Security** tab, then **Report a vulnerability**
(<https://github.com/agenticfabriq/internswarm/security/advisories/new>).
Please do not open a public issue for it.

Include what you can of:

- the version, from the desktop app or `iswarm-server --version`;
- how to reproduce it, and what an attacker gains;
- whether it needs the hosted relay, a network position, or a signed-in user.

The conversation continues in that private advisory, and a fix is released
before the advisory is published.

## Scope

The desktop app, `iswarm-server`, the `iswarm-tunnel` relay client, the
installer and the release builds. A vulnerability in the hosted relay service
itself can be reported the same way.

## Supported versions

The latest release. Fixes are not backported to earlier ones.
