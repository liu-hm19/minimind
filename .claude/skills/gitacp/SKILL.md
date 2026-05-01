---
name: gitacp
description: Use when the user asks to stage, commit, or push changes — e.g. "commit", "push", "git acp", "提交一下", "提交并推送", "推一下", "ACP", or after finishing a unit of work that should land on the remote. Covers safe staging, commit-message authoring, and pre-push remote sync verification.
---

# gitacp — safe `git add` + `commit` + `push` workflow

## Overview

`gitacp` (add-commit-push) is the standard workflow for landing local changes on the remote. Each stage has a distinct failure mode that this skill prevents:

- **add** — accidentally staging junk: `.venv/`, `.vscode/`, model checkpoints, secrets, oversized data files
- **commit** — vague messages like `update` or `fix bug` that destroy future-you's ability to read history
- **push** — clobbering teammates' work, or pushing untested/unwanted commits to the remote

**Iron rule:** Never run any of the three commands without first inspecting state. `git status` → `git diff --staged` → `git fetch` are non-negotiable.

## When to Use

Triggers (English + Chinese):
- "commit", "commit and push", "git acp", "ACP", "push"
- "提交", "提交一下", "提交并推送", "推一下", "推送"
- After finishing a logical unit of work the user wants persisted to remote.

Do NOT use for:
- Reading git history (`git log`, `git blame`)
- Resolving merge conflicts (use a dedicated conflict-resolution flow)
- Force-push, rebase-push, or rewriting published history — these require **explicit** user instruction and are out of scope.

## The Workflow

```
SAFE ADD  →  SMART COMMIT  →  CONFIRMED PUSH
   ↓             ↓                  ↓
inspect     inspect diff       fetch + sync check
filter      derive message     show user the plan
stage       HEREDOC commit     wait for yes
explicit    no --no-verify     push (no --force)
```

Run all three stages in order. If any stage exposes a problem (oversized file, divergent remote, hook failure), **stop and report** — don't paper over it.

---

## Stage 1 — Safe `git add`

### Step 1.1 — Inspect what's there

```bash
git status
git status --ignored          # see what gitignore is hiding
```

Then size-check untracked files (catches model weights, datasets, accidental tarballs):

```bash
git ls-files --others --exclude-standard -z \
  | xargs -0 du -sh 2>/dev/null \
  | sort -h | tail -20
```

### Step 1.2 — Filter what should NOT be committed

| Category | Patterns | Why |
|---|---|---|
| Editor / IDE | `.vscode/`, `.idea/`, `*.swp`, `.DS_Store`, `Thumbs.db` | User-specific, not project state |
| Virtual envs | `.venv/`, `venv/`, `env/`, `.conda/` | Reproducible from `requirements.txt` |
| Build / cache | `__pycache__/`, `*.pyc`, `dist/`, `build/`, `*.egg-info/`, `*.so`, `*.o` | Regenerable artifacts |
| Secrets | `.env`, `*.pem`, `*.key`, `credentials.json`, `id_rsa*` | **Never commit** — rotate if leaked |
| Logs / runs | `*.log`, `logs/`, `wandb/`, `swanlog/` | Local-only telemetry |
| **Model weights** (this project) | `*.pth`, `*.bin`, `*.safetensors`, `*.ckpt`, `*.gguf`, `out/`, `checkpoints/` | Large binaries; share via HF Hub / release assets |
| Datasets | `*.parquet`, `*.arrow`, `*.tfrecord`, raw `*.jsonl` > ~10 MB | Should live in storage, not git |

**Size rules:**
- Any single file > **10 MB** → confirm with the user before staging.
- Any single file > **100 MB** → **hard stop**. GitHub rejects it. Recommend Git LFS or external storage.

### Step 1.3 — Stage explicit paths

Prefer naming files: `git add path/to/file1 path/to/file2`.
**Avoid** `git add .` and `git add -A` — they are how `.venv/`, `out/`, and `wandb/` end up in commits.

If the user clearly wants "everything I just edited", review `git status` first, then stage the listed files explicitly.

### Step 1.4 — `.gitignore` hygiene

If you find a category of file that's clearly never meant to be committed (e.g. `.venv/`), suggest adding it to `.gitignore` rather than relying on memory next time. Never use `git update-index --assume-unchanged` to hide tracked files — it creates silent drift.

If a junk file is **already tracked**, untrack with `git rm --cached <path>` before committing.

### Red flags — STOP and confirm

- A staged file is > 10 MB
- A staged file matches a secret pattern (`.env`, `*.key`, `*.pem`, `credentials*`)
- The user typed `git add .` or `git add -A` and the diff includes anything from the table above
- `node_modules/`, `.venv/`, `out/`, `checkpoints/`, or `wandb/` is showing up in `git status` as new

---

## Stage 2 — Smart `git commit`

### Step 2.1 — Read the diff before writing the message

```bash
git diff --staged --stat       # overview: which files, how many lines
git diff --staged              # full diff — read it
```

Don't write a message based on the user's request alone. Write it based on **what the diff actually changes**.

### Step 2.2 — Pick a conventional type

