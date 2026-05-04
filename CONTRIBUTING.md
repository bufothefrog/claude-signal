# Contributing to signal-channel

Thanks for considering a contribution. This is a small project — no formal contributor onboarding, just a few conventions worth knowing before you open a PR.

## Filing issues

Use the [issue templates](.github/ISSUE_TEMPLATE/) — bug report or feature request. The bug template asks for plugin version, signal-cli version, bun version, OS, and Java version. Redact any PII (phone numbers, UUIDs, message content) before pasting `.env` or log output.

For security issues: see [SECURITY.md](./SECURITY.md). Don't file vulnerabilities as public issues.

## Versioning

[SemVer](https://semver.org/), with one project-specific convention: **major version bumps gate on hands-on confidence, not feature-completeness alone.** v1.0 was promoted from v0.7 after a week of real daily-driver use revealed no blocking issues. Future major bumps follow the same pattern — soak first, promote second.

The [`CHANGELOG.md`](./CHANGELOG.md) follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Every PR should add a bullet under an `[Unreleased]` section (or the appropriate version section if a release is imminent).

## Commit style

Single-line title, optional body separated by a blank line. Title format: `v<version>: <theme>` for release commits, `<type>: <summary>` for everything else (e.g. `chore:`, `fix:`, `feat:`). Run `git log --oneline` for examples.

## Type-checking

```sh
bun build --target=bun server.ts > /dev/null
```

The bun bundler doubles as a type checker. The `--target=bun` flag is required (default `browser` mis-resolves `node:process`). Run after every code change.

## Code conventions

- Single-file bridge: `server.ts` is the entire MCP server. Don't split it into separate transport/protocol layers.
- Tool names are `snake_case` (`update_profile`, `get_attachment`, etc.). Don't introduce camelCase or kebab-case for the MCP surface.
- Every mutation-bearing tool throws if `SIGNAL_ACCESS_MODE=static` with the standard message: `'<tool_name> blocked: SIGNAL_ACCESS_MODE=static'`.
- Inbound envelope routings emit via `notifications/claude/channel` with a `meta.event_type` discriminator — never sub-methods. The Claude Code host filters non-canonical channel sub-methods.
- `meta` values must be strings — Claude Code's channel router silently drops events with non-string meta values. Use `String(...)` for numbers/booleans, `arr.join(',')` for arrays.

## What's not in the public repo

The `.claude/` directory is gitignored. It holds project-internal docs (`plan.md`, `CLAUDE.md`) used by the maintainer's Claude Code session for design context and shouldn't drive contributor expectations.

## PR checklist

- [ ] CHANGELOG entry added
- [ ] `bun build --target=bun server.ts > /dev/null` passes
- [ ] Commit message follows the conventions above
- [ ] No personal data (phone numbers, UUIDs, real chat IDs) in code or docs
