# git-local

Persist local-only changes in git-tracked files — without ever committing them.

## The Problem

You have a file like `vite.config.ts` that is tracked by git. You need to add a local development setting (e.g. `allowedHosts: ['localdomain.test']`) that should **never** be committed — but the file itself must remain tracked, and other changes to it should commit normally.

`.gitignore` can only ignore entire files. `git update-index --skip-worktree` ignores the whole file. Neither lets you commit *some* changes to a file while permanently suppressing *others*.

`git-local` solves this at the hunk level.

## How It Works

Selected hunks are saved as patch files in `.git/local-patches/`. Because they live inside `.git/`, they are:

- **Never committed** — `.git/` is not part of the repository tree
- **Never pushed** — same reason
- **Never need a `.gitignore` entry** — they are already outside the tracked tree
- **Persistent across branch switches and rebases** — they live in the repo's `.git/`, not in the working tree

When you commit, a pre-commit hook strips the patches (removes them from both the working tree and the index), verifies that something else is actually staged, and the post-commit hook re-applies them. To you, it's invisible — you just `git commit` normally.

---

## Installation

### Requirements

- Python 3.6+
- Git

### Install

```sh
# Clone the repo
git clone https://github.com/wilkesreid/git-local.git
cd git-local

# Copy the script somewhere on your PATH
cp git-local /usr/local/bin/git-local
chmod +x ~/bin/git-local

# Verify
git-local --help
```

Because the script is named `git-local`, git also recognises it as a subcommand:

```sh
git local save vite.config.ts   # same as: git-local save vite.config.ts
```

---

## Typical Workflow

```sh
# 1. Install hooks so commits strip/re-apply automatically (one-time setup per repo)
git-local install-hooks

# 2. Make your local-only change in the file as normal

# 3. Save the hunk(s) you want to keep local — interactive picker like `git add -p`
git-local save vite.config.ts

# 4. Work normally — commit, push, pull. The patch stays in .git/ and never goes out.
git add . && git commit -m "feat: whatever"

# 5. Your local change is automatically re-applied after the commit.
```

---

## Commands

### `save <file>`

Interactively select hunks to mark as local-only. Uses the same hunk-picker interface as `git add -p`.

```sh
git-local save vite.config.ts
```

Prompts for each hunk:
- `y` — save as local-only (this hunk will never be committed)
- `n` — skip (leave this hunk as a normal staged/unstaged change)
- `a` — save all remaining hunks without prompting
- `q` — quit without saving

**Flags:**

| Flag | Description |
|------|-------------|
| `-a`, `--all` | Save all hunks without prompting |
| `--replace` | Overwrite an existing patch without prompting for confirmation |

```sh
# Save everything in the file as local-only, no prompts
git-local save -a vite.config.ts

# Re-capture a patch after the file has changed significantly
git-local save --replace vite.config.ts
```

---

### `apply [files...]`

Re-apply saved patches to the working tree. Run this to restore your local changes after a fresh clone, a rebase, or a manual `strip`.

```sh
# Apply all patches
git-local apply

# Apply only specific files
git-local apply vite.config.ts src/config/local.ts
```

**Flags:**

| Flag | Description |
|------|-------------|
| `-f`, `--force` | Re-apply even if the patch is already marked as applied |

---

### `strip [files...]`

Remove saved patches from the working tree and index. Useful before a manual commit if you are not using the automated hooks.

```sh
# Strip all patches
git-local strip

# Strip only specific files
git-local strip vite.config.ts
```

**Flags:**

| Flag | Description |
|------|-------------|
| `-f`, `--force` | Strip even if the patch is already marked as stripped |

---

### `list`

Show all saved patches and their cached applied/stripped status.

```sh
git-local list
```

Example output:

```
Local patches:

  [applied  ]  vite.config.ts  (312B)
  [stripped ]  src/env.ts      (89B)
```

> Note: `list` reads from `.state.json` (cached state). Use `status` for a live check.

---

### `status`

Verify the actual state of each patch via a dry-run `git apply --check`. More accurate than `list` because it reflects the current working tree, not just the last recorded state.

```sh
git-local status
```

Possible states:

| State | Meaning |
|-------|---------|
| `applied` | Patch is currently in the working tree |
| `stripped` | Patch is not in the working tree |
| `clean` | No changes at all in the file (patch applies and reverses cleanly — effectively a no-op) |
| `conflict` | Patch cannot apply or reverse; the file likely changed incompatibly |

---

### `show <file>`

Print the raw saved patch for a file, with syntax highlighting.

```sh
git-local show vite.config.ts
```

Useful for inspecting what is saved before deciding whether to re-capture or drop it.

---

### `drop <file>`

Permanently delete the saved patch for a file. Does **not** modify your working tree — if the patch is currently applied, those changes remain in the file but will no longer be tracked by `git-local`.

```sh
git-local drop vite.config.ts
```

Prompts for confirmation before deleting.

---

### `install-hooks`

Install `pre-commit` and `post-commit` git hooks in the current repository that automatically strip and re-apply patches around every commit.

```sh
git-local install-hooks
```

**pre-commit hook behaviour:**
1. Strips all local patches (from both working tree and index)
2. Checks if anything is still staged
3. If nothing is staged (your local patch was the *only* change), re-applies the patches and aborts the commit with a clear message — no empty commit, no stuck state
4. If changes remain, allows the commit to proceed

**post-commit hook behaviour:**
1. Re-applies all local patches immediately after the commit completes

If a hook file already exists and already contains a `git-local` entry, the install is skipped for that hook. If a hook exists without a `git-local` entry, the snippet is appended.

> You only need to run `install-hooks` once per repository clone.

---

## Edge Cases

### The base file changes significantly

If a teammate rewrites the area around your saved hunk, `apply` will fail with a conflict message. To recover:

```sh
# 1. See what the saved patch looks like
git-local show vite.config.ts

# 2. Manually re-apply your change in the editor

# 3. Re-capture the patch, replacing the old one
git-local save -a --replace vite.config.ts
```

### Working without hooks

If you prefer not to use hooks, strip and re-apply manually around commits:

```sh
git-local strip
git add . && git commit -m "..."
git-local apply
```

### Checking in after a fresh clone

Patches live in `.git/local-patches/` and are not committed, so a fresh clone starts without them. After cloning, make your local changes and save them:

```sh
git-local save vite.config.ts
git-local install-hooks
```

---

## File Layout

```
.git/
  local-patches/
    vite.config.ts.patch     ← the saved hunk(s) as a unified diff
    src__config__local.patch ← slashes replaced with __ in filenames
    .state.json              ← cached applied/stripped state per file
  hooks/
    pre-commit               ← strips patches before commit
    post-commit              ← re-applies patches after commit
```

---

## License

MIT
