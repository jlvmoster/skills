# nvm command reference

Reference for nvm v0.40.4. Organized by purpose: install/uninstall versions, switching, listing, aliases, `.nvmrc`, package migration, mirrors, env vars, cleanup, auto-switch snippets.

## Table of contents

- [Installing a Node version](#installing-a-node-version)
- [Uninstalling a Node version](#uninstalling-a-node-version)
- [Switching versions](#switching-versions)
- [Listing versions](#listing-versions)
- [Special version aliases](#special-version-aliases)
- [LTS notation](#lts-notation)
- [Aliases](#aliases)
- [Setting the default](#setting-the-default)
- [.nvmrc](#nvmrc)
- [Migrating global packages](#migrating-global-packages)
- [npm version handling](#npm-version-handling)
- [Default global packages file](#default-global-packages-file)
- [Offline install](#offline-install)
- [Mirrors](#mirrors)
- [System Node](#system-node)
- [Cleanup commands](#cleanup-commands)
- [Cache commands](#cache-commands)
- [Colorized output](#colorized-output)
- [Environment variables exposed by nvm](#environment-variables-exposed-by-nvm)
- [Bash completion](#bash-completion)
- [Auto-switch on cd](#auto-switch-on-cd)

## Installing a Node version

```sh
nvm install <version>           # e.g. 14.7.0, 16.3.0, 12.22.1
nvm install node                # latest Node release
nvm install --lts               # latest LTS
nvm install --lts=argon         # latest from the "argon" LTS line
nvm install 'lts/*'             # same as --lts
nvm install lts/argon           # same as --lts=argon
```

The first version installed becomes the `default`. `nvm install` automatically `nvm use`s the version afterward.

**Compile from source** (instead of downloading the pre-built binary) — useful on Alpine (musl) or when the official binary is incompatible:

```sh
nvm install -s 0.8.6            # force source build
```

**Aliases work everywhere** — install / use / run / exec / which / ls-remote all accept aliases (e.g. `node`, `lts/*`, `default`, user-defined ones).

## Uninstalling a Node version

```sh
nvm uninstall <version>
nvm uninstall --lts             # latest LTS
nvm uninstall --lts=argon
nvm uninstall 'lts/*'
nvm uninstall lts/argon
```

## Switching versions

```sh
nvm use <version>               # switches current shell
nvm use --lts
nvm use --lts=argon
nvm use system                  # switch to the system-installed node
```

Other ways to run a specific version without persistently switching the shell:

```sh
nvm run 20 --version            # run `node --version` under v20
nvm run --lts -- script.js      # run a script under the latest LTS
nvm exec 4.2 node --version     # run any command in a subshell with this Node on PATH
nvm exec --lts npm test
nvm which 12.22                 # print path to that Node's binary
nvm current                     # show currently active version
```

## Listing versions

```sh
nvm ls                          # versions installed locally
nvm ls <pattern>                # filter installed by pattern
nvm ls-remote                   # all versions available to install
nvm ls-remote --lts             # only LTS versions
nvm ls-remote 'lts/*'
nvm ls-remote lts/argon
nvm ls --no-colors              # plain output, useful in scripts
TERM=dumb nvm ls                # also disables colors
```

## Special version aliases

These work anywhere a version is accepted:

- `node` — latest Node.js release
- `iojs` — latest io.js release (mostly historical; io.js merged into Node v4)
- `stable` — deprecated; only meaningful for Node ≤ v0.12. Currently an alias for `node`.
- `unstable` — deprecated; points at Node v0.11.
- `system` — the system-installed Node (the one outside nvm's control)
- `default` — whatever you've aliased as `default` (see [Setting the default](#setting-the-default))

## LTS notation

`lts/*` = the latest LTS line. `lts/<name>` = a specific LTS line (e.g. `lts/argon`, `lts/erbium`, `lts/hydrogen`, `lts/iron`). These work as arguments to `install`, `uninstall`, `use`, `run`, `exec`, `ls-remote`, `version-remote`, and in `.nvmrc`.

The LTS alias files live in `$NVM_DIR/alias/lts/`. **Do not edit them by hand** — nvm rewrites them every time it talks to nodejs.org, so your changes will be undone and may cause unsupported bugs.

## Aliases

```sh
nvm alias my_alias v14.4.0      # create or update an alias
nvm alias                       # list all aliases
nvm unalias my_alias            # remove an alias
```

Aliases must not contain spaces or slashes (except for the built-in `lts/*` form managed by nvm).

User-defined aliases live as files under `$NVM_DIR/alias/` — each file's contents is the version the alias maps to.

## Setting the default

The `default` alias is what new shells start with:

```sh
nvm alias default node          # latest installed Node
nvm alias default 18            # latest installed v18.x
nvm alias default 18.12         # latest installed v18.12.x
nvm alias default 'lts/*'       # follow the latest LTS line
```

If `nvm current` returns `system` even after setting `default`, the system Node's `PATH` is being applied after the `nvm.sh` source line. See `troubleshooting.md`.

## .nvmrc

A `.nvmrc` file pins a directory tree to a specific Node version. With no version on the command line, these commands all read `.nvmrc`:

- `nvm use`
- `nvm install`
- `nvm exec`
- `nvm run`
- `nvm which`

Examples:

```sh
echo "5.9" > .nvmrc
echo "lts/*" > .nvmrc            # pin to the latest LTS line
echo "node" > .nvmrc             # pin to the latest Node
echo "18.17.0" > .nvmrc          # pin to an exact version
```

**Format rules** (these are strict — see "validation" below):

- Exactly **one `<version>` followed by a newline**.
- `#` introduces a comment. The `#` and everything after it on the line is ignored.
- Blank lines, and leading/trailing whitespace, are ignored.
- `key=value` pairs are accepted and ignored, but **reserved for future use** — they may become validation errors. Don't write any.
- `<version>` accepts the same forms as the CLI: exact versions, partials (`18`, `18.12`), `node`, `lts/*`, `lts/<name>`.

**Directory traversal.** `nvm use` (and friends) walk *upward* from the current directory looking for `.nvmrc`. So a `.nvmrc` at the repo root applies in every subdirectory.

**Validation.** Run [`npx nvmrc`](https://npmjs.com/nvmrc) to validate a `.nvmrc` file. If `npx nvmrc` and nvm disagree on a file, one has a bug — file an issue.

**`nvm install` reads `.nvmrc` too.** Run `nvm install` (no version) in a project with a `.nvmrc` and it'll install + switch to the pinned version automatically. This is the friendliest onboarding command for new contributors.

**Watch out for Windows `cmd` heredocs.** If you generate `.nvmrc` from a Windows shell for a Linux deployment, drop the surrounding double quotes — `cmd` writes them into the file and breaks parsing.

## Migrating global packages

When installing a new Node version, bring globals along:

```sh
nvm install --reinstall-packages-from=current 'lts/*'
nvm install --reinstall-packages-from=node node
nvm install --reinstall-packages-from=5 6
nvm install --reinstall-packages-from=iojs v4.2
nvm install --reinstall-packages-from=default --latest-npm 'lts/*'
```

How it works: nvm resolves the source version (`current`, `default`, or whatever you pass), installs the new target, then runs `nvm reinstall-packages` to install the same global packages on the new version.

**`--reinstall-packages-from` does NOT update npm.** This is deliberate — it prevents npm from being silently upgraded to a version that's broken on the new Node. To update npm at the same time, add `--latest-npm`.

## npm version handling

```sh
nvm install-latest-npm           # bump npm on the current Node to the latest supported
nvm install --latest-npm <ver>   # install Node X and bump its bundled npm to the latest supported
```

**If you've already hit an "npm does not support Node.js" error**: see `troubleshooting.md` for the revert-uninstall-reinstall recovery.

## Default global packages file

Create `$NVM_DIR/default-packages` with one package per line. Every new `nvm install` will install these globally on the new version:

```
# $NVM_DIR/default-packages

rimraf
object-inspect@1.0.2
stevemao/left-pad
```

Anything `npm` accepts as a CLI argument works here (versions, git URLs, GitHub shorthands, scoped packages).

## Offline install

If the cache already has the tarball, `--offline` skips network entirely:

```sh
nvm install --offline 14.7.0
nvm install --offline --lts      # latest cached LTS; only works if `nvm ls-remote --lts` has populated the LTS aliases previously
```

Useful on planes, in air-gapped environments, or to avoid network latency on a known-cached version.

## Mirrors

```sh
export NVM_NODEJS_ORG_MIRROR=https://nodejs.org/dist
nvm install node

NVM_NODEJS_ORG_MIRROR=https://nodejs.org/dist nvm install 4.2
```

For io.js binaries:

```sh
export NVM_IOJS_ORG_MIRROR=https://iojs.org/dist
nvm install iojs-v1.0.3
```

Authorization header for the mirror (e.g., private artifact registry):

```sh
NVM_AUTH_HEADER="Bearer secret-token" nvm install node
```

`NVM_SYMLINK_CURRENT=true` — when set, `nvm use` creates a `$NVM_DIR/current` symlink. Helpful for IDEs that want a stable path to "the active Node." **Race-condition warning**: using this with multiple shell tabs running `nvm use` concurrently can produce inconsistent symlinks.

## System Node

```sh
nvm use system
nvm run system --version
```

Switches to the system-installed Node (the one outside nvm). Useful for testing scripts against a base OS install.

## Cleanup commands

```sh
nvm deactivate                  # remove nvm's PATH additions (restores pre-nvm PATH)
nvm unload                      # completely unload nvm from the current shell (used by uninstall)
```

`nvm deactivate` is reversible — sourcing `nvm.sh` again restores nvm. `nvm unload` removes the functions entirely; you'd source `nvm.sh` to get them back.

## Cache commands

```sh
nvm cache clear                 # delete cached downloads
nvm cache dir                   # print the cache directory
```

Run `nvm cache clear` if `nvm install` fails with something like `curl: (33) HTTP server doesn't seem to support byte ranges. Cannot resume.` — this is a partial-download in the cache.

## Colorized output

`nvm help`, `nvm ls`, `nvm ls-remote`, and `nvm alias` produce colorized output. Color codes (lowercase = normal, uppercase = bold):

```
r/R red       g/G green     b/B blue
c/C cyan      m/M magenta   y/Y yellow
k/K black     e/W light grey / white
```

Set five-character custom colors (in order: installed version, current, alias, default, system):

```sh
nvm set-colors rgBcm
```

Persist via env var in your profile:

```sh
export NVM_COLORS='cmgRY'
```

Disable colors entirely:

```sh
nvm ls --no-colors
TERM=dumb nvm ls
```

## Environment variables exposed by nvm

After sourcing nvm and switching to a version, nvm sets:

- `NVM_DIR` — nvm's installation directory
- `NVM_BIN` — where the active Node, npm, and global packages live (`$NVM_DIR/versions/node/<ver>/bin`)
- `NVM_INC` — Node's include directory (for building native addons)
- `NVM_CD_FLAGS` — internal, for zsh compatibility
- `NVM_RC_VERSION` — version from `.nvmrc` if one was used

Additionally, nvm modifies `PATH`, and, if present, `MANPATH` and `NODE_PATH`, when switching versions.

Configuration variables you can set:

- `NVM_DIR` — where nvm itself lives (set before sourcing)
- `NVM_NODEJS_ORG_MIRROR` — mirror for Node binaries
- `NVM_IOJS_ORG_MIRROR` — mirror for io.js binaries
- `NVM_AUTH_HEADER` — header for mirror requests
- `NVM_COLORS` — persistent custom colors
- `NVM_SYMLINK_CURRENT` — set to `true` to create a stable `current` symlink
- `NVM_DEBUG` — set to `1` for verbose internal logging

## Bash completion

Source bash completion alongside `nvm.sh`:

```sh
[[ -r $NVM_DIR/bash_completion ]] && \. $NVM_DIR/bash_completion
```

This must be **below** the `nvm.sh` source line. After sourcing, Tab-completes commands:

```
nvm <TAB>           # alias, install, use, ls, ls-remote, run, exec, ...
nvm alias <TAB>     # default, iojs, lts/*, lts/argon, lts/boron, ..., node, stable, unstable
nvm use <TAB>       # installed versions and user aliases
nvm uninstall <TAB> # installed versions and user aliases
```

## Auto-switch on cd

Recipes that run `nvm use` automatically when you `cd` into a directory with a `.nvmrc`. These are **contributed by nvm users** — they are not officially supported by the maintainers, but they are documented in the README and work well in practice. Paste into the user's rc file *after* the nvm source line, and review with them first.

### bash

Put at the end of `~/.bashrc`:

```bash
cdnvm() {
    command cd "$@" || return $?
    nvm_path="$(nvm_find_up .nvmrc | command tr -d '\n')"

    # If there are no .nvmrc file, use the default nvm version
    if [[ ! $nvm_path = *[^[:space:]]* ]]; then

        declare default_version
        default_version="$(nvm version default)"

        # If there is no default version, set it to `node`
        # This will use the latest version on your machine
        if [ $default_version = 'N/A' ]; then
            nvm alias default node
            default_version=$(nvm version default)
        fi

        # If the current version is not the default version, set it to use the default version
        if [ "$(nvm current)" != "${default_version}" ]; then
            nvm use default
        fi
    elif [[ -s "${nvm_path}/.nvmrc" && -r "${nvm_path}/.nvmrc" ]]; then
        declare nvm_version
        nvm_version=$(<"${nvm_path}"/.nvmrc)

        declare locally_resolved_nvm_version
        # `nvm ls` will check all locally-available versions
        # If there are multiple matching versions, take the latest one
        # Remove the `->` and `*` characters and spaces
        # `locally_resolved_nvm_version` will be `N/A` if no local versions are found
        locally_resolved_nvm_version=$(nvm ls --no-colors "${nvm_version}" | command tail -1 | command tr -d '\->*' | command tr -d '[:space:]')

        # If it is not already installed, install it
        # `nvm install` will implicitly use the newly-installed version
        if [ "${locally_resolved_nvm_version}" = 'N/A' ]; then
            nvm install "${nvm_version}";
        elif [ "$(nvm current)" != "${locally_resolved_nvm_version}" ]; then
            nvm use "${nvm_version}";
        fi
    fi
}

alias cd='cdnvm'
cdnvm "$PWD" || exit
```

### zsh

Put at the end of `~/.zshrc`, **after** nvm initialization:

```zsh
autoload -U add-zsh-hook

load-nvmrc() {
  local nvmrc_path
  nvmrc_path="$(nvm_find_nvmrc)"

  if [ -n "$nvmrc_path" ]; then
    local nvmrc_node_version
    nvmrc_node_version=$(nvm version "$(cat "${nvmrc_path}")")

    if [ "$nvmrc_node_version" = "N/A" ]; then
      nvm install
    elif [ "$nvmrc_node_version" != "$(nvm version)" ]; then
      nvm use
    fi
  elif [ -n "$(PWD=$OLDPWD nvm_find_nvmrc)" ] && [ "$(nvm version)" != "$(nvm version default)" ]; then
    echo "Reverting to nvm default version"
    nvm use default
  fi
}

add-zsh-hook chpwd load-nvmrc
load-nvmrc
```

After saving, `source ~/.zshrc`.

### fish

Requires [bass](https://github.com/edc/bass).

```fish
# ~/.config/fish/functions/nvm.fish
function nvm
  bass source ~/.nvm/nvm.sh --no-use ';' nvm $argv
end

# ~/.config/fish/functions/nvm_find_nvmrc.fish
function nvm_find_nvmrc
  bass source ~/.nvm/nvm.sh --no-use ';' nvm_find_nvmrc
end

# ~/.config/fish/functions/load_nvm.fish
function load_nvm --on-variable="PWD"
  set -l default_node_version (nvm version default)
  set -l node_version (nvm version)
  set -l nvmrc_path (nvm_find_nvmrc)
  if test -n "$nvmrc_path"
    set -l nvmrc_node_version (nvm version (cat $nvmrc_path))
    if test "$nvmrc_node_version" = "N/A"
      nvm install (cat $nvmrc_path)
    else if test "$nvmrc_node_version" != "$node_version"
      nvm use $nvmrc_node_version
    end
  else if test "$node_version" != "$default_node_version"
    echo "Reverting to default Node version"
    nvm use default
  end
end

# ~/.config/fish/config.fish
# Call on initialization, otherwise directory-switch listening won't work
load_nvm > /dev/stderr
```

For Fish users who'd rather not depend on bass, point them at `nvm.fish` or `fish-nvm` (listed in `install.md`).

## Other useful commands

```sh
nvm debug                       # diagnostic info: env, paths, tool versions
nvm --version                   # nvm's own version
nvm version <alias-or-version>  # resolve to a concrete version (without switching)
nvm version-remote <alias>      # resolve a remote alias (e.g. lts/*)
```
