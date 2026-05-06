# signal-channel

Claude Code channel plugin that talks to Signal via signal-cli.

## What it is

Inbound Signal messages arrive in your Claude Code session as channel events; outbound goes through the `reply` tool. DMs, groups, attachments, edits, and reactions all flow through the same bridge. Read receipts, reaction events, group metadata changes, and linked-device contact-sync updates also surface as structured channel events, so Claude can reason about thread state ("read 3 minutes ago, still no reply") and not just message content. One Signal account, one bridge process.

## Prereqs

- [signal-cli](https://github.com/AsamK/signal-cli). Latest stable or later, on `PATH`.

  <details>
  <summary>Install</summary>

  - macOS: `brew install signal-cli`
  - Arch: `pacman -S signal-cli`
  - Debian/Ubuntu, RedHat/Fedora: no official package. Install the GraalVM native build system-wide from the upstream tarball (self-contained binary, skip the Java prereq):

    ```sh
    VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
    curl -L -O https://github.com/AsamK/signal-cli/releases/download/v"${VERSION}"/signal-cli-"${VERSION}"-Linux-native.tar.gz
    sudo tar xf signal-cli-"${VERSION}"-Linux-native.tar.gz -C /opt
    sudo ln -sf /opt/signal-cli /usr/local/bin/
    ```

  </details>

- [bun](https://bun.sh). Latest stable or later. Runs the MCP server.

  <details>
  <summary>Install</summary>

  - Debian/Ubuntu, RedHat/Fedora, macOS: `curl -fsSL https://bun.sh/install | bash`
  - macOS (Homebrew): `brew install bun`
  - Arch: `pacman -S bun`

  </details>

- Java 21+ JRE. Required by signal-cli (skip if you installed the GraalVM native build above).

  <details>
  <summary>Install</summary>

  - Debian/Ubuntu: `apt install openjdk-21-jre`
  - RedHat/Fedora: `dnf install java-21-openjdk`
  - Arch: `pacman -S jre-openjdk`
  - macOS: `brew install openjdk@21` or [adoptium.net](https://adoptium.net)

  </details>

## Account setup

The bridge does not register or link Signal accounts for you. Pick one of these paths:

<details>
<summary>Link as a secondary device (existing Signal account)</summary>

If you already use Signal on a phone, link the bridge as a secondary device:

```sh
signal-cli link -n "claude-bridge"
```

This prints a `sgnl://linkdevice...` URL. Render it as a QR code and scan with the Signal app on your phone (Settings → Linked devices → Link new device).

Once linked, the bridge auto-detects the account on first launch.

</details>

<details>
<summary>Headless registration (fresh number, no primary phone)</summary>

For a fresh, dedicated Signal account on a server or container, register signal-cli directly:

```sh
signal-cli -u +<PHONE_NUMBER> register
signal-cli -u +<PHONE_NUMBER> verify <CODE_FROM_SMS>
```

If `register` errors with `Captcha required`, Signal wants a one-time challenge:

1. Open <https://signalcaptchas.org/registration/generate.html> in a browser. signal-cli's docs recommend doing this from the same network/IP as the host.
2. Solve the captcha. When the "Open Signal" link appears, right-click and copy its URL; it starts with `signalcaptcha://signal-recaptcha-v2...`.
3. Re-run register with the token, unquoted:

   ```sh
   signal-cli -u +<PHONE_NUMBER> register --captcha signalcaptcha://...
   ```

   The token expires within a few minutes, so paste it quickly.

Notes:

- Cryptographic entropy can be a problem on minimal containers and embedded systems. If `register` hangs or fails, install `haveged` first.
- Verify with a test send before launching the bridge: `signal-cli -u +<PHONE_NUMBER> send -m "test" +<RECIPIENT>`.

</details>

## Install

Add the marketplace, install the plugin:

```
/plugin marketplace add bufothefrog/claude-signal
/plugin install signal@claude-signal
```

Then relaunch Claude Code with the channels flag:

```sh
claude --channels plugin:signal@claude-signal
```

If exactly one signal-cli account is linked, the bridge auto-detects it. If multiple are linked, point it at the one you want:

```
/signal:configure account +15551234567
```

Run `/signal:configure` with no arguments to see all toggles and their current values, or copy [`.env.example`](./.env.example) to `~/.claude/channels/signal/.env` for an annotated reference of every setting.

## Profile name

Set a Signal display name via `/signal:configure profile-name <name>` or `SIGNAL_PROFILE_NAME=<name>` in `.env`. The bridge applies it on first boot and tracks the last-applied value at `~/.claude/channels/signal/.profile-set` so it never overwrites a manually-changed profile on subsequent boots. Delete the marker if you ever need to force a re-apply. Empty by default (no auto-set).

## Access control

Default policy is **pairing**. Send a Signal message to your account; the bridge replies with a 6-character code. In your Claude Code terminal, run:

```
/signal:access pair <code>
```

The sender is now in `allowFrom` and future messages flow through directly.

Groups follow the same pattern. When the bridge is added to a group, it DMs the owner with a code; the owner runs `/signal:access group pair <code>` to enable routing. Group messages only deliver when they mention the bridge (by Signal's @-picker or a regex over text).

DMs and groups have independent policies (`dmPolicy`, `groupPolicy`), each with three modes:

- **pairing** (default): prompt for approval
- **allowlist**: only known senders; unknowns dropped silently
- **disabled**: drop all traffic

Run `/signal:access` to inspect current state.

## Tools

| Tool | Purpose |
| --- | --- |
| `reply` | Send a Signal message. `chat_id` is a UUID (DM) or `group:<base64>`. Supports `reply_to`, `files`. |
| `edit_message` | Edit a message the bridge previously sent. `message_id` is the timestamp returned by `reply`. |
| `react` | Add an emoji reaction to a message by id. |
| `typing` | Show or stop a typing indicator in a chat. |
| `remote_delete` | Delete a previously-sent message on the recipient's side. |
| `chat_messages` | Query persistent message history. Filter by `chat_id`, `since`/`until`, `search`, `limit`. |
| `list_contacts` | List signal-cli contacts. Optional `match` substring filter. |
| `list_groups` | List Signal groups the bridge belongs to. Optional `match` filter on title/id/description. |
| `list_identities` | List identity records (safety numbers, trust levels). Optional `number` filter. |
| `mark_read` | Send a read receipt for a previously-received message. |
| `get_user_status` | Check if a phone/UUID/username is registered on Signal. Validates a `chat_id` before sending. |
| `get_attachment` | Re-fetch an attachment after signal-cli has garbage-collected it. Writes to the canonical attachments path channel events already point to. |
| `trust` | Trust a contact's identity after their safety number changes. |
| `block` / `unblock` | Block (or unblock) a contact or group. |
| `update_contact` | Update local-only contact fields (nickname, note, disappearing-message expiration). |
| `remove_contact` | Remove a contact. Mutually exclusive `hide` (reversible) / `forget` (wipes identity keys). |
| `join_group` | Join a Signal group via invite link. Returns the new `groupId`. |
| `quit_group` | Leave a group. Optional `delete` clears local state; optional `admins` transfers admin if last admin. |
| `update_group` | Rename / change description / membership / admins / permissions on a group the bridge admins. |
| `update_profile` | Update the bridge account's own profile (`given_name`, `about`, `avatar`, etc.). |

## Channel events

Inbound traffic surfaces as `notifications/claude/channel` events distinguished by `meta.event_type`. All events share `chat_id`, `message_id`, `user`, and `ts` in `meta`; the type-specific fields are listed below.

| `event_type` | When it fires | Extra `meta` fields |
| --- | --- | --- |
| _(none, default)_ | A regular DM, group message, edit, attachment, or own-account sync | `file_path` (when an attachment is present), `edited` (when an edit) |
| `receipt` | A recipient marked one of our outbound messages read or viewed (delivery receipts are filtered out as too noisy) | `receipt_type` (`read` \| `viewed`), `target_timestamps` (csv of acked outbound timestamps) |
| `reaction` | Someone added or removed an emoji reaction on a message in a chat the bridge participates in | `emoji`, `target_author`, `target_sent_timestamp`, `is_remove` |
| `group_update` | Group metadata changed (rename, member add/remove, permission change) | `group_id`, `group_name`, `revision` |
| `contact_update` | A linked device pushed a contacts-sync blob (e.g. user renamed a contact on their phone). Re-fetch via `list_contacts` for the up-to-date state | _(none beyond the shared fields)_ |

Reactions land in `messages.db` with empty `text`, so `chat_messages` `search` won't surface them by content. However, `chat_messages` filtered by `chat_id` returns them in chronological context alongside the messages they target.

## History and search

Inbound and outbound messages are persisted to `~/.claude/channels/signal/messages.db` (SQLite) as they flow. An author index at `~/.claude/channels/signal/authors.json` tracks display names, first/last seen, and message counts per sender. Cross-session `react` works because the bridge looks up the original author from SQLite, not memory.

**Install-time limitation:** messages from before v0.3 install are not in the cache. signal-cli has no `listMessages` command, so historical bootstrap isn't possible. Capture is reactive from the moment the upgraded bridge starts.

**Opt out:** run `/signal:configure history off` (or set `SIGNAL_DISABLE_HISTORY=true` in `.env`) to skip the database entirely. Live message routing keeps working and the `authors.json` display-name cache keeps tracking, but `chat_messages` / `react` / `mark_read` start throwing a clear "history disabled" error. Useful for privacy-conscious deploys and ephemeral / stateless containers.

## Auto read-receipts (optional)

By default, a sender doesn't know the bridge saw their message until Claude calls `mark_read` (or sends a reply). Flip this with:

```
/signal:configure auto-receipts on
```

(Or directly: `SIGNAL_AUTO_READ_RECEIPTS=true` in `.env`.) Every inbound message gets an automatic read receipt as it lands, before Claude has even responded. Own-account syncMessages and message edits are skipped. Think of it as a presence signal: useful for chats where "seen but not yet replied" is friendlier than radio silence, but a privacy regression for chats where you want Claude to choose what to acknowledge.

## Permission relay

Claude can authorize tool calls via Signal. When Claude needs to run something sensitive (Bash, Write, etc.), a 🔐 prompt arrives at the owner's Signal chat. Reply `yes <code>` or `no <code>` to allow or deny.

Owner defaults to the linked account. Override with:

```
/signal:configure owner +15551234567
```

## Running long-lived

The plugin lives and dies with your Claude Code session. To keep it alive, use your OS's standard process manager: `tmux` / `screen` / `zellij` for cross-platform terminal multiplexing, or a service manager like `launchd` (macOS), `systemd` (Linux), or Task Scheduler (Windows) for unattended operation. Run `/signal:configure service` for a ready-to-go systemd user unit.

## Troubleshooting

Bridge fails to start: run `signal-cli -a $SIGNAL_ACCOUNT receive` from a terminal to see the actual error. Usually it's an account lock or a missing link.

Upstream signal-cli issues: file at [AsamK/signal-cli](https://github.com/AsamK/signal-cli/issues).

Plugin issues: file at this repo's issues.

## License

AGPL-3.0. See [LICENSE](./LICENSE).
