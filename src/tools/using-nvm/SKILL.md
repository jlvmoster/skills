---
name: using-nvm
description: Use whenever the user is working with Node.js version management via nvm (Node Version Manager) — installing nvm, switching Node versions, setting up `.nvmrc`, troubleshooting "nvm: command not found", configuring auto-switch on `cd`, migrating global npm packages between Node versions, or handling platform-specific issues (Apple Silicon, WSL, Alpine, Docker, npm prefix conflicts). Trigger on phrases like "switch node version", "need node X for this project", ".nvmrc", "nvm not found", "set default node", and on any error message containing `nvm_` or referencing `$NVM_DIR`. Do NOT trigger for general Node.js coding, npm package installs, or other version managers (Volta, asdf, fnm, nvm-windows, nvs).
---

# Using nvm

nvm (Node Version Manager) is a **sourced shell function**, not a binary. It is installed per-user and operates per-shell — every new shell loads it from your profile, and version selection lives in the current shell's environment. That single fact explains the bulk of nvm's failure modes, so keep it in mind throughout this skill.

This skill helps you install nvm, drive it on the user's machine, and recover from its common failure modes. The three reference files cover everything in depth; this body covers the model and the 80% of daily commands.

## Diagnose first

Route to the right reference based on what the user is asking:

- **"nvm: command not found"**, `which nvm` returns nothing, `$NVM_DIR` errors, default not applying in new shells, npm "does not support Node.js" → `references/troubleshooting.md`
- **How do I do X with nvm** (install a version, switch, alias, `.nvmrc`, migrate globals, auto-switch on cd) → `references/commands.md`
- **Install or uninstall nvm itself**, Docker recipes, Ansible, Alpine, profile snippet, macOS Xcode CLT → `references/install.md`
- **Apple Silicon, WSL, Alpine, Docker, Fish, Windows-without-WSL** → cross-reference `install.md` for setup and `troubleshooting.md` for known gotchas

If you're not sure which reference applies, scan their headings — each is organized for fast lookup (commands.md by command, troubleshooting.md by symptom).

## Core mental model

These five facts apply everywhere and prevent most mistakes. Keep them in mind before suggesting commands or edits to the user's shell config.

1. **nvm is a shell function.** `which nvm` returns nothing because there is no binary on `PATH`. Use `command -v nvm` to confirm it's loaded. If that prints `nvm`, you're good; if it prints nothing, nvm isn't sourced in the current shell.

2. **nvm must be sourced, not executed.** The profile snippet (`[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"`) is what loads the function into a shell. Without it, nothing else in this skill works. Non-interactive shells (Docker `RUN`, CI, cron) don't read interactive profiles by default — they need a deliberate sourcing strategy (see `references/install.md`).

3. **Version selection is per-shell.** `nvm use 20` only affects the current shell. New shells start with whatever the `default` alias points to. If a user says "I set node 20 but a new terminal still has 18," they almost always need `nvm alias default 20`.

4. **`.nvmrc` is the cross-machine contract.** For repos, prefer a committed `.nvmrc` over a postinstall script. `nvm use` with no args reads `.nvmrc` from the current dir (and traverses upward), and so do `nvm install`, `nvm exec`, `nvm run`, `nvm which`. Format: one `<version>` followed by a newline; `#` comments and blank lines are ignored. `lts/*` and `node` are accepted.

5. **nvm and npm `prefix` don't mix.** `prefix=...` in `~/.npmrc`, or `$NPM_CONFIG_PREFIX` / `$PREFIX` in the environment, will silently break `nvm use`. So will `set -e` in shell startup. If `nvm use` "works" but the wrong node ends up on PATH, suspect these first.

Two more rules that have specific recovery paths in the troubleshooting reference but are worth flagging up front:

- **Do not use `sudo` with nvm-managed Node.** nvm installs into `$HOME/.nvm/versions/...` — there is no need for root, and using sudo creates root-owned files that break later `nvm use`.
- **Do not install nvm via Homebrew.** The maintainers explicitly do not support Homebrew installs. If the user has one, uninstall it (`brew uninstall nvm`) before reinstalling via the curl/wget script.

## Common one-liners

These are the 80% of daily use. Each has a fuller entry in `references/commands.md`.

```sh
# Install the latest LTS
nvm install --lts

# Switch to a version (current shell only)
nvm use --lts
nvm use 20
nvm use 18.17.0

# List what's installed / what's available
nvm ls
nvm ls-remote --lts

# Use the repo's pinned version (reads .nvmrc upward from cwd)
nvm use

# Pin a repo to a Node version
echo "lts/*" > .nvmrc        # or "20", "18.17.0", "node"

# Set the version new shells start with
nvm alias default 20         # or 'lts/*', or a specific version

# Bump and bring global packages along
nvm install --reinstall-packages-from=current 'lts/*'

# Confirm what's active right now
nvm current
node -v
```

## When acting on a user's machine

Practical guidance for driving nvm on a real developer's box:

- **Start with `command -v nvm`.** Confirm nvm is loaded in the *current* shell before running anything else. If it isn't, the rest of the session will be confusing. Send the user to `references/troubleshooting.md` (the "command not found" section).

- **In scripts, source nvm yourself.** Don't assume a script's shell has nvm loaded. Use the same snippet from the profile: `export NVM_DIR="${NVM_DIR:-$HOME/.nvm}"; [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"`. For Dockerfiles, see the `BASH_ENV` and `ENTRYPOINT` recipes in `references/install.md`.

- **Show diffs before touching dotfiles.** `~/.bashrc`, `~/.zshrc`, and `~/.profile` are personal files the user may have customized heavily. Propose the change, show what you'd add/remove, and ask before writing. Order matters too — if the user has a `/usr/local/bin` PATH set after the nvm source line, the default alias won't take effect (see `troubleshooting.md` for the symptom).

- **Prefer `.nvmrc` over scripting `nvm use`.** When the user says "make this repo use node X," writing `.nvmrc` is almost always the right answer. It's cross-machine, version-controlled, and works with everyone's auto-switch hooks.

- **Don't suggest sudo or Homebrew nvm.** Even if the user mentions them. Both lead to broken setups the maintainers won't help debug.

## Reference files

- **`references/install.md`** — Read when installing, updating, or uninstalling nvm itself, or when setting up nvm in Docker, CI, Ansible, Alpine, or macOS with Xcode CLT issues. Includes the profile snippet (both `XDG_CONFIG_HOME` and `HOME` variants), Git/manual install, and pointers to non-supported environments (Fish, native Windows).

- **`references/commands.md`** — Read when looking up how to do something with nvm. Organized by purpose: install/uninstall, switching, listing, aliases, `.nvmrc` rules, package migration, `default-packages` file, mirrors, environment variables, cleanup commands, and the auto-switch-on-cd snippets for bash/zsh/fish.

- **`references/troubleshooting.md`** — Read when the user reports a symptom. Organized by what the user actually pastes or sees: command-not-found cases, `which nvm` returning nothing, default alias not applying, npm/node incompatibility, byte-range cache errors, Apple Silicon compilation failures, vim `:!node -v` mismatch, WSL DNS, Alpine musl, npm prefix conflicts, `$HOME` case mismatch, Homebrew nvm, Fish, native Windows.

When a question spans multiple areas (e.g., "set up nvm + node 20 in a Dockerfile"), read both `install.md` (for the Docker recipe) and `commands.md` (for `.nvmrc` and version-selection details).
