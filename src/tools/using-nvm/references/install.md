# Installing nvm

This file covers installing, updating, and uninstalling nvm itself, plus platform-specific setup (Docker, Ansible, Alpine, macOS). All URLs pinned to **v0.40.4**, the latest at the time this skill was written.

## Table of contents

- [Install / update script](#install--update-script) — the normal path
- [Profile snippet](#profile-snippet) — what the install script adds, and why
- [Install script env variables](#install-script-env-variables)
- [Verifying the install](#verifying-the-install)
- [Linux troubleshooting after install](#linux-troubleshooting-after-install)
- [macOS troubleshooting after install](#macos-troubleshooting-after-install)
- [Ansible](#ansible)
- [Installing in Docker (interactive)](#installing-in-docker-interactive)
- [Installing in Docker for CI/CD](#installing-in-docker-for-cicd)
- [Git install](#git-install)
- [Manual install](#manual-install)
- [Manual upgrade](#manual-upgrade)
- [Alpine Linux](#alpine-linux)
- [Uninstalling nvm](#uninstalling-nvm)
- [Unsupported environments and alternatives](#unsupported-environments-and-alternatives)
- [Compatibility notes](#compatibility-notes)

## Install / update script

To install or update nvm, run the install script. Either of these works — the installer auto-detects between `git`, `curl`, and `wget`:

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

```sh
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

The script clones the nvm repository to `~/.nvm` (or `$XDG_CONFIG_HOME/nvm` if that variable is set), and attempts to add the source lines from the profile snippet below to the right profile file: `~/.bashrc`, `~/.bash_profile`, `~/.zshrc`, or `~/.profile`.

If the installer edits the wrong profile, set `$PROFILE` to the file's path and rerun the install script.

## Profile snippet

This is the snippet the install script adds. It is what makes `nvm` available in new shells:

```sh
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
```

The two lines do:
1. Set `$NVM_DIR` to `$HOME/.nvm` (default) or `$XDG_CONFIG_HOME/nvm` if XDG is set.
2. Source `nvm.sh` if it exists — this defines the `nvm` shell function in the current shell.

A simpler form (the README's manual-install snippet, fine when XDG isn't relevant):

```sh
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

### `--no-use` variant

If you want nvm available but **not** auto-using the default version when shells start, append `--no-use` to the source line:

```sh
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" --no-use
```

This speeds up shell startup. Users invoke `nvm use ...` explicitly when they need it.

## Install script env variables

You can customize the install by setting these before running the installer:

- `NVM_SOURCE` — alternate source URL (default: the GitHub repo)
- `NVM_DIR` — where to install (no trailing slash). Default: `$HOME/.nvm` or `$XDG_CONFIG_HOME/nvm`.
- `PROFILE` — explicit path to the profile file the installer should edit. Set to `/dev/null` to tell the installer not to edit any profile (useful when you manage completions/sourcing via a plugin manager like `zsh-nvm` or oh-my-zsh).
- `NODE_VERSION` — version of Node to install immediately after nvm itself

Example: install nvm without touching shell config:

```sh
PROFILE=/dev/null bash -c 'curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash'
```

## Verifying the install

```sh
command -v nvm
```

This should print `nvm`. **Do not use `which nvm`** — nvm is a sourced shell function, not an executable on `PATH`, so `which` returns nothing even when nvm is loaded correctly.

If `command -v nvm` returns nothing right after install, see the Linux / macOS troubleshooting sections below, or `troubleshooting.md`.

## Linux troubleshooting after install

If `nvm: command not found` or `command -v nvm` returns nothing:

- Close and reopen your terminal, then try again.
- Alternatively, manually source the profile file in the current shell:
  - bash: `source ~/.bashrc`
  - zsh: `source ~/.zshrc`
  - ksh: `. ~/.profile`

## macOS troubleshooting after install

Since OS X 10.9, `/usr/bin/git` is preset by Xcode command line tools, so the installer can't reliably detect whether git is installed. **Install Xcode command line tools manually** before running the install script. Otherwise the install can fail.

If `nvm: command not found` after the install script:

- **macOS 10.15+ default shell is zsh** but `~/.zshrc` doesn't exist by default. Create it (`touch ~/.zshrc`) and rerun the install script.
- **If you use bash**, your system may not have `~/.bash_profile` or `~/.bashrc`. Create one (`touch ~/.bash_profile` or `touch ~/.bashrc`), rerun the install script, then `. ~/.bash_profile` or `. ~/.bashrc`.
- **If you previously used bash but now have zsh installed**, manually add the profile snippet (above) to `~/.zshrc`, then `. ~/.zshrc`.
- Restart the terminal instance, or run `. ~/.nvm/nvm.sh`. Opening a new tab/window also picks up the new config.

If those don't fix it:

- If your `~/.bash_profile` (or `~/.profile`) doesn't source `~/.bashrc`, add `source ~/.bashrc` to it.
- Or paste the profile snippet from above directly into the right file (`~/.bash_profile`, `~/.zshrc`, `~/.profile`, or `~/.bashrc`).

For Apple Silicon Macs, see the Apple Silicon section in `troubleshooting.md` — node v15.3 added experimental support and v16.0 full support, but compiling earlier versions needs the Rosetta workflow.

## Ansible

```yaml
- name: Install nvm
  ansible.builtin.shell: >
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
  args:
    creates: "{{ ansible_env.HOME }}/.nvm/nvm.sh"
```

The `creates:` directive makes the task idempotent — it won't re-run if `nvm.sh` already exists.

## Installing in Docker (interactive)

When bash runs non-interactively (which Docker `RUN` steps do), the regular profile files are *not* sourced — so nvm isn't loaded, and `nvm install` / `nvm use` will fail in subsequent layers. The fix: tell bash to source a known file via `$BASH_ENV`.

```Dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV BASH_ENV "${HOME}/.bash_env"
RUN touch "${BASH_ENV}"
RUN echo '. "${BASH_ENV}"' >> ~/.bashrc

# Download and install nvm — point its installer at BASH_ENV so the source
# lines go where non-interactive bash will read them.
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | PROFILE="${BASH_ENV}" bash
RUN echo node > .nvmrc
RUN nvm install
```

## Installing in Docker for CI/CD

More robust pattern. Works in both interactive and non-interactive containers — based on sourcing nvm explicitly inside the `ENTRYPOINT`.

```Dockerfile
FROM ubuntu:latest
ARG NODE_VERSION=20

RUN apt update && apt install curl -y

RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash

ENV NVM_DIR=/root/.nvm

RUN bash -c "source $NVM_DIR/nvm.sh && nvm install $NODE_VERSION"

ENTRYPOINT ["bash", "-c", "source $NVM_DIR/nvm.sh && exec \"$@\"", "--"]
CMD ["/bin/bash"]
```

Override the version at build time:

```sh
docker build -t nvmimage --build-arg NODE_VERSION=19 .
```

Then both interactive and non-interactive runs have `node`/`npm`/`nvm` available:

```sh
docker run --rm -it nvmimage
docker run --rm -it nvmimage node -v
docker run --rm -it nvmimage npm -v
```

This pattern comes from issue #3531; refer to that thread if you need to tweak it.

## Git install

If you have `git ≥ 1.7.10`:

```sh
cd ~/
git clone https://github.com/nvm-sh/nvm.git .nvm
cd ~/.nvm
git checkout v0.40.4
. ./nvm.sh
```

Then add to `~/.bashrc`, `~/.profile`, or `~/.zshrc`:

```sh
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

## Manual install

Fully manual clone-and-source, picking up the latest tagged release:

```sh
export NVM_DIR="$HOME/.nvm" && (
  git clone https://github.com/nvm-sh/nvm.git "$NVM_DIR"
  cd "$NVM_DIR"
  git checkout `git describe --abbrev=0 --tags --match "v[0-9]*" $(git rev-list --tags --max-count=1)`
) && \. "$NVM_DIR/nvm.sh"
```

Then add the same source lines as the git-install above to your shell rc file.

## Manual upgrade

To upgrade a manual/git install to the latest tag:

```sh
(
  cd "$NVM_DIR"
  git fetch --tags origin
  git checkout `git describe --abbrev=0 --tags --match "v[0-9]*" $(git rev-list --tags --max-count=1)`
) && \. "$NVM_DIR/nvm.sh"
```

For installs done via the curl/wget script, just rerun the install script — it doubles as the updater.

## Alpine Linux

Alpine uses **musl** instead of glibc, so the pre-built Node binaries from nodejs.org won't run. You'll see "...does not exist" errors if you try `nvm install X` on plain Alpine.

To install nvm and build Node from source on Alpine, install the build deps first, then use `nvm install -s X` (the `-s` flag forces a source build):

### Alpine 3.13+

```sh
apk add -U curl bash ca-certificates openssl ncurses coreutils python3 make gcc g++ libgcc linux-headers grep util-linux binutils findutils
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

### Alpine 3.5 – 3.12

Use `python2` instead of `python3`:

```sh
apk add -U curl bash ca-certificates openssl ncurses coreutils python2 make gcc g++ libgcc linux-headers grep util-linux binutils findutils
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

Note the Alpine ↔ max-Node-version compatibility ceiling (newer Alpine builds more recent Node):

- Alpine 3.5 → up to Node v6.9.5
- Alpine 3.6 → v6.10.3
- Alpine 3.7 → v8.9.3
- Alpine 3.8 → v8.14.0
- Alpine 3.9 → v10.19.0
- Alpine 3.10 → v10.24.1
- Alpine 3.11 → v12.22.6
- Alpine 3.12 → v12.22.12
- Alpine 3.13 / 3.14 → v14.20.0
- Alpine 3.15 / 3.16 → v16.16.0

(These are main-branch versions.) For production Alpine workloads, consider `@mhart`'s pre-built `alpine-node` Docker images instead of building Node from source via nvm.

## Uninstalling nvm

```sh
nvm_dir="${NVM_DIR:-$HOME/.nvm}"
nvm unload
rm -rf "$nvm_dir"
```

Then edit `~/.bashrc` (or other shell rc file) and remove these lines:

```sh
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
[[ -r $NVM_DIR/bash_completion ]] && \. $NVM_DIR/bash_completion
```

## Unsupported environments and alternatives

nvm does not support these environments. Point users at the alternatives:

- **Native Windows (no WSL)** — use [`nvm-windows`](https://github.com/coreybutler/nvm-windows), [`nodist`](https://github.com/marcelklehr/nodist), or [`nvs`](https://github.com/jasongin/nvs). These are **separate projects** with different command surfaces; the things in `commands.md` here don't apply to them.
- **Fish shell** — see [`nvm.fish`](https://github.com/jorgebucaran/nvm.fish), [`fish-nvm`](https://github.com/FabioAntunes/fish-nvm), [`plugin-nvm`](https://github.com/derekstavis/plugin-nvm) for Oh My Fish, or [`bass`](https://github.com/edc/bass) to shim bash utilities. The fish auto-switch recipe in `commands.md` requires `bass`.
- **FreeBSD** — no official pre-built Node binary; building from source may require patches. See nvm issue #900 and nodejs issue #3716.

WSL is supported, but use **WSL2** specifically. Cygwin and Git Bash work but with caveats (path translation, symlinks, file permissions).

## Compatibility notes

- **C++ compiler required for source builds** — `build-essential` and `libssl-dev` on Debian/Ubuntu, Xcode (or Xcode CLT) on macOS.
- **Homebrew nvm is not supported.** If a user has it, `brew uninstall nvm` first, then install via the script. The maintainers will not help debug Homebrew installs.
- **`zsh-nvm` plugin** is a popular zsh-friendly wrapper. If the user already uses it, set `NVM_AUTO_USE=true` for automatic `.nvmrc` detection rather than adding the auto-switch snippets from `commands.md`.
- **`set -e`**, `prefix=...` in `~/.npmrc`, and `$NPM_CONFIG_PREFIX` / `$PREFIX` are known to break nvm — see `troubleshooting.md`.
