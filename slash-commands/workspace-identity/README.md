# workspace-identity

*You have two lives: a work you and a personal you.*

*Git does not care. It will happily sign your midnight side-project with your corporate email,
or push to `acme-corp/secrets` as `xXx_gamer_2009`.*

*This fixes that. Keep your identities in separate rooms, and the moment you walk into one,
everything — git, GitHub, SSH — quietly becomes the right you. No costume changes, no
`git config` walk of shame.*

Per-folder identity for **git**, **`gh`**, and **SSH**, driven by [direnv](https://direnv.net).
Each workspace under your workspaces root (default `~/workspaces`, override with
`WORKSPACES_ROOT`) gets its own git author/committer, GitHub account/token, and SSH key,
switched automatically on `cd`. No manual toggling.

## Requirements

| Dependency | Min version | Why |
|---|---|---|
| git | **≥ 2.31** | `GIT_CONFIG_COUNT/KEY/VALUE` — inject git identity as real config (2.31, Mar 2021) |
| direnv | 2.x | the `cd`-driven switching + `direnvrc` helper |
| gh CLI | recent | `GH_CONFIG_DIR` — isolate account/token |
| OpenSSH | any | `ssh` / `ssh-keygen`, the `IdentitiesOnly` option |
| POSIX shell + direnv hook | — | `direnv hook <shell>` (`zsh` / `bash` / `fish`) |

**Platform notes.** The steps below are cross-platform. Only optional niceties differ by OS:
to persist the SSH passphrase, macOS uses the Keychain (`ssh-add --apple-use-keychain` plus
`UseKeychain yes` in `ssh_config`); on Linux use `ssh-agent` / your keyring. Clipboard helpers
differ too (`pbcopy` vs `xclip` / `wl-copy`), but the flow avoids them by using `gh`.

**Windows:** works under WSL (identical to Linux) or Git Bash — the `.envrc` is a bash script
and self-locates via `${BASH_SOURCE[0]}`, so it needs a bash-like shell (direnv itself uses
bash to evaluate `.envrc`). Native PowerShell / cmd are not supported without a port.

## Model

```
<workspaces-root>/            # default ~/workspaces (override: WORKSPACES_ROOT)
└── <workspace>/              # e.g. wamly
    ├── .envrc                # git identity + GH_CONFIG_DIR + GIT_SSH_COMMAND (self-locating)
    ├── dotfiles/
    │   ├── gh/               # GH_CONFIG_DIR (gh config + token)   [SECRET, machine-local]
    │   └── ssh/              # id_ed25519 (+ .pub)                 [SECRET, machine-local]
    └── <repos...>            # every nested repo inherits the identity
```

- **Every workspace is explicit.** Each one carries its own `.envrc`, so identity never
  depends on ambient shell/system state (see [Scope](#scope)).
- **direnv evaluates `.envrc` by its own location.** A single router `.envrc` at the root
  does NOT work — direnv won't re-evaluate when moving between sibling subdirs under the same
  `.envrc`. So each workspace carries its own `.envrc`.
- **The `.envrc` is self-locating** — it resolves its own directory via `${BASH_SOURCE[0]}`,
  so `GH_CONFIG_DIR` and the SSH key path are relative to the workspace. Nothing is tied to a
  fixed workspaces root; move or rename the tree and it still works.
- `GIT_CONFIG_*` injection sets real config → drives both author and committer AND shows in
  `git config user.email` (a genuine sanity check). See `templates/direnvrc`.
- `GH_CONFIG_DIR` isolates the `gh` token; `GIT_SSH_COMMAND` (`-i … IdentitiesOnly=yes`)
  forces this workspace's SSH key and stops the agent leaking other keys into pushes.

## Templates

Single source of truth — the README never inlines their bodies, only links them:

- [`templates/direnvrc`](templates/direnvrc) → `~/.config/direnv/direnvrc` (the `git_identity` helper)
- [`templates/workspace.envrc`](templates/workspace.envrc) → `<workspace>/.envrc` (self-locating; fill NAME/EMAIL only)

## Workspaces root (remembered)

`/add-workspace` needs to know where workspaces live. It resolves the root once and remembers
it in `${XDG_CONFIG_HOME:-$HOME/.config}/agent-toolbox/config` (machine-local, not versioned):

- **First run** with nothing set → the command asks for the root (suggesting `~/workspaces`)
  and saves your answer.
- **Later runs** read it from the config file, no prompt.
- Set `WORKSPACES_ROOT` in the environment to override for a single run.

The workspace `.envrc` is self-locating, so the root only tells the command where to create
new workspaces — nothing else depends on it.

## Add a new workspace

**Entry point:** [`add-workspace.md`](add-workspace.md) — symlink it into your assistant's
command dir (Claude Code: `~/.claude/commands/`) and run `/add-workspace <name>`. It drives the
steps below. Manual reference:

Automatable steps:

```bash
ws="${WORKSPACES_ROOT:-$HOME/workspaces}/<name>"
mkdir -p "$ws/dotfiles/gh"
mkdir -p "$ws/dotfiles/ssh" && chmod 700 "$ws/dotfiles/ssh"
# copy templates/workspace.envrc → "$ws/.envrc" and fill NAME/EMAIL (paths self-locate)
direnv allow "$ws"
```

Interactive / secret steps (run by you; NOT versioned):

```bash
ssh-keygen -t ed25519 -C "EMAIL" -f "$ws/dotfiles/ssh/id_ed25519"
ssh-add "$ws/dotfiles/ssh/id_ed25519"          # macOS: add --apple-use-keychain to persist the passphrase

cd "$ws" && gh auth login                       # choose "Skip" on the SSH-upload prompt
cd "$ws" && gh auth refresh -h github.com -s admin:public_key
cd "$ws" && gh ssh-key add "$ws/dotfiles/ssh/id_ed25519.pub" --title "<name>-$(hostname)"
```

### Verify

```bash
cd "$ws"
git config user.email                      # → EMAIL
git var GIT_AUTHOR_IDENT                    # → NAME <EMAIL>
gh auth status                             # → the workspace's account
eval "$GIT_SSH_COMMAND -T git@github.com"  # → Hi <workspace-login>!
```

## Secrets are never committed

Private SSH key (`dotfiles/ssh/id_ed25519`) and gh token (`dotfiles/gh/hosts.yml`) are
machine-local and MUST NOT be versioned. On a fresh machine, recreate them via the
interactive steps. Only config (this repo + templates) is portable.

## Gotchas

- A repo with its own `.envrc` shadows the workspace `.envrc` — add `source_up` at the top
  of that repo's `.envrc` so the workspace identity still loads.
- Changing an `.envrc` requires re-running `direnv allow <path>`.
- A new terminal is needed after first install for the direnv hook to be active.
- `gh` uses HTTPS+token by default → `GH_CONFIG_DIR` governs `gh`; the SSH key only governs
  `git push/pull` over SSH remotes. Independent concerns.

## Scope

The contract is simple: identity guarantees hold **inside a declared workspace** — a directory
with its own `.envrc`. Declare one and you get a pinned git identity, gh account, and SSH key;
step outside it and you're back to system defaults, by design.

One consequence is worth stating, because SSH makes it non-obvious: **git-over-SSH picks a key
by what `ssh-agent` offers, not by directory.** Inside a workspace this is handled — the
`.envrc` pins the key via `GIT_SSH_COMMAND … IdentitiesOnly=yes`, so no other key can slip in.
Outside any workspace nothing pins it, so the agent may offer the wrong account's key. That is
not a gap in the framework — it's the edge of it. If you want a safe default out there too, add
a per-host key in `~/.ssh/config` with `IdentitiesOnly yes`.

## Fresh-machine bootstrap

```bash
# install direnv (macOS: brew install direnv — Linux: package manager or https://direnv.net)
brew install direnv
# hook it into your shell (zsh shown; swap in the bash/fish variant if needed):
printf '\n# direnv\neval "$(direnv hook zsh)"\n' >> ~/.zshrc      # open a new terminal after
cp templates/direnvrc ~/.config/direnv/direnvrc                  # the git_identity helper
git config --global user.name  "Full Name"                       # base identity for repos
git config --global user.email "you@example.com"                 #   outside any workspace
```
