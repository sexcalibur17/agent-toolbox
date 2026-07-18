---
description: Add a new per-workspace identity (git + gh + ssh) under the workspaces root
argument-hint: <workspace-name>
---

You are setting up a new isolated workspace identity. The rationale, gotchas, and the
`templates/` (source of truth for `.envrc`) live **alongside this command file** — resolve
this command's path (e.g. the symlink target of the invoked command) and read its sibling
`README.md`; do not invent a different shape.

## 1. Resolve the workspaces root (do this first)

The root is remembered across runs in a config file, so it's set once and then reused. Check
for it before anything else:

```bash
cfg="${XDG_CONFIG_HOME:-$HOME/.config}/agent-toolbox/config"
[ -f "$cfg" ] && . "$cfg"                         # loads saved WORKSPACES_ROOT, if any
```

- If `WORKSPACES_ROOT` is now set (from the environment as a one-off override, or from `$cfg`)
  → use it and move on.
- If it is still **unset** → **ask the user** for the root (suggest `~/workspaces`). Use their
  answer — never write a default silently — then persist it and export it:

```bash
root="<user's answer>"
mkdir -p "$(dirname "$cfg")" && printf 'WORKSPACES_ROOT=%s\n' "$root" > "$cfg"
export WORKSPACES_ROOT="$root"
```

## 2. Resolve remaining inputs (ask; never guess)

- **Workspace name** = `$ARGUMENTS`. If empty, ask the user; do not proceed with an empty or
  guessed name. Validate it is a single path segment (no spaces or `/`). Call it `<name>`.
- git author **NAME** and **EMAIL** for this workspace.
- Confirm it needs its own **GitHub account** and **SSH key** (usually yes).

## 3. Automatable setup — you run these

```bash
ws="$WORKSPACES_ROOT/<name>"
mkdir -p "$ws/dotfiles/gh"
mkdir -p "$ws/dotfiles/ssh" && chmod 700 "$ws/dotfiles/ssh"
```

Create `"$ws/.envrc"` from `templates/workspace.envrc`, substituting only `NAME` and `EMAIL`
(the template self-locates its paths — leave those as-is). Then:

```bash
direnv allow "$ws"
```

Precondition: `~/.config/direnv/direnvrc` must define `git_identity` (see `templates/direnvrc`).
If missing, install it first.

## 4. Interactive / secret steps — hand these to the user (via `!`)

These need a passphrase prompt and browser auth, and produce machine-local secrets that are
NOT versioned. Give them to the user to run, do not attempt yourself:

```bash
ssh-keygen -t ed25519 -C "EMAIL" -f "$ws/dotfiles/ssh/id_ed25519"
ssh-add "$ws/dotfiles/ssh/id_ed25519"          # macOS: add --apple-use-keychain to persist the passphrase

cd "$ws" && gh auth login                       # choose "Skip" on the SSH-upload prompt
cd "$ws" && gh auth refresh -h github.com -s admin:public_key
cd "$ws" && gh ssh-key add "$ws/dotfiles/ssh/id_ed25519.pub" --title "<name>-$(hostname)"
```

## 5. Verify (after the user confirms step 4)

```bash
cd "$ws"
git config user.email                      # → EMAIL
git var GIT_AUTHOR_IDENT                    # → NAME <EMAIL>
gh auth status                             # → the workspace's account
eval "$GIT_SSH_COMMAND -T git@github.com"  # → Hi <workspace-login>!
```

Report the verification results back to the user.
