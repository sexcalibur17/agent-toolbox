# workspace-identity

*You have two lives: a work you and a personal you.*

*Git does not care. It will happily sign your midnight side-project with your corporate email,
or push to `acme-corp/secrets` as `xXx_gamer_2009`.*

*This fixes that. Keep your identities in separate rooms, and the moment a repo lives in one,
git — in the terminal or the IDE — quietly becomes the right you. No costume changes, no
`git config` walk of shame.*

Per-folder **git identity + SSH key**, wired through `~/.gitconfig` with `includeIf`. Every repo
under a workspace gets the workspace's author/committer and push key, read by git from disk — so
it works the same in the terminal and in any GUI (WebStorm, Fork, …). An **optional** direnv
layer additionally isolates a per-workspace `gh` CLI account.

## How it works

- **Identity is disk-based.** git reads `~/.gitconfig` on every run; an
  `includeIf "gitdir:<ws>/"` block pulls in the workspace's `dotfiles/git/config` whenever a
  repo's `.git` lives under that workspace, setting `user.*` and `core.sshCommand`. Because git
  reads config from disk *itself*, this applies to the terminal **and** every GUI — that's the
  whole point, and why it isn't env/shell-based.
- **One block per workspace**, trailing slash = recursive to all nested repos.
- **`git config` shows the truth.** It's real config, so `git config user.email` inside a
  workspace returns the workspace email — a genuine sanity check, not something hidden in env.
- **gh is the exception.** `gh` has no conditional-config mechanism — it only reads
  `GH_CONFIG_DIR`. So per-workspace gh isolation is *optional* and needs direnv (terminal only).
  Everything used by daily git (identity + push key) is env-free.

## Requirements

| Dependency | Min version | Why |
|---|---|---|
| git | **≥ 2.13** | `includeIf "gitdir:"` conditional includes; `core.sshCommand` (≥ 2.10) |
| OpenSSH | any | `ssh` / `ssh-keygen`, the `IdentitiesOnly` option |
| direnv | 2.x | **optional** — only for per-workspace `gh` isolation |
| gh CLI | recent | **optional** — only if a workspace needs its own `gh` account |

**Platform notes.** Cross-platform; only optional niceties differ by OS. To persist the SSH
passphrase, macOS uses the Keychain (`ssh-add --apple-use-keychain` + `UseKeychain yes` in
`ssh_config`); on Linux use `ssh-agent` / your keyring.

**Windows:** the git side is pure `~/.gitconfig`, so it works natively (Git for Windows).
The optional gh/direnv side needs a bash-like shell — use WSL or Git Bash.

## Model

```
~/.gitconfig
    [user]  base identity                     # repos outside any workspace
    [includeIf "gitdir:<ws>/"]                # one block per workspace (trailing / = recursive)
        path = <ws>/dotfiles/git/config

<workspaces-root>/                            # default ~/workspaces (override: WORKSPACES_ROOT)
└── <workspace>/                              # e.g. wamly
    ├── dotfiles/
    │   ├── git/config                        # [user] identity + [core] sshCommand  (git reads this)
    │   ├── ssh/                              # id_ed25519 (+ .pub)       [SECRET, machine-local]
    │   └── gh/                               # GH_CONFIG_DIR (optional)  [SECRET, machine-local]
    ├── .envrc                                # OPTIONAL: gh isolation only (needs direnv)
    └── <repos...>                            # every nested repo inherits identity via includeIf
```

## Templates

Single source of truth — the README never inlines their bodies, only links them:

- [`templates/git-config`](templates/git-config) → `<workspace>/dotfiles/git/config`, wired into
  `~/.gitconfig` via `includeIf` (fill NAME/EMAIL + the key path)
- [`templates/workspace.envrc`](templates/workspace.envrc) → `<workspace>/.envrc` (optional,
  gh-only, self-locating)

## Workspaces root (remembered)

`/add-workspace` needs to know where workspaces live. It resolves the root once and remembers it
in `${XDG_CONFIG_HOME:-$HOME/.config}/agent-toolbox/config` (machine-local, not versioned):