| Type | Use for |
|---|---|
| `feat` | New user-facing capability |
| `fix` | Bug fix |
| `docs` | Docs only (README, comments, CLAUDE.md) |
| `refactor` | Code change with no behavior change |
| `perf` | Performance improvement |
| `test` | Adding / fixing tests |
| `chore` | Tooling, deps, build config |
| `style` | Formatting only (no code change) |

### Step 2.3 — Format the message

```
<type>(<scope>): <imperative subject, ≤72 chars, no trailing period>

<body wrapped at ~72 chars, explaining WHY the change is needed
and any non-obvious tradeoffs. Skip the body for trivial commits.>
```

- **Imperative mood** ("add", "fix", "remove" — not "added", "fixes", "removing").
- **Subject ≤ 72 chars**, no period.
- **Body explains WHY**, not what (the diff already shows what).
- **Match repo style** — run `git log -10 --oneline` first; if the project uses `[update]`/`[fix]` prefixes (this repo does), match that.
- **No AI co-author attributions** unless the user explicitly asks for them.

### Step 2.4 — Commit with HEREDOC

Always use a HEREDOC so multi-line messages format correctly and quoting is safe:

```bash
git commit -m "$(cat <<'EOF'
feat(trainer): add resume support to PPO trainer

Previously PPO had no checkpointing and a crashed run lost all
progress. Resume reads checkpoints/ppo_<dim>_resume.pth and
restores model, optimizer, and step state.
EOF
)"
```

### If a pre-commit hook fails

- The commit did NOT happen — fix the underlying issue and create a **new** commit.
- **Never** use `--no-verify`. Hooks exist for a reason.
- **Never** use `--amend` to mask the failure — that rewrites the previous commit, which may not be yours.

---

## Stage 3 — Confirmed `git push`

### Step 3.1 — Sync check (mandatory)

```bash
git fetch origin
BR=$(git branch --show-current)
git log --oneline HEAD..origin/$BR    # commits remote has that we don't
git log --oneline origin/$BR..HEAD    # commits we have that remote doesn't
```

| Result | Action |
|---|---|
| Both empty | Already up to date — nothing to push. |
| Only local ahead | Safe to push. |
| Only remote ahead | **Stop.** Recommend `git pull --rebase origin $BR` and re-run sync check. Don't run pull silently. |
| Both have commits (diverged) | **Stop.** Show the user both lists. Ask whether to rebase or merge — never decide for them. |

### Step 3.2 — Show the user the push plan

Before pushing, surface:

- Branch name (and warn if it's `main` / `master` / `release/*` / any protected branch).
- Remote URL (`git remote get-url origin`).
- Number of commits and their subject lines (the output of `git log --oneline origin/$BR..HEAD`).

### Step 3.3 — Wait for explicit confirmation

Ask a yes/no question. Wait for the answer. Do not push on an implicit "ok".

If the user previously authorized a single push, that authorization does **not** carry over to subsequent pushes — re-confirm each time.

### Step 3.4 — Push

```bash
git push origin "$BR"
```

- **Never** `--force`.
- **Never** `--force-with-lease` unless the user explicitly authorized a force-push for a known reason.
- **Never** `--no-verify` to skip pre-push hooks.
- **Never** force-push to `main` / `master` / protected branches.

### Step 3.5 — Verify

```bash
git status                            # should be "up to date with origin/$BR"
```

Report success: which branch, how many commits, the remote URL.

---

## Quick Reference

| Want to… | Do |
|---|---|
| See what's staged | `git diff --staged --stat`, then `git diff --staged` |
| Find big untracked files | `git ls-files --others --exclude-standard -z \| xargs -0 du -sh \| sort -h \| tail -20` |
| Untrack a file already committed | `git rm --cached <path>` then commit |
| Check remote sync | `git fetch origin && git log --oneline HEAD..origin/$(git branch --show-current)` |
| Match repo commit style | `git log -10 --oneline` |
| Multi-line commit message | HEREDOC with `git commit -m "$(cat <<'EOF' ... EOF)"` |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `git add .` stages `.venv/` and `out/` | Stage explicit paths; add patterns to `.gitignore` |
| Commit message is "update" or "fix" | Read the diff and write what actually changed and why |
| `git push` without `git fetch` first | Always fetch and check divergence before pushing |
| `--no-verify` to bypass a failing hook | Fix the hook's complaint; create a new commit |
| `--amend` after pre-commit fails | Hook failure means the commit didn't happen — make a new commit |
| Committed a 200 MB `.pth` file | `git rm --cached`, add to `.gitignore`, recommend Git LFS or HF Hub |
| Pushed without asking | Always confirm — push is the irreversible step |

---

## Red Flags — STOP and Start Over

- About to `git add .` or `git add -A` in a repo with model weights / datasets / venvs
- About to commit a file > 10 MB (especially > 100 MB)
- About to commit a file matching `.env`, `*.key`, `*.pem`, `credentials*`
- About to push without running `git fetch` first
- About to push without showing the user which branch + commits
- About to use `--force`, `--force-with-lease`, or `--no-verify` without explicit user authorization
- Pre-commit hook failed and the urge is to `--amend` or `--no-verify`

**All of these mean: stop, surface the issue to the user, and let them decide.**
