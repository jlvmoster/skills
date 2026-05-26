# Troubleshooting nvm

Organized by **symptom** — what the user actually sees, types, or pastes. Find the symptom that matches, then apply the documented fix.

Every fix in this file comes from nvm's README or AGENTS.md (v0.40.4). If a symptom doesn't appear here and isn't obviously a variant of one of these, say so to the user — don't invent a fix.

## Table of contents

- [`nvm: command not found` after install](#nvm-command-not-found-after-install)
- [`which nvm` returns nothing](#which-nvm-returns-nothing)
- [Default alias not applying in new shells](#default-alias-not-applying-in-new-shells)
- [`npm` ERR! "does not support Node.js" after install](#npm-err-does-not-support-nodejs-after-install)
- [`curl: (33) HTTP server doesn't seem to support byte ranges`](#curl-33-http-server-doesnt-seem-to-support-byte-ranges)
- [Apple Silicon: old Node fails to install or run](#apple-silicon-old-node-fails-to-install-or-run)
- [vim `:!node -v` shows the system node](#vim-node--v-shows-the-system-node)
- [WSL: `curl: (6) Could not resolve host`](#wsl-curl-6-could-not-resolve-host)
- [Alpine: installed Node binary "does not exist"](#alpine-installed-node-binary-does-not-exist)
- [npm `prefix` conflicts](#npm-prefix-conflicts)
- [`set -e` breaks nvm](#set--e-breaks-nvm)
- [`$HOME` case mismatch on macOS](#home-case-mismatch-on-macos)
- [Homebrew nvm causing weirdness](#homebrew-nvm-causing-weirdness)
- [Homebrew "insecure directories" zsh warning](#homebrew-insecure-directories-zsh-warning)
- [Fish shell — nvm doesn't support it](#fish-shell--nvm-doesnt-support-it)
- [Windows user, not in WSL](#windows-user-not-in-wsl)
- [`sudo node` not finding nvm Node](#sudo-node-not-finding-nvm-node)
- [Source build fails on older Node versions](#source-build-fails-on-older-node-versions)
- [Concurrent installs across multiple shells](#concurrent-installs-across-multiple-shells)
- [`$NVM_DIR` with a trailing slash](#nvm_dir-with-a-trailing-slash)

## `nvm: command not found` after install

Most common nvm problem. The install script ran, but `nvm` isn't loaded in the current shell.

**Diagnose first.** Run `command -v nvm` (not `which nvm` — nvm is a shell function, not a binary). If it returns nothing, nvm isn't sourced.

**Linux:** Close and reopen your terminal, then try again. Alternatively, source the rc file in the current shell:

- bash: `source ~/.bashrc`
- zsh: `source ~/.zshrc`
- ksh: `. ~/.profile`

**macOS:** One of the following is usually the cause.

- **macOS 10.15+ has zsh as the default shell, but `~/.zshrc` doesn't exist by default.** Create it and rerun the install script:
  ```sh
  touch ~/.zshrc
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
  ```
- **bash user without `~/.bash_profile` or `~/.bashrc`.** Create one, rerun the installer, then source it:
  ```sh
  touch ~/.bash_profile         # or ~/.bashrc
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
  . ~/.bash_profile             # or . ~/.bashrc
  ```
- **Switched from bash to zsh.** Manually add the profile snippet from `references/install.md` to `~/.zshrc`, then `. ~/.zshrc`.
- **Try restarting the terminal** (new tab/window) or `. ~/.nvm/nvm.sh`.

If none of the above work:

- `~/.bash_profile` (or `~/.profile`) may not source `~/.bashrc`. Add `source ~/.bashrc` to it.
- Or paste the profile snippet from `references/install.md` directly into the right rc file.

If you find the install script edits the wrong profile, set `$PROFILE` to the correct path and rerun.

## `which nvm` returns nothing

This is **not a bug**. `nvm` is a sourced shell function, not an executable on `PATH`. `which` only finds executables.

Use `command -v nvm` instead. If that prints `nvm`, you're loaded; if it prints nothing, see "nvm: command not found" above.

## Default alias not applying in new shells

Symptom: you ran `nvm alias default 20`, but a new terminal still has the wrong Node (often the system one — `nvm current` returns `system`).

**Cause:** the system's Node `PATH` (typically `/usr/local/bin` or `/opt/homebrew/bin`) is being added *after* the `nvm.sh` source line in your shell profile, so it overrides the nvm-managed PATH.

**Fix:** ensure the nvm source line is *after* any PATH manipulation that adds system Node directories. Order the rc file so nvm's source line is one of the last things.

This is documented in nvm issue [#658](https://github.com/nvm-sh/nvm/issues/658).

## `npm` ERR! "does not support Node.js" after install

You installed a newer Node, and the bundled npm now refuses to run because it's too old (or, less commonly, too new). To recover:

1. Revert to the previous working Node version:
   ```sh
   nvm ls
   nvm use <your latest working version>
   ```
2. Uninstall the broken Node version:
   ```sh
   nvm uninstall <broken version>
   ```
3. Reinstall with `--latest-npm` so npm is updated alongside Node:
   ```sh
   nvm install --latest-npm <version>
   ```

To prevent this in the future, use `--latest-npm` whenever you migrate packages:

```sh
nvm install --reinstall-packages-from=default --latest-npm 'lts/*'
```

You can also bump npm on the current Node at any time:

```sh
nvm install-latest-npm
```

## `curl: (33) HTTP server doesn't seem to support byte ranges`

`nvm install` is trying to resume a partial download, but the server can't resume. The fix is to clear the cache and let nvm re-download from scratch:

```sh
nvm cache clear
nvm install <version>
```

## Apple Silicon: old Node fails to install or run

**Background.** Node v15.3 added experimental Apple Silicon (arm64) support; v16.0 made it official. Versions before that need Rosetta 2 (x86_64 emulation) to compile and run on M-series Macs.

**Rosetta workflow:**

1. Install Rosetta if you don't have it:
   ```sh
   softwareupdate --install-rosetta
   ```
2. Open a shell that's running through Rosetta:
   ```sh
   arch -x86_64 zsh
   ```
   (Or right-click Terminal/iTerm in Finder → Get Info → check "Open using Rosetta".)
3. Make sure nvm is sourced in this Rosetta shell. If your dotfiles only source nvm for your usual shell, do it manually:
   ```sh
   export NVM_DIR="${NVM_DIR:-$HOME/.nvm}"
   source "$NVM_DIR/nvm.sh"
   ```
4. Install the older Node with `--shared-zlib`:
   ```sh
   nvm install v12.22.1 --shared-zlib
   ```
   The `--shared-zlib` flag is important. Without it, the install may succeed but later `npm install` operations will fail with `incorrect data check` errors, due to a bug in recent Apple `clang` versions. See nodejs issue [#39313](https://github.com/nodejs/node/issues/39313).
5. Exit back to your native shell:
   ```sh
   exit
   arch              # arm64 (or i386 if you used the Get Info checkbox instead of `arch -x86_64`)
   ```
6. Confirm the architecture:
   ```sh
   node -p process.arch
   # x64
   ```

Now the older Node works normally under Rosetta.

## vim `:!node -v` shows the system node

Symptom: in your shell, `nvm use 6.2.1` sets node to v6.2.1; in vim, `:!node -v` reports the system version (e.g., `v0.12.7`).

**Fix:**

```sh
sudo chmod ugo-x /usr/libexec/path_helper
```

This stops `path_helper` from re-running and replacing the PATH that nvm set. Background: [dotphiles/dotzsh#mac-os-x](https://github.com/dotphiles/dotzsh#mac-os-x).

## WSL: `curl: (6) Could not resolve host`

Symptom: `curl -o-https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash` fails with `Could not resolve host: raw.githubusercontent.com`. You can `ping 8.8.8.8` but not `ping google.com`. Antivirus, VPN, or WSL's DNS handling are common causes.

**Fix:**

```sh
sudo rm /etc/resolv.conf
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "[network]" > /etc/wsl.conf'
sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'
sudo chattr +i /etc/resolv.conf
```

This replaces the auto-generated `resolv.conf` with one pointing at Google DNS, and tells WSL not to regenerate it. Verify:

```sh
cat /etc/resolv.conf
```

## Alpine: installed Node binary "does not exist"

Alpine uses **musl**, not glibc. The pre-built Node binaries nvm downloads are linked against glibc, so they fail with "...does not exist" errors on Alpine.

**Fix:** install build dependencies and force a source build with `nvm install -s`. The full apk command (3.13+) is:

```sh
apk add -U curl bash ca-certificates openssl ncurses coreutils python3 make gcc g++ libgcc linux-headers grep util-linux binutils findutils
nvm install -s <version>
```

For Alpine 3.5–3.12, replace `python3` with `python2`. See `references/install.md` for the Alpine-version ↔ max-Node-version compatibility table.

For production Alpine workloads, consider [`@mhart`'s `alpine-node` Docker images](https://github.com/mhart/alpine-node) instead of source-building via nvm.

## npm `prefix` conflicts

The single biggest "nvm seems to work but the wrong node is on PATH" cause. Check **three** things:

1. `~/.npmrc` — must not contain `prefix='some/path'`. Remove that line if present.
2. Environment variables — `$NPM_CONFIG_PREFIX` and `$PREFIX` must not be set. Unset them:
   ```sh
   unset NPM_CONFIG_PREFIX
   unset PREFIX
   ```
3. See "set -e breaks nvm" below.

Background: nvm issues [#606](https://github.com/nvm-sh/nvm/issues/606) and [#1245](https://github.com/nvm-sh/nvm/issues/1245).

## `set -e` breaks nvm

`set -e` (the "exit on error" shell option) in your profile or startup scripts causes nvm to fail in subtle ways — nvm internally uses commands whose non-zero exit codes are not actual errors.

**Fix:** remove `set -e` from your shell profile. If you need it in scripts that source nvm, set it *after* sourcing nvm.sh.

## `$HOME` case mismatch on macOS

Symptom: nvm seems to work but `nvm use` doesn't fully switch, and you can't quite tell why. You've eliminated `~/.npmrc` and the prefix env vars.

**Cause:** the user directory name in `$HOME` is capitalized differently from the actual directory under `/Users/`. For example, `$HOME=/Users/Alex` while `ls /Users/` shows `alex`.

**Fix:** rename the user directory and/or the account so the casing matches everywhere. Apple's docs: [Change the name of your macOS user account and home folder](https://support.apple.com/en-us/HT201548).

Background: nvm issue [#2261](https://github.com/nvm-sh/nvm/issues/2261).

## Homebrew nvm causing weirdness

The nvm maintainers do **not** support Homebrew installations. If a user has Homebrew nvm and reports weird behavior:

```sh
brew uninstall nvm
```

Then install via the official curl/wget script (see `install.md`). Don't try to debug Homebrew installs — the maintainers will close issues that originate from one.

## Homebrew "insecure directories" zsh warning

```
zsh compinit: insecure directories, run compaudit for list.
Ignore insecure directories and continue [y] or abort compinit [n]?
```

This is a Homebrew problem (insecure perms on `/usr/local/share/zsh/site-functions` etc.), **not** an nvm problem. Refer the user to [zsh-completions issue #680](https://github.com/zsh-users/zsh-completions/issues/680) for fixes.

## Fish shell — nvm doesn't support it

nvm does not work natively in Fish (the project explicitly does not support it; see nvm issue [#303](https://github.com/nvm-sh/nvm/issues/303)). Recommend one of:

- [`nvm.fish`](https://github.com/jorgebucaran/nvm.fish) — Fish-native rewrite (different project, different commands)
- [`fish-nvm`](https://github.com/FabioAntunes/fish-nvm) — wrapper around nvm, lazy-loaded
- [`plugin-nvm`](https://github.com/derekstavis/plugin-nvm) for [Oh My Fish](https://github.com/oh-my-fish/oh-my-fish)
- [`bass`](https://github.com/edc/bass) — lets you shim bash utilities (including nvm itself) into Fish. The fish auto-switch recipe in `commands.md` depends on bass.
- [`fast-nvm-fish`](https://github.com/brigand/fast-nvm-fish) — version-numbers-only but doesn't slow down shell startup

These are **separate projects** with different command surfaces. The rest of this skill (which covers upstream nvm) doesn't necessarily apply to them.

## Windows user, not in WSL

nvm doesn't run on native Windows. Point the user at one of the Windows-native alternatives — **these are separate projects**, not maintained by the nvm team:

- [`nvm-windows`](https://github.com/coreybutler/nvm-windows) — most popular, similar CLI but different internals
- [`nvs`](https://github.com/jasongin/nvs) — cross-platform (Windows + POSIX), Microsoft-affiliated
- [`nodist`](https://github.com/marcelklehr/nodist) — older, less active

If they have WSL available, recommend WSL2 + this nvm instead (WSL1 has compatibility issues — `install.md` covers the WSL setup).

## `sudo node` not finding nvm Node

Don't use `sudo` with nvm-managed Node. nvm installs into `$HOME/.nvm/versions/...` — there's no need for root privileges, and using sudo creates root-owned files that will break subsequent `nvm use` operations.

When using nvm, install global packages without sudo:

```sh
npm install -g grunt        # NOT sudo npm install -g grunt
```

If the user has a "system" Node and a `~/.npmrc` with a `prefix=` line, remove that line — it's incompatible with nvm. See "npm prefix conflicts" above.

You can keep a previous system Node install alongside nvm, but be aware: other users on the machine will use `/usr/local/lib/node_modules/*`, while your user gets `~/.nvm/versions/node/vX.X.X/lib/node_modules/*`. This causes version mismatches across users.

Background: nvm issue [#43](https://github.com/nvm-sh/nvm/issues/43).

## Source build fails on older Node versions

When installing very old Node from source, the official binary may be incompatible with shared libs on modern systems. Force a source build:

```sh
nvm install -s 0.8.6
```

This is the same `-s` flag used for Alpine; whenever the pre-built binary doesn't work on the user's platform, `-s` builds from source. Requires a C++ compiler: Xcode (or Xcode CLT) on macOS, `build-essential` + `libssl-dev` on Debian/Ubuntu.

## Concurrent installs across multiple shells

Multiple shells running `nvm install` concurrently can conflict (race conditions on the cache and version directory). If a user reports flaky installs and they have many terminals open running nvm operations, suggest serializing them.

Setting `NVM_SYMLINK_CURRENT=true` makes this worse — concurrent `nvm use` calls in different shells will write to the same `current` symlink. If a user has that variable set and reports symlink weirdness, suggest unsetting it.

## `$NVM_DIR` with a trailing slash

`NVM_DIR` must not contain a trailing slash. If the user has `export NVM_DIR="$HOME/.nvm/"`, fix it to `export NVM_DIR="$HOME/.nvm"`. This is a documented constraint in the installer.

## When in doubt: `nvm debug`

```sh
nvm debug
```

Prints environment info, paths, and tool versions. If the user reports a problem not covered above, ask them to share `nvm debug` output rather than guessing — it surfaces shell, OS, NVM_DIR, NVM_BIN, NVM_INC, and what versions are installed.

You can also enable verbose internal logging:

```sh
NVM_DEBUG=1 nvm install <version>
```

If the symptom truly isn't documented in nvm's README or in this skill, tell the user so — don't invent a fix. Point them at the [nvm issue tracker](https://github.com/nvm-sh/nvm/issues) and have them include `nvm debug` output.
