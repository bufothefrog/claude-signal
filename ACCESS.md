# signal-channel — Access & Delivery

access control for the signal-channel.

## default policy: pairing

a fresh install starts with an empty allowlist and `dmPolicy: "pairing"`. the first message from any sender flows like this:

1. someone sends a signal message to your linked account.
2. the bridge replies with a 6-character hex code and drops the original message.
3. you run `/signal:access pair <code>` in your claude code terminal.
4. the sender is added to `allowFrom`. the bridge sends "Paired! say hi to claude." and future messages flow through directly.

if the same sender keeps messaging without you pairing, the bridge resends the code at most twice, then goes silent — no spam loop.

## other policies

- `allowlist` — no auto-pairing. only senders already in `allowFrom` reach claude. everyone else is dropped silently.
- `disabled` — drop everything, including allowlisted senders and groups.

set with `/signal:access policy <pairing|allowlist|disabled>`.

## sender IDs are UUIDs

modern signal-cli identifies senders by UUID, not phone number. `chat_id` for a DM looks like `<8-4-4-4-12 hex uuid>` (e.g. `00000000-0000-0000-0000-000000000000`). group IDs look like `group:<base64>`. the allowlist stores UUIDs.

pairing captures the UUID for you. if you need one manually, check the `chat_id` field in a recent inbound notification or look at the `pending` entries in `access.json`.

## skill subcommand reference

### `/signal:access`

| command | action | notes |
| --- | --- | --- |
| `/signal:access` | print state | policy, allowlist, pending pairings (with age), group config |
| `/signal:access pair <code>` | approve a pending code | adds sender to `allowFrom`, sends a confirmation in signal |
| `/signal:access deny <code>` | discard a pending code | sender is not notified |
| `/signal:access allow <senderId>` | add a UUID directly | bypass pairing |
| `/signal:access remove <senderId>` | remove from allowlist | |
| `/signal:access policy <pairing\|allowlist\|disabled>` | set `dmPolicy` | |
| `/signal:access group add <chat_id> [--no-mention] [--allow id1,id2]` | enable a group | quote the chat_id; group prefix is `group:` |
| `/signal:access group rm <chat_id>` | disable a group | |
| `/signal:access set <key> <value>` | set a config key | keys: `mentionPatterns`, `textChunkLimit`, `chunkMode` |

### `/signal:configure`

| command | action | notes |
| --- | --- | --- |
| `/signal:configure` | show status | lists accounts, shows which is selected and its source (`env` / `.env` / `auto`) |
| `/signal:configure account <number>` | pin the account | writes `SIGNAL_ACCOUNT` to `~/.claude/channels/signal/.env` |
| `/signal:configure account clear` | unpin the account | reverts to auto-detect |
| `/signal:configure owner <number>` | set the permission-relay owner | writes `SIGNAL_OWNER` to `.env` |
| `/signal:configure link` | print the link command | prints `signal-cli link` + a `qrencode` one-liner; does not execute |

both skills refuse to run when the invocation looks channel-sourced — they only act on direct user input.

## group config

each group entry under `groups` accepts:

- `requireMention` (bool, default `true`) — only deliver messages that match a `mentionPatterns` regex.
- `allowFrom` (UUID array, default `[]`) — restrict triggers to these members. empty means any member of the group can trigger (still subject to `requireMention`).

`mentionPatterns` is a top-level array of case-insensitive regexes. the bridge tests each against inbound text to decide if a group message mentions claude. signal has no structured @mentions for bots, so regex is the only mechanism.

```
/signal:access set mentionPatterns '["^claude\\b", "@assistant"]'
```

## permission relay

prompts go to `SIGNAL_OWNER` only — defaults to the linked account. replies must match the `yes/no <code>` regex (5 lowercase letters a-z minus `l`, to keep codes unambiguous). channel-sourced "approve me" requests are refused by design — only the owner's chat can authorize.

## configuration files

| path | purpose | mode |
| --- | --- | --- |
| `~/.claude/channels/signal/.env` | env overrides (account, owner, signal-cli paths) | 0600 |
| `~/.claude/channels/signal/access.json` | access policy + state | 0600 |
| `~/.claude/channels/signal/approved/<senderId>` | transient handoff between skill and server | server-consumed |

the `approved/` dir is how `/signal:access pair` tells the running bridge to send the "Paired!" confirmation. the skill writes the file; the server polls every 5s, sends the confirmation, and deletes the file.

## `.env` schema

```
SIGNAL_ACCOUNT=+15551234567       # optional; auto-detect if exactly one linked
SIGNAL_OWNER=+15551234567         # owner for permission relay; defaults to SIGNAL_ACCOUNT
SIGNAL_CLI_PATH=/path/to/signal-cli   # optional
SIGNAL_CLI_CONFIG=/path/to/data       # optional
```

## `access.json` schema

```json
{
  "dmPolicy": "pairing",
  "allowFrom": ["sender-uuid", "..."],
  "groups": {
    "group:<base64>": {"requireMention": true, "allowFrom": []}
  },
  "pending": {
    "<6-char-hex-code>": {
      "senderId": "...",
      "chatId": "...",
      "createdAt": 0,
      "expiresAt": 0,
      "replies": 1
    }
  },
  "mentionPatterns": [],
  "textChunkLimit": 2000,
  "chunkMode": "newline"
}
```

absent file is equivalent to `pairing` policy with empty lists. the server re-reads on every inbound message, so skill edits take effect without restarting.

## static mode

set `SIGNAL_ACCESS_MODE=static` to snapshot `access.json` at boot and ignore writes during runtime. useful for systemd deploys where state lives in env and the on-disk file is configuration, not state. pairing is downgraded to `allowlist` in static mode — pairing requires runtime mutation, so it can't function with a frozen snapshot.
