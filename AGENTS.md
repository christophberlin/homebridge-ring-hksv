# Agent instructions

## GitHub identity preflight (run before any commit or push)

This machine has **two GitHub identities**. Git and `gh` can resolve to
*different* accounts, so verify both before writing history or pushing.

```bash
git config --get user.email                                   # commit identity
printf 'protocol=https\nhost=github.com\n\n' | git credential fill | grep '^username='   # push identity
```

Both must resolve to `christophberlin`. Expected values:

| Layer | Expected |
| --- | --- |
| `user.name` | `Christoph Berlin` |
| `user.email` | `7737992+christophberlin@users.noreply.github.com` |
| credential `username` | `christophberlin` |

### If push identity is wrong

Xcode's system gitconfig sets `credential.helper = osxkeychain`, and the macOS
keychain may return the unrelated work account `cberlin_microsoft`. That causes:

```
remote: Permission to christophberlin/<repo>.git denied to cberlin_microsoft
fatal: ... The requested URL returned error: 403
```

Fix it by routing github.com credentials through `gh`:

```bash
gh auth setup-git
```

This writes a **host-scoped** helper to `~/.gitconfig`: an empty
`credential.https://github.com.helper` that clears the inherited `osxkeychain`
helper, followed by `!gh auth git-credential`. Host-scoped config takes
precedence over the generic `credential.helper`.

### If commit identity is wrong

When `user.email` is unset, git derives one from username and hostname (for
example `christophberlin@Christophs-MacBook-Air-M5.local`). Such commits push
successfully but are **not linked to the GitHub account** — no avatar, no
profile link, no contribution credit. Verify attribution with:

```bash
gh api repos/<owner>/<repo>/commits/<sha> --jq 'if .author then .author.login else "NULL (unattributed)" end'
```

Set the identity globally if missing:

```bash
git config --global user.name "Christoph Berlin"
git config --global user.email "7737992+christophberlin@users.noreply.github.com"
```

### Rules

- A `403` on push means **wrong identity**, not necessarily missing access.
  Run `gh auth status` and the credential-fill check before concluding anything.
- Never rewrite published history to fix attribution without explicit approval;
  amending requires a force-push.

## Repository scope

- `origin` is the fork `christophberlin/homebridge-ring-hksv`.
- `upstream` is `trinityhades/homebridge-ring-hksv`. Do not push to `upstream`.

## HKSV support boundary

HKSV support is permanently limited to always-powered premium Ring cameras on
capable hosts. Battery, solar, and low-performance deployments (including
Raspberry Pi-class hosts) are out of scope and must not be reintroduced as
reliability or roadmap targets. See `README.md` and `ANALYSIS.md`.

## Validation

```bash
npm run lint
npm test
```
