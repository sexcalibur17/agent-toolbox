---
description: Add a new per-workspace git identity (identity + SSH key, GUI-proof via includeIf)
argument-hint: <workspace-name>
---

You are setting up a new isolated workspace identity. Rationale, gotchas, and `templates/`
(source of truth for the config files) live **alongside this command file** — resolve this
command's path and read its sibling `README.md`; do not invent a different shape.

Identity is disk-based: it lives in the workspace's `dotfiles/git/config` and is wired into
`~/.gitconfig` with an `includeIf "gitdir:"`. git reads it from disk on every run, so it
applies in the terminal **and** any GUI (WebStorm, …) — no env/direnv needed for git.

## 1. Resolve the workspaces root (do this first)

Remembered across runs so it's set once and reused:

```bash
cfg="${XDG_CONFIG_HOME:-$HOME/.config}/agent-toolbox/config"
[ -f "$cfg" ] && . "$cfg"                         # loads saved WORKSPACES_ROOT, if any
```

- If `WORKSPACES_ROOT` is now set (env override, or from `$cfg`) → use it.
- If still **unset** → **ask the user** (suggest `~/workspaces`), never write a default
  silently, then persist and export:

```bash
root="<user's answer>"
mkdir -p "$(dirname "$cfg")" && printf 'WORKSPACES_ROOT=%s\n' "$root" > "$cfg"
export WORKSPACES_ROOT="$root"
```

## 2. Resolve remaining inputs (ask; never guess)

- **Workspace name** = `$ARGUMENTS`. If empty, ask; do not proceed with an empty or guessed
  name. Validate it is a single path segment (no spaces or `/`). Call it `<name>`.
- git author **NAME** and **EMAIL** for this workspace.

## 3. git identity (disk-based, GUI-proof) — you run these

```bash
ws="$WORKSPACES_ROOT/<name>"
mkdir -p "$ws/dotfiles/git"
mkdir -p "$ws/dotfiles/ssh" && chmod 700 "$ws/dotfiles/ssh"
```

Create `"$ws/dotfiles/git/config"` from `templates/git-config`, substituting `NAME`, `EMAIL`,
and the **absolute** `$ws` in the `sshCommand` key path. Then wire it into `~/.gitconfig` with
an `includeIf` (idempotent — do not duplicate if a block for this workspace already exists):

```bash
grep -q "gitdir:$ws/" ~/.gitconfig || printf '\n[includeIf "gitdir:%s/"]\n\tpath = %s/dotfiles/git/config\n' "$ws" "$ws" >> ~/.gitconfig
```

The trailing slash makes it recursive to every repo under the workspace.

## 4. SSH key — hand these to the user (via `!`)

Passphrase prompt + secret output (machine-local, NOT versioned):

```bash
ssh-keygen -t ed25519 -C "EMAIL" -f "$ws/dotfiles/ssh/id_ed25519"
ssh-add "$ws/dotfiles/ssh/id_ed25519"          # macOS: add --apple-use-keychain to persist the passphrase
```

Then register `"$ws/dotfiles/ssh/id_ed25519.pub"` on the workspace's git host (GitHub, Bitbucket,
…) — via that host's web UI, or `gh ssh-key add` if this workspace also sets up gh (step 6).

## 5. Verify git identity

```bash
# bare env (no direnv) — proves it works the way a GUI sees it:
env -i HOME="$HOME" PATH="$PATH" git -C "$ws" config user.email        # → EMAIL
env -i HOME="$HOME" PATH="$PATH" git -C "$ws" config core.sshCommand   # → the workspace key
# (run inside an actual repo under "$ws"; identity is inherited by every nested repo)
```

## 6. Optional: isolate a gh account for this workspace

Ask the user: **"Isolate a separate gh CLI account for this workspace via direnv?"** This is
only useful if the workspace uses a distinct GitHub account for `gh` (repo/PR/ssh-key commands).
If **yes**, run the sibling **`configure-gh`** command for `<name>`. If **no**, you're done —
git identity works without direnv.
