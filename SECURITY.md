# Security

`signal-channel` mediates between a Claude Code session (an LLM with shell + filesystem access) and Signal (an end-to-end encrypted messaging network). This document explains the trust model and how to report security issues.

## Reporting vulnerabilities

**Don't file security issues as public GitHub issues.** Use [GitHub Security Advisories](https://github.com/bufothefrog/claude-signal/security/advisories/new) for private disclosure. If GitHub Security Advisories are disabled or unavailable, contact the maintainer directly via the email in the latest commit author field.

## Threat model

The bridge handles two kinds of trust:

1. **Inbound trust gate** (`access.json`) — decides which Signal senders can deliver messages to the Claude session. Default policy is `pairing`: unknown senders get a 6-character code and must be approved out-of-band by the user typing `/signal:access pair <code>` in their terminal.
2. **Outbound capability gate** — Claude can call any tool the MCP host allows it to call (`reply`, `update_profile`, etc.). The user's `~/.claude/settings.json` permission rules govern this.

The bridge does NOT defend against:
- A compromised Claude Code installation. If the LLM session is fully owned by an attacker, the bridge is just a transport — they can read your message history, send messages as you, and configure access however they like.
- A compromised host machine. Standard OS-level threats (root access, filesystem read, kernel attacks) trump any application-level controls.
- Signal-cli vulnerabilities. The bridge wraps signal-cli's JSON-RPC; any CVE in signal-cli is inherited.

## Pairing-code semantics

The 6-character codes that appear in `access.json` `pending[]` entries are **not credentials**. They're lookup keys for pending pairing requests, with these properties:

- TTL of 1 hour (entries auto-expire).
- Maximum of 3 simultaneous pending entries (rate-limit against spam-pairing).
- Maximum of 2 prompts sent per pending entry (one prompt code + one reminder for DMs; one prompt for groups).
- Unguessable in practice (3 bytes of `crypto.randomBytes` = 16M possibilities, 1-hour expiry, low rate of guesses available).

An attacker who knows a pending code but does NOT have terminal access cannot pair — `/signal:access pair <code>` requires the user to type it themselves, and the skill explicitly refuses pairings requested via Signal channel content (prompt-injection defense in `skills/access/SKILL.md`).

## OWNER bypass

The `SIGNAL_OWNER` environment variable identifies a "trusted account" whose messages bypass the pairing gate. **Setting `SIGNAL_OWNER` correctly is critical** — the default fallback to `currentAccount` (the bridge's own account number) is functionally a no-op for the bypass and silently breaks the assumed trust pattern (group-pairing prompts get routed to the bridge itself instead of to you).

Always set `SIGNAL_OWNER` to your primary device's UUID via `/signal:configure owner <uuid>` or directly in `.env`.

## File permissions

Sensitive files are written `0600` (owner-only) via atomic `chmod`-then-rename:
- `~/.claude/channels/signal/access.json` (allowlist + pending pairings)
- `~/.claude/channels/signal/.profile-set` (last-applied profile name marker)
- `~/.claude/channels/signal/.env` (environment overrides)

`messages.db` (SQLite) and `authors.json` use default permissions (`0644`). They contain message history and contact metadata; if your home directory is multi-user, lock down with `chmod 600 ~/.claude/channels/signal/{messages.db,authors.json}`.

## Static-mode deployments

Set `SIGNAL_ACCESS_MODE=static` for reproducible deploys where access state should be baked into the deploy artifact rather than mutated at runtime. In this mode:
- `access.json` is read once at boot and treated as read-only — pairings cannot be approved, allowFrom cannot be modified, no atomic-rename writes occur.
- `dmPolicy: pairing` and `groupPolicy: pairing` are downgraded to `allowlist` at boot (since pairing requires runtime mutation).
- All mutation-bearing tools (`update_profile`, `update_contact`, `block`, `trust`, etc.) throw with the standard `<tool> blocked: SIGNAL_ACCESS_MODE=static` message.

Useful for: container deployments where access state is provisioned at build time, or any setup where you want the running bridge to be incapable of altering its trust state.

## Boot-time stale-bridge cleanup

At startup the bridge scans for orphan `signal-cli ... jsonRpc` daemons whose parent is a `bun` process and kills both — this self-heals after a crash where the previous bridge didn't shut down cleanly. The match is account-agnostic and runs as the same Unix user as the new bridge.

If you run unrelated `bun` scripts that themselves spawn `signal-cli` daemons under the same user, those would be matched and killed by the cleanup. This is intentional (the bridge can't know which signal-cli daemons belong to it specifically) but worth knowing if you have a multi-tenant signal-cli setup. Run signal-cli daemons under a separate Unix user if you need isolation.

## What the bridge has access to

By design, the bridge can:
- Read every Signal message your account receives (via signal-cli's daemon).
- Send messages on your behalf to any contact or group.
- Read/modify your Signal profile (display name, about, avatar).
- See your full contact list.
- Join, leave, modify groups you're in.

These are the same capabilities your Signal mobile app has. The bridge is a remote-control surface for your own Signal identity.

## Privacy note

The bridge persists message history to `~/.claude/channels/signal/messages.db` (SQLite). Contents are NOT end-to-end encrypted at rest — Signal's E2E protection ends at signal-cli; what the bridge stores is plaintext on your local disk. If your threat model includes local-disk exposure (laptop theft, multi-user host), encrypt the home directory at the OS level (FileVault, LUKS, etc.).
