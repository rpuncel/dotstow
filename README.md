# dotstow

Dotfiles managed with [GNU Stow](https://www.gnu.org/software/stow/). Each top-level dir is a **package** whose contents mirror `$HOME`; stowing symlinks them into place.

## How it fits together

- **Target is `~`** — always pass `-t ~` (repo lives in `~/Workspaces`, not `~`).
- **`aliases.d/`** — packages drop `*.sh`/`*.zsh` into `~/aliases.d/`; `load_aliases` in `~/.zshrc` sources them all at shell start.
- **mise** — `mise/.config/mise/config.toml` is the global mise config (env vars, tools). Activated in `~/.zshrc` via `mise activate zsh`. Whole dir is symlinked, so `mise use -g …` writes back into this repo.

| Package   | Provides                                        |
|-----------|-------------------------------------------------|
| `containers` | `~/.local/bin/container-doctor` — effective selectors, does the endpoint answer (`--all`: every context + docker.sock) |
| `kubectl` | `~/aliases.d/kubectl_aliases.sh` (`k`, `kg`, `kd`, …) |
| `mise`    | `~/.config/mise/` — podman as container engine (`DOCKER_CONTEXT`, `CONTAINER_ENGINE`, `KIND_EXPERIMENTAL_PROVIDER`); prepends `~/.local/shims` to PATH |
| `shims`   | `~/.local/shims/` — wrappers that pin a tool to one engine (`minikube` → Colima) |
| `topic2`  | `~/aliases.d/touch.sh` (placeholder)            |

## Where things go

| What | Where |
|---|---|
| Global, portable alias/function worth help text | `~/Workspaces/dots` — `tools/` or `workflows/` shell files (`dot-config/shell/rc.d/*.sh`, loaded by `dots_load rc.d`) |
| Directory-scoped alias or env | the project's `mise.toml` — `[shell_alias]` / `[env]` |
| Container engine selection | selectors only: `DOCKER_CONTEXT`, `CONTAINER_ENGINE`, `KIND_EXPERIMENTAL_PROVIDER` in mise `[env]` or a Makefile. Never `docker context use` (mutates `~/.docker/config.json`), never `DOCKER_HOST` in rc files |
| A tool that must always use one engine | a wrapper in `shims/.local/shims/` |
| Work hostnames, credentials, work-only tools | the work repo; for global mise, `~/.config/mise/conf.d/work.toml` (see caveat below) |

### Container engines

Engines: **Podman** (rootful, owns `/var/run/docker.sock`, context `podman`/`default`) and **Colima** (context `colima`, hosts minikube). Contexts are registered once; pick one with a selector.

How selectors reach a process:

- Interactive zsh: mise re-applies `[env]` on every prompt, so `cd` into a project switches engines — and a manual `export DOCKER_CONTEXT=…` is **reset at the next prompt**. For one-offs use `DOCKER_CONTEXT=colima docker …` or `docker --context colima …`.
- `zsh -c`, `make`, agent shells: inherit the caller's env; `cd` does nothing. Use `mise exec -C <dir> -- …`, or have the Makefile set its own.
- `DOCKER_HOST` beats `DOCKER_CONTEXT` and mise doesn't manage it — `eval $(minikube docker-env)` silently overrides everything until you unset it.

`docker compose` / `docker buildx` come from Homebrew (`docker-compose`, `docker-buildx`); the CLI finds them through `cliPluginsExtraDirs: ["/opt/homebrew/lib/docker/cli-plugins"]` in `~/.docker/config.json` (a one-off edit; that file isn't stowed). Run `container-doctor --all` to check the whole setup.

Makefile convention:

```make
CONTAINER_ENGINE ?= podman             # follow the caller (mise), default podman
DOCKER_CONTEXT   ?= podman             # a context name, not derived from the engine (colima runs engine "docker")
export CONTAINER_ENGINE DOCKER_CONTEXT
# hard pin for targets that need one engine (works with macOS make 3.81):
minikube-%: export DOCKER_CONTEXT := colima
```

### mise config is a folded symlink

`~/.config/mise` → `mise/.config/mise` (whole dir). `mise use -g` / `mise shell-alias set` write through it into this repo, preserving comments; they also keep a file-level symlink intact if the dir is ever unfolded. Caveat: anything created under `~/.config/mise/` (e.g. `conf.d/work.toml`) lands **in this repo's working tree** — gitignore it, or unfold the dir (`mkdir` a real `~/.config/mise`, link files individually) before another repo stows into it.

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
