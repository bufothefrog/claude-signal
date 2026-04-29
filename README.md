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

## profile name

freshly registered signal-cli accounts have no profile, so messages from them show up as "unknown contact" until profile keys are exchanged. on first boot the bridge sets the profile name to `OpenClaw` for you. override the name via env or `.env`:

```
SIGNAL_PROFILE_NAME=Claude
```

set it to empty to disable. the bridge writes a marker file at `~/.claude/channels/signal/.profile-set` and only re-applies if the configured name changes — your signal profile won't be silently overwritten. delete the marker if you ever need to force a re-set.

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

## permission relay

claude can authorize tool calls via signal. when claude needs to run something sensitive (Bash, Write, etc.), a 🔐 prompt arrives at the owner's signal chat. reply `yes <code>` or `no <code>` to allow or deny.

owner defaults to the linked account. override with:

```
/signal:configure owner +15551234567
```

## running 24/7

a systemd user unit lives at `contrib/systemd/claude-signal.service`. to install:

```sh
loginctl enable-linger <user>
systemctl --user enable --now claude-signal
```

`enable-linger` lets the service run without an active login session.

## troubleshooting

bridge fails to start — run `signal-cli -a $SIGNAL_ACCOUNT receive` from a terminal to see the actual error. usually it's an account lock or a missing link.

upstream signal-cli issues — file at [AsamK/signal-cli](https://github.com/AsamK/signal-cli/issues).

plugin issues — file at this repo's issues.

## license

AGPL-3.0. see [LICENSE](./LICENSE).
