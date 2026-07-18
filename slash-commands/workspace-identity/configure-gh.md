---
description: Isolate a gh CLI account for a workspace via direnv (optional add-on)
argument-hint: <workspace-name>
---

You are adding **optional** gh-account isolation to an existing workspace. Rationale and the
`.envrc` template live **alongside this command file** — resolve its path and read the sibling
`README.md`; use `templates/workspace.envrc` as the source of truth.

Why this needs direnv: `gh` picks its config dir only from `GH_CONFIG_DIR` (no per-directory or
conditional config like git's `includeIf`), so per-folder gh isolation requires an env var,
which direnv sets on `cd`. git identity is handled separately and does NOT depend on this.

## 1. Resolve the workspace

- **Workspace name** = `$ARGUMENTS`. If empty, ask. Call it `<name>`.
- Load the remembered root: `cfg="${XDG_CONFIG_HOME:-$HOME/.config}/agent-toolbox/config"; [ -f "$cfg" ] && . "$cfg"`.
  Then `ws="${WORKSPACES_ROOT:-$HOME/workspaces}/<name>"`. Confirm `"$ws"` exists.

## 2. Ensure direnv is installed and hooked

```bash
command -v direnv >/dev/null || echo "install direnv first (macOS: brew install direnv — Linux: pkg manager / https://direnv.net)"
grep -q 'direnv hook' ~/.zshrc || printf '\n# direnv\neval "$(direnv hook zsh)"\n' >> ~/.zshrc   # zsh shown; new terminal needed after
```

## 3. Set up the isolated gh config — you run these

```bash
mkdir -p "$ws/dotfiles/gh"
```

Create `"$ws/.envrc"` from `templates/workspace.envrc` (self-locating; nothing to substitute),
then:

```bash
direnv allow "$ws"
```

## 4. Log in the workspace's gh account — hand to the user (via `!`)

Browser auth into the isolated config (`cd` in so `GH_CONFIG_DIR` is active):

```bash
cd "$ws" && gh auth login                       # choose "Skip" on the SSH-upload prompt
# optional, to manage SSH keys for this account:
cd "$ws" && gh auth refresh -h github.com -s admin:public_key
cd "$ws" && gh ssh-key add "$ws/dotfiles/ssh/id_ed25519.pub" --title "<name>-$(hostname)"
```

## 5. Verify

```bash
cd "$ws" && gh auth status        # → the workspace's account, isolated from other workspaces
```

Report the result to the user.
