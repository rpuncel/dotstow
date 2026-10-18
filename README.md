# dotstow

Dotfiles managed with [GNU Stow](https://www.gnu.org/software/stow/). Each top-level dir is a **package** whose contents mirror `$HOME`; stowing symlinks them into place.

## How it fits together

- **Target is `~`** — always pass `-t ~` (repo lives in `~/Workspaces`, not `~`).
- **`aliases.d/`** — packages drop `*.sh`/`*.zsh` into `~/aliases.d/`; `load_aliases` in `~/.zshrc` sources them all at shell start.
- **mise** — `mise/.config/mise/config.toml` is the global mise config (env vars, tools). Activated in `~/.zshrc` via `mise activate zsh`. Whole dir is symlinked, so `mise use -g …` writes back into this repo.

| Package   | Provides                                        |
|-----------|-------------------------------------------------|
| `kubectl` | `~/aliases.d/kubectl_aliases.sh` (`k`, `kg`, `kd`, …) |
| `mise`    | `~/.config/mise/` — podman as container engine (`DOCKER_CONTEXT`, `CONTAINER_ENGINE`, `KIND_EXPERIMENTAL_PROVIDER`) |
| `topic2`  | `~/aliases.d/touch.sh` (placeholder)            |

## Stow cheatsheet

Run from the repo root.

```sh
stow -t ~ <pkg>          # link package
stow -t ~ -D <pkg>       # unlink package
stow -t ~ -R <pkg>       # relink (after adding/removing files)
stow -t ~ -n -v <pkg>    # dry run — show what would happen
stow -t ~ */             # link everything
stow -t ~ --adopt <pkg>  # pull existing real files into the repo, then link (check diff after!)
```

## Adding things

```sh
# new alias topic
mkdir -p foo/aliases.d && $EDITOR foo/aliases.d/foo.sh
stow -t ~ foo && exec zsh

# new app config (mirror the $HOME path)
mkdir -p bar/.config/bar && $EDITOR bar/.config/bar/config
stow -t ~ bar
```

Tip: if the target dir doesn't exist yet, stow symlinks the whole dir (tree folding). Pre-create it (`mkdir -p ~/.config/x`) to link individual files instead.

## mise cheatsheet

```sh
mise env                 # show env mise would export here
mise config ls           # which config files are active
mise use -g node@22      # add a global tool (writes to this repo)
mise ls                  # installed tools
mise doctor              # diagnose activation issues
```
