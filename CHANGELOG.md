# Changelog

All notable changes to `signal-channel` are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning follows [SemVer](https://semver.org/).

## [1.0.0] - 2026-05-03

Promotion to v1.0 after hands-on validation: group pairing flow verified end-to-end with real Signal traffic (DM-to-OWNER prompt routing, code-keyed approval, structured @-mention detection), `dmPolicy`/`groupPolicy` independence working as designed, dynamic profile-name mention pattern surviving runtime renames. Versioning per the v0.6 versioning decision: v1.0 is gated on hands-on confidence, not feature-completeness alone. The work shipped in this release was developed and tagged internally as v0.7; promoting directly to v1.0 because no other gating criteria remain.

Theme: group pairing + smarter mention handling. The DM pairing flow now extends to groups — adding the bridge to an unknown group with `groupPolicy: pairing` (default) DMs the owner with a 6-char code, mirroring the v0.1 DM pairing UX. Also fixes a longstanding mismatch between Signal's @-mention UI and the gate's mention check that was silently dropping every Signal-UI-picked mention.

### Added

- `groupPolicy: 'pairing' | 'allowlist' | 'disabled'` field in `access.json`, default `'pairing'`. Parallel to `dmPolicy`. Decouples group access policy from DM access policy. `pairing` = unknown group → bridge DMs owner with a 6-char code; `allowlist` = drop unknown silently (matches v0.6 behavior; opt-out for users who want the old shape); `disabled` = drop all group traffic, even from known `groups[]` entries.
- `'group_pair'` `GateResult` variant. Gate mints a `kind: 'group'` pending entry on first non-owner inbound from an unknown group; deduped by `chatId` so a busy group doesn't spam owner DMs. Dispatch sends a one-shot prompt DM to OWNER.
- `kind: 'dm' | 'group'` discriminator on pending entries. Backwards-compat: missing field defaults to `'dm'` on read; v0.6 entries acquire the field on next save naturally — no migration write.
- `/signal:access group pair <code>` subcommand. Sets `groups[chatId] = { requireMention: true, allowFrom: [] }` (matches `group add` defaults), clears the pending entry. No `approved/` file is written — there's no in-band sender to acknowledge; the terminal confirmation IS the entire UX.
- `/signal:access group deny <code>` subcommand. Drops pending entry silently. Bridge stays in the group at the signal-cli level — denial blocks routing only, not membership.
- `/signal:access group policy <pairing|allowlist|disabled>` subcommand. Sets `groupPolicy`. Mirrors the existing `policy` subcommand for `dmPolicy`.
- Defensive `kind` checks in `/signal:access pair`/`deny`/`group pair`/`group deny`. A typo'd code that matches the wrong kind surfaces a "use the right subcommand" message instead of mutating state under the wrong assumption.
- Static-mode downgrade for `groupPolicy: 'pairing'` → `'allowlist'` at boot under `SIGNAL_ACCESS_MODE=static`, parallel to the existing `dmPolicy` treatment.
- **Structured Signal @-mention detection.** signal-cli renders `@<picked-contact>` as a single U+FFFC object-replacement character (`￼`) in the message text and surfaces the actual identity in `data.mentions[]` (each entry: `uuid`/`number`/`name`/`start`/`length`). v0.7's gate now matches `mentions[].number === currentAccount` or `mentions[].uuid === currentAccount` *before* falling back to text-pattern matching. Discovered during v0.7 smoke testing: a friend's `@OpenClaw test` (rendered as `￼ test`) was being silently dropped by the gate because no regex pattern matched the literal `￼`. The new check is identity-based and survives display-name changes.
- **Implicit `@<PROFILE_NAME>` text-pattern fallback.** Adds the bridge's current profile name as an always-on regex pattern alongside any explicit `mentionPatterns` from `access.json`. Refreshes at runtime when `update_profile` mutates `given_name`. Lets users mention the bridge with literal `@OpenClaw` (any case) text without per-rename `access.json` updates. Useful for typed `@bot` triggers and for messages that don't go through Signal's @-picker UI.

### Changed

- `dmPolicy: 'disabled'` is now DM-only. v0.6 implicitly dropped both DMs and groups when `dmPolicy === 'disabled'` because the check sat above the `isGroup` branch in `gate()`. v0.7 splits them: each policy is independent. To kill all traffic, set both `dmPolicy: 'disabled'` *and* `groupPolicy: 'disabled'`. v0.6 users who relied on `dmPolicy: 'disabled'` as a global kill switch should set `groupPolicy: 'disabled'` after upgrading.
- Pending pool is now mixed (DM + group entries). Lookup loops in `gate()` filter by `kind` to avoid cross-matching a DM `senderId` against a group entry that happens to share the same UUID.

### Notes

- v0.7 surfaced a pre-existing config trap: when `SIGNAL_OWNER` is unset in `.env`, OWNER falls back to `currentAccount` (the bridge's own account number). This silently breaks the OWNER bypass for any sender whose identifier doesn't match the bridge — and v0.7's group-pairing prompt then gets DMed to the bridge itself (a black hole). Existing v0.6 users with explicit `allowFrom` UUIDs probably wouldn't notice, since DMs still hit `allowFrom` before the bypass mattered. Recommended: explicitly set `SIGNAL_OWNER` to your phone's UUID via `/signal:configure owner <uuid>` (or directly in `.env`). A future polish item is a stderr warning at boot when `SIGNAL_OWNER` is unset.

## [0.6.0] - 2026-05-01

Theme: competent participant. Closes the remaining Tier A gap from the gap analysis — every signal-cli capability that maps cleanly to an unattended LLM is exposed. After v0.6 the bridge can manage groups, contacts, and its own profile without you reaching for `signal-cli` directly, and inbound reactions / group changes / linked-device contact updates surface as structured events rather than empty channel messages or silent drops. Versioned at 0.6 (not 1.0) deliberately — v1.0 is gated on a soak period and hands-on confidence, not feature-completeness alone.

### Added

- `update_profile` MCP tool. Updates fields on the bridge account's own Signal profile: `given_name`, `family_name`, `about`, `about_emoji`, `mobile_coin_address`, `avatar` (file path), `remove_avatar` (boolean). `avatar` and `remove_avatar` are mutually exclusive. Reuses the camelCased CLI-flag pattern signal-cli's `updateProfile` JSON-RPC expects (e.g. `givenName`, `aboutEmoji`). Throws if `SIGNAL_ACCESS_MODE=static`.
- `update_contact` MCP tool. Local-only contact-field updates: `given_name`, `family_name`, `nick_given_name`, `nick_family_name`, `note`, `expiration` (disappearing-message seconds; 0 disables). Required `chat_id`. Uses the `singleRecipientParams` helper because signal-cli's `updateContact` takes a single-string `recipient`, not an array. Local to this account; does not propagate. Throws if `SIGNAL_ACCESS_MODE=static`.
- `remove_contact` MCP tool. Required `chat_id`. Mutually exclusive `hide` / `forget`: `hide` keeps history but drops from contact list (reversible); `forget` wipes all local data including identity keys (irreversible without re-pairing). Default (neither flag) just clears profile/contact info. Throws if `SIGNAL_ACCESS_MODE=static`.
- `join_group` MCP tool. Required `uri` (signal.group/#... invite link). Returns the new `groupId` for immediate use with `list_groups`, `reply`, `update_group`, `quit_group`. Throws if `SIGNAL_ACCESS_MODE=static`.
- `quit_group` MCP tool. Required `group_id` (accepts either bare base64 or `group:<base64>`). Optional `delete=true` removes local group state. Optional `admins` array transfers admin to those members if the bridge is the last admin. Throws if `SIGNAL_ACCESS_MODE=static`.
- `update_group` MCP tool. Largest schema in v0.6. Required `group_id`. Identity fields: `name`, `description`, `avatar`, `expiration`. Membership: `members`, `remove_members`, `admins`, `remove_admins`, `banned`, `unbanned` (arrays of UUIDs/phones/usernames). Permissions: `link` (`enabled` | `enabled-with-approval` | `disabled`), `permission_add_member` / `permission_edit_details` / `permission_send_messages` (`every-member` | `only-admins`). Throws if `SIGNAL_ACCESS_MODE=static`.
- `get_attachment` MCP tool. Required `attachment_id` and `chat_id`. Re-fetches an attachment from Signal's servers and writes it to the bridge's canonical `<config>/attachments/<id>` path — the same path channel-event `file_path` metadata already points to, so subsequent reads continue to work. Closes the data-loss case where signal-cli garbage-collects the file before Claude reads it. Read-only (no `STATIC` gate).
- Reaction envelope routing. Inbound reactions arrive as a `dataMessage.reaction` field; v0.6 surfaces them via the canonical `notifications/claude/channel` method with `meta.event_type='reaction'` and `meta.{emoji, target_author, target_sent_timestamp, is_remove}` (all string-typed — see Fixed below). Follows the same canonical-method-with-discriminator pattern v0.5 settled on for receipts.
- Group-update envelope routing. `dataMessage.groupInfo.type='UPDATE'` (rename, member changes, permission changes) emits with `meta.event_type='group_update'` and `meta.{group_id, group_name, revision}`. `type='DELIVER'` (normal group messages) keeps the existing routing.
- Contact-sync envelope routing. `syncMessage.contacts` (linked-device contacts blob) emits with `meta.event_type='contact_update'` and a synthetic `chat_id='sync'`. The actual contact diff is delivered as an attachment blob signal-cli decodes internally; the event tells Claude to re-fetch via `list_contacts` if the up-to-date state matters.

### Fixed

- Reaction-only and groupInfo-only `dataMessage` envelopes no longer emit empty-content channel notifications. Pre-v0.6, `onEnvelope`'s generic notify path fired for any `dataMessage` that wasn't a typing indicator — including reaction-only and metadata-only ones — producing channel events with empty `content`. The new structured routings short-circuit those paths with their own discriminated events.
- Channel notification `meta` values are now uniformly stringified. Discovered during v0.6 smoke testing: the Claude Code channel router silently drops `notifications/claude/channel` events whose `meta` contains numbers, booleans, or other non-string values — no error surfaces, the event simply doesn't reach the session. v0.5's receipt routing satisfied this by accident (CSV-joining number arrays before emit). v0.6's reaction routing first shipped with `target_sent_timestamp: <number>` and `is_remove: <boolean>`; smoke testing confirmed the bridge processed envelopes correctly but no events surfaced. Stringifying both fields (and `revision` in `group_update`) immediately resolved it. Documented as a load-bearing quirk in `.claude/CLAUDE.md`.
- `get_attachment` semantics clarified. The tool successfully resolves attachment metadata when the local cached file is present, but signal-cli does not appear to re-fetch from Signal's CDN once the local file is moved or deleted (returns "Could not find attachment with ID"). The "GC recovery" framing in the original v0.6 design is therefore overstated — the tool is closer to "verify cached file is still resolvable" than "recover deleted bytes." Useful within the cache window; less useful for true post-GC recovery.

## [0.5.0] - 2026-04-30

Theme: assistant-defense + thread-state observability. The bar is "safe to leave running unattended for a month" — when something goes sideways (spam wave, identity reset, accidental outbound, message someone read but didn't reply to), Claude has the tools and visibility to handle it without you reaching for `signal-cli` yourself.

### Added

- `block` / `unblock` MCP tools. Wrap signal-cli's `block` / `unblock` JSON-RPC. Required `chat_id` (UUID/phone for a contact, `group:<base64>` for a group). Both throw if `SIGNAL_ACCESS_MODE=static`.
- `get_user_status` MCP tool. Wraps `getUserStatus`. Lets Claude validate a `chat_id` is registered on Signal before a `reply` errors out with an opaque "unknown recipient". Read-only.
- `list_identities` MCP tool. Wraps `listIdentities`. Optional `number` filter. Returns identity records with `safetyNumber`/`scannableSafetyNumber`/`trustLevel`/`addedTimestamp`. Useful for inspecting a contact's new safety number after a phone change.
- `trust` MCP tool. Wraps `trust`. Required `chat_id`; provide either `trust_all_known_keys=true` or `safety_number=<verified>`. Resolves the `UntrustedIdentity` failures that v0.4's `error.data` surfacing made visible. Throws if `SIGNAL_ACCESS_MODE=static`.
- `remote_delete` MCP tool. Wraps `remoteDelete`. Required `chat_id` and `target_timestamp`. "Oops" recovery for a bad outbound — deletes the message on the recipient's side. Throws if `SIGNAL_ACCESS_MODE=static`.
- Receipt envelope routing. signal-cli emits `receiptMessage` envelopes when a recipient acknowledges our outbound (delivered / read / viewed); v0.4 dropped these at the early-return in `onEnvelope`. v0.5 surfaces them via the canonical `notifications/claude/channel` method with `meta.event_type='receipt'` and `meta.receipt_type='read'|'viewed'`. (Initially shipped as a separate `notifications/claude/channel/receipt` method, but empirical testing showed the Claude Code host filters non-canonical channel sub-methods even when the matching capability is declared. Reverted to the canonical-method-with-discriminator pattern; v0.6's reaction/group_update/contact_update routings will follow the same shape.) Filtered to read/viewed only — delivery receipts fire 1-2× per outbound and would pollute the channel with no marginal value. Lets Claude reason about thread state.
- `singleRecipientParams` helper at `server.ts:528-532`. Sibling to the existing `recipientParams`, returns `{recipient: chatId}` (single string) instead of `{recipient: [chatId]}` (array) — matches the shape commands like `trust` and v0.6's `update_contact` / `remove_contact` / `get_attachment` require.

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
