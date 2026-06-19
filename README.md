# ff — Brightly CLI version feature flags

Central version registry for Brightly-published CLI utilities. Each tool
checks this repo periodically (or on demand via `<tool> update`) to
discover whether a newer version is available, and to learn the minimum
version it's allowed to keep running on.

This repo is intentionally tiny and slow-moving — a YAML pointer per
utility, bumped whenever we cut a new release. It's also intentionally
public (or at least cheaply readable), so CLI consumers don't need
authenticated access to perform the version check.

## Layout

One directory per CLI binary. Inside, a `version.yml` file holds the
version flags for that tool.

```
ff/
├── README.md
├── cc/
│   └── version.yml          # coding-container launcher
└── s3map/
    └── version.yml          # planned — s3map version pointer
```

The directory name is the **binary name as users invoke it**, not the
formula name. `cc/` covers the `coding-container` brew formula because
the binary is `cc`; if we ever ship a formula whose binary differs from
its name, the directory follows the binary.

## `version.yml` schema

```yaml
min: 1.0.0           # minimum version a running CLI is allowed to be on
current: 1.0.0       # version the CLI should upgrade to
last_update: 2026_06_18   # date the entry was last bumped (YYYY_MM_DD)
```

| Key            | Type   | Required | Meaning                                                                                       |
| -------------- | ------ | -------- | --------------------------------------------------------------------------------------------- |
| `min`          | semver | yes      | Hard floor. Versions below this should refuse to run (or force-upgrade) when they next check. |
| `current`      | semver | yes      | The version we want users on. Anything between `min` and `current` is soft-nag territory.    |
| `last_update`  | date   | yes      | Underscore-separated `YYYY_MM_DD`. Lets consumers display "last updated on …" in their notice. |

`min` and `current` together give us a two-tier upgrade policy without
extra machinery:

- **below `min`** — block or force-upgrade (security floor)
- **between `min` and `current`** — soft nag (suggest upgrading, keep running)
- **at or above `current`** — silent, up to date

Set `min == current` when there is no soft-nag window (every release is
considered mandatory; older versions get blocked immediately on next
check). Set `min < current` when an older release is still safe but
we'd prefer users move forward.

## How CLIs consume this

The expected flow for any tool that opts in:

1. The CLI fetches `https://raw.githubusercontent.com/teambrightly/ff/main/<binary>/version.yml`
   on an `<tool> update` invocation (and optionally on a cadence — e.g.
   once a day on first launch).
2. It parses `min` and `current`, compares them to the version baked
   into the binary, and decides whether to:
   - print an "up to date" message (≥ `current`),
   - print a soft upgrade nag (≥ `min`, < `current`),
   - refuse to run / force the upgrade (< `min`).
3. If an upgrade is needed, the tool tells the user the exact command
   to run for its install method (e.g. `brew upgrade teambrightly/tap/coding-container`).

The CLI itself never writes to this repo — the registry is read-only
from the consumer's perspective. Updates land via the publish flow
below.

A consumer should fail soft on network errors: if `ff` is unreachable
the check is a no-op, not a blocker (otherwise an outage here would
take every Brightly CLI offline).

## Publishing an update

When you cut a new release of a registered CLI:

1. Tag and push the release in the source repo (e.g. `teambrightly/coding-container`).
2. Open this repo, edit `<binary>/version.yml`:
   - Bump `current` to the new version.
   - Bump `min` only when the previous version is genuinely unsafe to
     keep running (security fix, broken protocol contract, etc.).
     Otherwise leave `min` alone and let it lag behind.
   - Update `last_update` to today's date in `YYYY_MM_DD` form.
3. Commit and push to `main`. The change goes live as soon as GitHub
   serves the new `raw.githubusercontent.com` content (usually within a
   few seconds, occasionally up to a minute on cache).

There is no review pipeline by design — this is a small file in a
small repo, optimized for fast turnaround on release day. If we ever
need approval gates, branch protection on `main` is the obvious place
to add them.

## Adding a new tool

1. Create a directory at the binary name (`mkdir <binary>`).
2. Add a `version.yml` with the three required keys.
3. Wire the consumer side into the tool's `<tool> update` (or
   equivalent) so it actually reads the file.
4. Document the consumer's behavior in the tool's own README — there
   is no central "consumer behavior" spec; each CLI decides how loud
   to be about a stale version.

## Privacy and access

This repo holds **public** version flags — no secrets, no per-customer
metadata, no install telemetry. That keeps the consumer side trivially
simple (a plain unauthenticated HTTPS GET) and lets us host the file
on GitHub's raw CDN without auth.

Anything that needs auth or per-user state belongs somewhere else,
not here.