- **First run** with nothing set → asks for the root (suggesting `~/workspaces`) and saves it.
- **Later runs** read it from the config file, no prompt.
- Set `WORKSPACES_ROOT` in the environment to override for a single run.

## Add a new workspace

**Entry points** (symlink into your assistant's command dir; Claude Code: `~/.claude/commands/`):

- [`add-workspace.md`](add-workspace.md) — `/add-workspace <name>`: the git identity + SSH key
  (disk-based, GUI-proof). At the end it offers the optional gh step.
- [`configure-gh.md`](configure-gh.md) — `/configure-gh <name>`: the optional direnv + `gh`
  isolation, runnable standalone to retrofit an existing workspace.

Manual reference for the git part:

```bash
ws="${WORKSPACES_ROOT:-$HOME/workspaces}/<name>"
mkdir -p "$ws/dotfiles/git"
mkdir -p "$ws/dotfiles/ssh" && chmod 700 "$ws/dotfiles/ssh"
# copy templates/git-config → "$ws/dotfiles/git/config"; fill NAME/EMAIL + absolute key path
grep -q "gitdir:$ws/" ~/.gitconfig || \
  printf '\n[includeIf "gitdir:%s/"]\n\tpath = %s/dotfiles/git/config\n' "$ws" "$ws" >> ~/.gitconfig
```

Then the SSH key (interactive / secret, NOT versioned):

```bash
ssh-keygen -t ed25519 -C "EMAIL" -f "$ws/dotfiles/ssh/id_ed25519"
ssh-add "$ws/dotfiles/ssh/id_ed25519"          # macOS: add --apple-use-keychain to persist the passphrase
# register "$ws/dotfiles/ssh/id_ed25519.pub" on the workspace's git host (web UI, or gh ssh-key add)
```

### Verify

```bash
# bare env (no direnv) — proves it works the way a GUI sees it:
env -i HOME="$HOME" PATH="$PATH" git -C "$ws/<repo>" config user.email       # → EMAIL
env -i HOME="$HOME" PATH="$PATH" git -C "$ws/<repo>" config core.sshCommand  # → the workspace key
```

## Secrets are never committed

Private SSH key (`dotfiles/ssh/id_ed25519`) and gh token (`dotfiles/gh/hosts.yml`) are
machine-local and MUST NOT be versioned. On a fresh machine, recreate them via the interactive
steps. Only config (this repo + templates + the `dotfiles/git/config` shape) is portable.

## Gotchas

- `includeIf` must come **after** the base `[user]` in `~/.gitconfig` to override it.
- The `gitdir:` path in `~/.gitconfig` is absolute — the git config file moves with the
  workspace, but if you relocate the tree you must update that path (it is not self-locating).
- The optional `.envrc` for gh: changing it requires re-running `direnv allow <path>`, and a new
  terminal is needed after first installing the direnv hook.
- gh is independent of the SSH key: `GH_CONFIG_DIR` governs `gh` (HTTPS+token); `core.sshCommand`
  governs `git push/pull` over SSH remotes.

## Scope

The contract: git identity guarantees hold **inside a declared workspace** — a directory covered
by an `includeIf`. Inside one you get a pinned identity and, via `core.sshCommand … IdentitiesOnly=yes`,
a pinned push key (in the terminal and in GUIs alike). Step outside every workspace and you're
back to the base `~/.gitconfig` identity, by design.

One edge is worth stating because SSH makes it non-obvious: **outside any workspace there is no
`core.sshCommand`, so git-over-SSH falls back to whatever key `ssh-agent` offers** — which may be
the wrong account's. That's the edge of the framework, not a gap. If you want a safe default out
there too, add a per-host key in `~/.ssh/config` with `IdentitiesOnly yes`.

## Fresh-machine bootstrap

```bash
git config --global user.name  "Full Name"      # base identity for repos outside any workspace
git config --global user.email "you@example.com"
# then run /add-workspace per workspace to append its includeIf + recreate keys.
# Optional, only if you use per-workspace gh accounts:
#   install direnv (macOS: brew install direnv — Linux: pkg manager / https://direnv.net)
#   printf '\n# direnv\neval "$(direnv hook zsh)"\n' >> ~/.zshrc   # new terminal after
```
