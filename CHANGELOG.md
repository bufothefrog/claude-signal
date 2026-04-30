# Changelog

All notable changes to `signal-channel` are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning follows [SemVer](https://semver.org/).

## [0.4.1] - 2026-04-29

### Added
- `/signal:configure` subcommands for behavior toggles: `auto-receipts on|off`, `signature on|off`, `profile-name <name>`, `profile-name clear`. Discoverable from the same skill that configures account/owner.
- `/signal:status` Settings section surfacing current values of profile name, auto read-receipts, append signature, and access mode — each labeled with its source (`default`/`.env`/`env`).
- `CHANGELOG.md` and `.env.example` for distribution.
- `.github/ISSUE_TEMPLATE/` with bug-report and feature-request templates.

### Fixed
- `package.json` version was stuck at `0.1.0` despite four point releases in the plugin manifest. Now tracks `plugin.json`.

## [0.4.0] - 2026-04-29

### Added
- `list_groups` MCP tool. Wraps signal-cli's `listGroups` JSON-RPC. Optional `match` substring filter on title/id/description. Closes the v0.3 gap where Claude could only message groups it had already seen inbound traffic from.
- `SIGNAL_AUTO_READ_RECEIPTS` env var (default off). When enabled, every inbound message gets an automatic read receipt as it lands; useful for chats where "seen but not yet replied" is friendlier than radio silence. Skips own-account syncMessages and edits.

### Changed
- `rpc` error path now includes signal-cli's `error.data` payload, exposing the actual `SendMessageException` cause (untrusted identity, missing profile key, recipient-not-registered) instead of the opaque `error.message` ("Failed to send message").
- README "running long-lived" section reframed cross-platform (tmux, screen, launchd, systemd, Task Scheduler) so the plugin no longer implies systemd is canonical. The systemd unit at `contrib/systemd/claude-signal.service` is now framed as a community-contributed example.

## [0.3.1] - 2026-04-29

### Fixed
- `mark_read` failed with "Failed to send message" because `recipient` was wrapped in an array. signal-cli's `SendReceiptCommand.java` defines recipient as a single positional (no `nargs`), unlike `send` which is multi-recipient. Fix: pass the sender's UUID as a string. Also corrected the `react` tool description from "in-memory map" (v0.2 wording) to "persistent message log" (v0.3 sqlite-backed lookup).

## [0.3.0] - 2026-04-29

### Added
- Persistent message history at `~/.claude/channels/signal/messages.db` (sqlite via `bun:sqlite`). Inbound messages captured from `onEnvelope` after the access gate; outbound captured from a single `recordSent` funnel at every `rpc('send')` site. Schema versioned via `PRAGMA user_version`.
- Persistent author cache at `~/.claude/channels/signal/authors.json`. Write-coalesced (5s debounce + flush on exit). Replaces the in-memory `messageAuthors` Map — `react` and quote-replies now survive bridge restarts via sqlite lookup.
- `chat_messages` MCP tool. Time-window + `LIKE`-based substring search. Optional `chat_id` scope, default limit 50 / max 500. Tool description explicitly documents the install-time history limitation (signal-cli has no `listMessages` JSON-RPC command).
- `list_contacts` MCP tool. Wraps signal-cli's `listContacts`. Optional `match` substring filter.
- `mark_read` MCP tool. Wraps `sendReceipt` with `type: 'read'`. Routes to original sender via the messages table, so groups work transparently.
- `/signal:status` enrichment: total message count, in/out split, distinct chats, top 5 authors by message count, plus `messages.db` and `authors.json` file metadata.

## [0.2.0] - 2026-04-29

### Added
- `cleanupStaleBridges()` runs at boot, finds orphan signal-cli daemons whose parent bun process is also still alive, and kills the pair before attempting to acquire the lock. Self-healing across crashes.
- Username recipient routing. `recipientParams` now detects `nickname.discriminator` patterns (regex `^[a-z][a-z0-9_]{2,31}\.\d{2,}$`) and routes through signal-cli's `username` field.
- Auto-set profile name on first boot (default `OpenClaw`, configurable via `SIGNAL_PROFILE_NAME`). Marker file at `~/.claude/channels/signal/.profile-set` tracks last-applied value so the bridge only re-applies on change.

### Fixed
- Orphan bridge on Claude Code `/exit`. The MCP SDK's stdio transport doesn't notice an EOF on stdin; we now hook `process.stdin.on('end'/'close')` in addition to `mcp.onclose` so transport-EOF triggers a clean shutdown of both bun and the signal-cli child within ~2s.
- Lockfile-collision fail-fast. signal-cli's stderr is now piped and parsed; on `Config file is in use, waiting…` we kill the child and `exit(2)` within tens of milliseconds instead of blocking indefinitely.

## [0.1.0] - 2026-04-28

### Added
- Initial release. Single-file MCP bridge in `server.ts` spawning `signal-cli -a $account jsonRpc`.
- Inbound DMs, group messages, edits, and syncMessages flow as `notifications/claude/channel` events.
- Outbound via `reply`, `edit_message`, `react`, `typing` MCP tools. Text chunking, attachment paths, quote-replies.
- Access control with `dmPolicy: pairing|allowlist|disabled`, group policies, atomic state at `~/.claude/channels/signal/access.json`.
- Permission relay — Claude tool-use prompts forwarded to the owner's DM; `yes/no <code>` replies consumed before the access gate.
- Three skills: `/signal:configure`, `/signal:access`, `/signal:status`.
- Process resilience — unhandled rejection/exception handlers, signal-cli auto-respawn with backoff.
- `assertSendable` blocks bridge state-dir paths from `reply`'s `files` argument.
