# signal-channel

claude code channel plugin that talks to signal via signal-cli.

## what it is

inbound signal messages arrive in your claude code session as channel events; outbound goes through the `reply` tool. dms, groups, attachments, edits, and reactions all flow through the same bridge. one signal account, one bridge process.

## prereqs

- [signal-cli](https://github.com/AsamK/signal-cli) — latest stable or later, on `PATH`.
- [bun](https://bun.sh) — latest stable or later. runs the MCP server.
- java 21+ JRE — signal-cli's requirement.

## account setup

link or register signal-cli following [their guide](https://github.com/AsamK/signal-cli/blob/master/README.md) before installing this plugin. the bridge does not register or link accounts for you.

## install

```
/plugin install signal@<your-marketplace>
```

then relaunch claude code with the channels flag:

```sh
claude --channels plugin:signal@<your-marketplace>
```

if exactly one signal-cli account is linked, the bridge auto-detects it. if multiple are linked, point it at the one you want:

```
/signal:configure account +15551234567
```

run `/signal:configure` with no arguments to see all toggles and their current values, or copy [`.env.example`](./.env.example) to `~/.claude/channels/signal/.env` for an annotated reference of every setting.

## profile name

freshly registered signal-cli accounts have no profile, so messages from them show up as "unknown contact" until profile keys are exchanged. on first boot the bridge sets the profile name to `OpenClaw` for you. change it via:

```
/signal:configure profile-name Claude
```

or directly in `~/.claude/channels/signal/.env` as `SIGNAL_PROFILE_NAME=Claude`. set the env var to empty to disable. the bridge writes a marker file at `~/.claude/channels/signal/.profile-set` and only re-applies if the configured name changes — your signal profile won't be silently overwritten. delete the marker if you ever need to force a re-set.

## access control

default policy is **pairing**. send a signal message to your account; the bridge replies with a 6-character code. in your claude code terminal, run:

```
/signal:access pair <code>
```

the sender is now in `allowFrom` and future messages flow through directly. see [ACCESS.md](./ACCESS.md) for the full subcommand reference, group config, and the `access.json` schema.

## tools

| tool | purpose |
| --- | --- |
| `reply` | send a signal message. `chat_id` is a UUID (DM) or `group:<base64>`. supports `reply_to`, `files`. |
| `edit_message` | edit a message the bridge previously sent. `message_id` is the timestamp returned by `reply`. |
| `react` | add an emoji reaction to a message by id. |
| `typing` | show or stop a typing indicator in a chat. |
| `chat_messages` | query persistent message history. filter by `chat_id`, `since`/`until`, `search`, `limit`. |
| `list_contacts` | list signal-cli contacts. optional `match` substring filter. |
| `list_groups` | list signal groups the bridge belongs to. optional `match` filter on title/id/description. |
| `mark_read` | send a read receipt for a previously-received message. |

## history & search

inbound and outbound messages are persisted to `~/.claude/channels/signal/messages.db` (sqlite) as they flow. an author index at `~/.claude/channels/signal/authors.json` tracks display names, first/last seen, and message counts per sender. cross-session `react` works because the bridge looks up the original author from sqlite, not memory.

**install-time limitation:** messages from before v0.3 install are not in the cache. signal-cli has no `listMessages` command, so historical bootstrap isn't possible — capture is reactive from the moment the upgraded bridge starts.

## auto read-receipts (optional)

by default, a sender doesn't know the bridge saw their message until claude calls `mark_read` (or sends a reply). flip this with:

```
/signal:configure auto-receipts on
```

(or directly: `SIGNAL_AUTO_READ_RECEIPTS=true` in `.env`.) every inbound message gets an automatic read receipt as it lands, before claude has even responded. own-account syncMessages and message edits are skipped. think of it as a presence signal — useful for chats where "seen but not yet replied" is friendlier than radio silence, but a privacy regression for chats where you want claude to choose what to acknowledge.

## permission relay

claude can authorize tool calls via signal. when claude needs to run something sensitive (Bash, Write, etc.), a 🔐 prompt arrives at the owner's signal chat. reply `yes <code>` or `no <code>` to allow or deny.

owner defaults to the linked account. override with:

```
/signal:configure owner +15551234567
```

## running long-lived

the plugin itself is a Claude Code MCP server — it lives and dies with your Claude Code session. keeping the session up between machines is OS-specific, so the plugin doesn't pick a winner. some lightweight options:

- **tmux / screen / zellij** — simplest cross-platform answer. start `claude` in a detached tmux session; reattach when you want to read along.
- **nohup + log** — `nohup claude --channels plugin:signal@<your-marketplace> > ~/claude.log 2>&1 &`.
- **macOS launchd** — write a per-user `LaunchAgent` plist; `launchctl load` it.
- **Linux systemd** — community-contributed example unit at [`contrib/systemd/claude-signal.service`](./contrib/systemd/claude-signal.service); see comments inside for `loginctl enable-linger` if you want it to run without an active login.
- **Windows** — Task Scheduler with "run whether user is logged on or not", or a wrapper service via NSSM.

contributions of additional service-manager examples to `contrib/` are welcome.

## troubleshooting

bridge fails to start — run `signal-cli -a $SIGNAL_ACCOUNT receive` from a terminal to see the actual error. usually it's an account lock or a missing link.

upstream signal-cli issues — file at [AsamK/signal-cli](https://github.com/AsamK/signal-cli/issues).

plugin issues — file at this repo's issues.

## license

AGPL-3.0. see [LICENSE](./LICENSE).
