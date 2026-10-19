# Handoff: container-engine tidy-up

Status as of 2026-10-18. **All phases (0–5) are done.** Remaining work is listed under "Still open". The day-to-day rules live in `README.md` ("Where things go", "Container engines"); this file is the record of how we got there.

Machine: M1 Max, zsh, Homebrew, mise 2026.10.12, GNU stow.

## Ground rules (from the original brief)

- **Vocabulary:** *Engine* = VM + daemon. *Endpoint* = the socket it exposes. *Context* = the Docker CLI's name for an endpoint. *Selector* = an env var choosing context/engine (`DOCKER_CONTEXT`, `CONTAINER_ENGINE`, `KIND_EXPERIMENTAL_PROVIDER`).
- Register contexts once; select with selectors in mise `[env]` or Makefiles. **Never `docker context use`**, because it mutates global `~/.docker/config.json`.
- Work one phase at a time and STOP at each phase boundary. The user wants each change explained.
- No sudo, uninstalls, `podman machine rm`, `colima delete`, or pruning from the agent: print the command and let the user run it.
- Persistent config goes into **`~/Workspaces/dotstow`** as a stow package (`stow -t ~ <pkg>`), never directly into `~`. Show the diff before writing.
- Discover socket paths from the tools (`podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}'`, `colima status`, `docker context inspect`). Never hard-code one.
- Brewfile owns docker/compose/buildx CLIs; mise owns runtimes and dev tools. **Brewfile work is still deferred.**

## Repos

| Repo | Role |
|---|---|
| `~/Workspaces/dotstow` | Stow packages and design notes: `mise/`, `shims/`, `containers/`, `kubectl/`, `topic2/`. |
| `~/Workspaces/dots` | `tools/`, `workflows/` stow trees; owns `~/.zshrc`, `~/.zshenv`, `~/.config/shell/load.zsh`. |
| `~/.dotfiles` | Old repo; has the `Brewfile` (no container entries) and `~/.zprofile`. Deferred. |

## Decisions made

- **Keep Podman** (primary) and **Colima** (the single Moby/BuildKit engine). **Keep minikube**, running on Colima. **Drop Rancher Desktop** (done).
- `/var/run/docker.sock` stays **intentionally owned by Podman** (Podman Desktop "Docker Compatibility" ON → podman-mac-helper).
- The Podman machine is **rootful**; the default connection stays on `podman-machine-default-root` so docker.sock, the API socket and kind all see root's image store.
- Colima runs with k3s **off** and 6 GiB, freeing room for minikube.
- Don't create a minikube docker context.
- `~/.docker/config.json` is machine-written (auths, credsStore) and is **not stowed**; one-off edits only.
- minikube is pinned to Colima by a **shim** (`shims/.local/shims/minikube`), put first on PATH by the global mise config (`_.path`). `~/.local/bin` couldn't host it: `~/.zprofile` appends it after Homebrew, and moving it would change which `claude` runs.

## Final state (verified with `container-doctor --all`, exit 0)

| Item | State |
|---|---|
| Contexts | `podman` and `default` → Podman 6.1.4 (fedora); `colima` → Docker 29.5.2 (Ubuntu 24.04). |
| docker.sock | → Podman's API socket. |
| `currentContext` | Key absent in `~/.docker/config.json`, which Docker treats as `default`. |
| Selectors | Only in `mise/.config/mise/config.toml` `[env]`: all `podman`. No `DOCKER_HOST` anywhere. |
| minikube | Profile `minikube` on Colima, running; it is the only kube context. `minikube` resolves to the shim in mise shells and their children. |
| compose / buildx | Homebrew `docker-compose` 5.6.0 and `docker-buildx` 0.38.0, found via `cliPluginsExtraDirs: ["/opt/homebrew/lib/docker/cli-plugins"]` in `~/.docker/config.json` (set by the user; the broken HTML `~/.docker/cli-plugins/docker-compose` is gone). |
| Doctor | `~/.local/bin/container-doctor` (package `containers/`). |

## What each phase found

- **Phase 2:** the `podman` context had already been recreated against the discovered socket, and `currentContext` was already unset, so no changes were needed.
- **Phase 3 (selectors):** mise applies project `[env]` on `cd` only in interactive zsh. `zsh -c`, `make` and agent shells inherit the caller's env; use `mise exec -C <dir> --`. A manual `export DOCKER_CONTEXT=…` is reset by mise at the next prompt. `DOCKER_HOST` isn't managed by mise and overrides everything.
- **Correction:** the earlier handoff said `~/.config/mise/config.toml` was a regular file. It's actually inside a **folded stow symlink** (`~/.config/mise` → `dotstow/mise/.config/mise`); `ls ~/.config/mise/` follows the link and hides that.
- **Phase 4:** `mise shell-alias set` / `mise use -g` write through both folded-dir and file-level symlinks and keep comments. The Makefile convention in the README is tested on macOS make 3.81. `DOCKER_CONTEXT` must default to a context name, not be derived from `CONTAINER_ENGINE` (Colima's engine is `docker`, its context is `colima`).
- **Phase 5:** `container-doctor` caps each endpoint check at about 3 s (macOS has no `timeout`), reads only `currentContext` from `config.json`, and exits 1 on any warning.

## Still open

- **Brewfile:** add `docker`, `docker-compose`, `docker-buildx`, `colima`, `lima`, `kubernetes-cli`, `minikube`, `docker-credential-helper`.
- **Work config in mise:** anything created under `~/.config/mise/` (e.g. `conf.d/work.toml`) lands in dotstow's working tree, and another stow dir can't stow into the folded link. Gitignore it, or unfold `~/.config/mise` first.
- Podman Desktop's standalone `/usr/local/bin/docker-compose` (5.1.3) is shadowed by Homebrew's; turn it off in Podman Desktop → Extensions → Compose if it gets re-offered.
- Optional Rancher leftovers: `~/.kuberlr`, `~/Library/Caches/rancher-desktop-updater`, `~/Library/Preferences/io.rancherdesktop.app.plist`.
- Two alias mechanisms coexist (dotstow `aliases.d/` via `load_aliases`, dots `rc.d` via `dots_load`); the README placement rules point new global aliases at dots.
