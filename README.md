# ShipGate

A **single pre-publish gate** you run (or wire as a git `pre-push` hook) before any repo or folder goes public. It fails closed: a hard finding blocks the push, so an automated or half-asleep `git push` can't leak a secret or ship something un-licensed.

Four checks, each must pass:

| Gate | Checks |
|---|---|
| **(a) gitleaks** | secret scan on **git history** plus every file a commit could carry: tracked files as they are on disk and untracked files `.gitignore` does not exclude (a plain folder is scanned whole) |
| **(b) trufflehog** | verified-secrets scan, filesystem + git history |
| **(c) LICENSE** | a `LICENSE`/`COPYING` file exists and holds a known license's grant text (warns on a name-only stub, unfilled placeholders such as `[year]`, or an SPDX line inside it, since that can stop GitHub naming the license), and **no raw `.env`** snuck in |
| **(d) attribution** | if content is flagged derivative (`DERIVATIVE.md`, `derivative: true`, or `responding-to:` frontmatter) it requires a `CREDIT.md`/`ATTRIBUTION.md`; nudges for an AI-assist disclosure |

If a scanner isn't installed, ShipGate **warns and runs the rest; it never silently passes.**

## Why

Secrets-in-git-history is the #1 way OSS releases leak credentials, and it survives even after you delete the file from `HEAD`. ShipGate scans history as well as the working tree, and bundles the boring-but-mandatory license/`.env`/attribution checks into one exit code so "is this safe to publish?" becomes a single command.

## Install

```bash
brew install gitleaks trufflehog        # the scanners (ShipGate warns if missing)
git clone https://github.com/LucasCooperBey/shipgate.git
cp shipgate/ship-gate.sh ~/.local/bin/  # or anywhere on PATH
```

## Use

```bash
ship-gate.sh .                  # gate the current repo (scans working tree + history)
ship-gate.sh ./some/folder      # gate a plain directory
ship-gate.sh ./a-single-file.md # gate a file (scans its parent dir)
```

Exit `0` = safe to publish · nonzero = blocked, with each `[FAIL]` to fix.

### Wire it as a pre-push hook (recommended)

From inside any repo:

```bash
GATE="$(command -v ship-gate.sh)"
H="$(git rev-parse --git-dir)/hooks/pre-push"
printf '#!/usr/bin/env bash\nexec "%s" "$(git rev-parse --show-toplevel)"\n' "$GATE" > "$H"
chmod +x "$H"
```

Now `git push` runs the gate first and aborts on any hard finding. (Note: `--no-verify` skips the hook, so the gate catches mistakes and cannot stop a deliberate bypass.)

## Example

A repo at `/path/to/leaky` where a key was committed and then deleted, with no LICENSE yet. Run from inside it:

```bash
ship-gate.sh .
```

```
══ GATE (a): gitleaks filesystem scan ══
[FAIL] gitleaks git: secret(s) detected in repo/history
  Findings:
  1. [stripe-access-token] config.py:1  →  REDACTED"...
[PASS] gitleaks dir: no secrets found in the working tree

══ GATE (b): trufflehog verified-secrets scan ══
[PASS] trufflehog filesystem: no verified secrets
[PASS] trufflehog git history: no verified secrets

══ GATE (c): LICENSE + SPDX ══
[FAIL] No LICENSE file found in /path/to/leaky. Every public ship needs a license.

GATE: BLOCKED. Fix all [FAIL] items above before publishing.
```

The second commit deleted `config.py`, but the first commit still holds the key, so the gate blocks. It exits 1. The same command on a repo with an MIT LICENSE and no secrets prints `GATE: PASS` and exits 0. trufflehog passes here because it reports only keys it can verify, and this one was made up.

## License

MIT. See [LICENSE](./LICENSE). By [@LucasCooperBey](https://github.com/LucasCooperBey). Extracted clean from a larger private release pipeline.
