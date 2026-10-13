AGENTS.md is the canonical source for these instructions.
# pash

## Overview

`pash` is a command-line password manager written in POSIX `sh`. It stores each secret in its own GnuPG file and calls `gpg` or `gpg2` directly. The script prints its version as `pash 2.3.0`.

The repository is two executable scripts and no library, daemon, or config file. `pash` (240 lines) is the password manager. `pashmenu` is an optional Bash helper that chooses an entry with `dmenu`. Behaviour is controlled by environment variables and by the files in the password directory.

This repository is Saif Salih's copy of `pash` (GitHub description: "My fork of pash"). `LICENSE.md` is the MIT License and names Dylan Araps (2016–2019) and Saif Salih (2022).

There is no test suite, no CI, and no roadmap file. `pash` itself is a usable password manager. `pashmenu --type` currently types an empty string; see Gotchas.

## Repository layout

```
.
├── .github/
│   └── copilot-instructions.md
├── AGENTS.md     canonical agent instructions
├── CLAUDE.md     same text as AGENTS.md, plus a one-line header
├── LICENSE.md    MIT License
├── README.md     human-facing documentation
├── pash          POSIX sh password manager
└── pashmenu      optional Bash + dmenu launcher
```

There is no `Makefile`, package manifest, or test directory. Agent instructions are `AGENTS.md` (this file), `CLAUDE.md`, and `.github/copilot-instructions.md`.

## Setup

`pash` needs:

- A POSIX `sh`. The shebang is `#!/bin/sh`. On Debian and Ubuntu, `/bin/sh` is dash.
- `gpg` or `gpg2` on `PATH`. When both exist, the script uses `gpg2`.
- `/dev/urandom`, used when a password is generated.
- A terminal for `add` and `del`. Those commands call `stty` and read either one byte or a hidden line.

Optional:

- A clipboard command for `copy`. The default is `xclip -sel c`, overridable with `PASH_CLIP`.
- `tree`, required only by `pash tree`.

`pashmenu` needs:

- Bash (`#!/usr/bin/env bash`).
- A display. With `WAYLAND_DISPLAY` set it runs `dmenu-wl` and, for `--type`, `ydotool`. With `DISPLAY` set it runs `dmenu` and, for `--type`, `xdotool`.
- `pash` on `PATH`. The helper calls the `pash` command, not a path relative to this repository.

Nothing in this repository installs packages or builds artifacts. Put the scripts on `PATH`:

```sh
mkdir -p ~/.local/bin
cp pash pashmenu ~/.local/bin/
chmod 755 ~/.local/bin/pash ~/.local/bin/pashmenu
```

`~/.local/bin` must already be on `PATH`. From a clone, `./pash` works without copying.

On first use `pash` creates the store and `cd`s into it:

```text
${PASH_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/pash}
```

## Build, run and test

There is no build. The scripts run as they are.

Syntax checks, both verified to exit 0:

```sh
sh -n pash
bash -n pashmenu
```

`/bin/sh` here is dash, and `dash -n pash` exits 0 as well. There is no other automated test.

ShellCheck 0.9.0 is the linter the sources refer to. It is not wired into CI. Current results:

```sh
shellcheck -s sh pash          # exit 1
shellcheck -S warning -s sh pash   # exit 0
shellcheck -s bash pashmenu    # exit 1
```

`pash` has one finding: SC2086 (info) on the unquoted `GPG_TTY=${GPG_TTY:-$(tty)}` assignment. `pashmenu` reports SC2100 (warning) on the string assignment `dmenu=dmenu-wl`, and SC2034 plus SC2154 (warning) on the `--type` branch.

Show the built-in usage (exit 1):

```sh
./pash
./pash -?
```

Smoke test against a throwaway store. Do not point `PASH_DIR` or `GNUPGHOME` at real keys or passwords. `add` reads a key with `stty`, so run it in a terminal and answer `y`.

```sh
tmpdir=$(mktemp -d)
export GNUPGHOME="$tmpdir/gnupg"
export PASH_DIR="$tmpdir/store"
mkdir -m 700 "$GNUPGHOME"
cat > "$tmpdir/key.conf" << 'EOF'
%no-protection
Key-Type: RSA
Key-Length: 2048
Name-Real: pash test
Name-Email: pash-test@example.com
Expire-Date: 0
%commit
EOF
gpg --batch --generate-key "$tmpdir/key.conf"
PASH_KEYID=$(gpg --list-keys --with-colons | awk -F: '/^fpr:/ {print $10; exit}')
export PASH_KEYID
export PASH_LENGTH=12 PASH_PATTERN='a-z'

./pash add dev/github
./pash list
./pash show dev/github
PASH_TIMEOUT=off PASH_CLIP='cat' ./pash copy dev/github
./pash tree
```

Verified with `PASH_KEYID` set to a passphrase-less test key:

- `./pash` and `./pash -?` print `pash 2.3.0 - simple password manager.` and exit 1.
- `./pash add dev/github`, answering `y`, writes `dev/github.gpg` mode `0600` and prints `Saved 'dev/github' to the store.` `tr` may also print `write error: Broken pipe`.
- `./pash list` prints `dev/github`.
- `./pash show dev/github` prints the password and a trailing newline.
- `PASH_TIMEOUT=off PASH_CLIP='cat' ./pash copy dev/github` prints the password and exits 0.
- `./pash tree` prints a tree and strips the `.gpg` suffix. It needs `tree` on `PATH`.
- `./pash show nope` exits 1 with `error: Pass file 'nope' doesn't exist.`
- `./pash add` exits 1 with `error: Missing [name] argument.`
- `./pash add /tmp/x` exits 1 with `error: Category can't start with '/'.`
- `./pash add foo/../x` exits 1 with `error: Category went out of bounds.`
- `./pash add dev/github` a second time exits 1 with `error: Pass file 'dev/github' already exists.`
- With `PATH` limited to an empty directory, `./pash list` exits 1 with `error: GPG not found.`
- Answering `n` at the generate prompt, then entering the same password twice, stores that password. Two different passwords exit 1 with `error: Passwords do not match.`
- `./pash del dev/github` reads one byte. `n` keeps the file. `y` deletes it and removes the empty parent directory.
- `bash pashmenu` with `DISPLAY` and `WAYLAND_DISPLAY` unset exits 1 with `Error: No Wayland or X11 display detected`.

Any command that gets past the initial usage check and is not attached to a terminal also prints this on stderr. The exit status stays the command's own status:

```text
stty: 'standard input': Inappropriate ioctl for device
```

`./pash` and `./pash -?` exit before that trap is installed, so they do not print it.

## Code style and conventions

`pash`:

- POSIX `sh` only. Leave out arrays, `[[ ]]`, `source`, `eval`, and Bash substitutions.
- Shebang `#!/bin/sh`. Indent with four spaces.
- The heredoc in `pw_add` is the exception. The `$pass` line and the `EOF` terminator are indented with tabs because `<<-EOF` strips tabs. Spaces on those lines would be encrypted into the file, or would fail to end the heredoc.
- Functions are `pw_add`, `pw_del`, `pw_show`, `pw_copy`, `pw_list`, `pw_tree`, `yn`, `sread`, `glob`, `die`, `usage`, and `main`.
- `die` prints `error: %s.` to stderr and exits 1.
- The script ends with `set +x` and `set -f`. `set +x` keeps xtrace from leaking a secret. `set -f` disables globbing for the whole process.
- `PASH_CLIP` is expanded without quotes on purpose. The matching `# shellcheck disable=2086` stays.
- `sread` assigns a variable named by its argument. The matching `# shellcheck disable=2154` stays.
- `glob` compares with `case $1 in $2`. The matching `# shellcheck disable=2254` stays.
- Passwords go into `gpg` through a heredoc. The comments in `pw_add` explain why that avoids putting the secret on `argv`.

`pashmenu`:

- Bash, indented with tabs, with `shopt -s nullglob globstar`.
- Keep it a separate helper. A change to `pash` stays POSIX `sh`.

Keep existing ShellCheck disable comments unless the code they mark actually changes.

## CI and tooling

Continuous integration is not applicable: this repository has no GitHub Actions workflow, no other CI config, no pre-commit config, and no Makefile.

ShellCheck is the local linter. Run the commands in Build, run and test. There is no `.shellcheckrc`. Warning-level ShellCheck of `pash` is `shellcheck -S warning -s sh pash`. `pashmenu` fails ShellCheck today.

## Working rules for agents

- Keep changes small and focused.
- Run the tests before committing. The tests are `sh -n pash` and `bash -n pashmenu`. Run ShellCheck as well when a script changes.
- Never commit secrets. That includes password stores, `*.gpg` entries, private keys, and `GNUPGHOME` directories.
- Update the docs when behaviour changes. Update `README.md` and this file together.
- Use conventional commit messages (`docs:`, `fix:`, `feat:`, and so on).
- Keep `AGENTS.md`, `CLAUDE.md`, and `.github/copilot-instructions.md` in sync. `AGENTS.md` is canonical. The other two files carry the same text plus a one-line header. Keep them as separate files; do not replace them with symlinks.

## Gotchas

- The first argument is matched as a glob. `a*` adds, `c*` copies, `d*` deletes, `s*` shows, `l*` lists, and `t*` trees. Any other word prints usage and exits 1. `list` and `tree` ignore a name. `add`, `copy`, `del`, and `show` require one.
- `add` refuses an existing `<name>.gpg`. `copy`, `del`, and `show` refuse a missing one. `del` prompts `Delete pass file '<name>'? [y/n]:` and accepts a single `y` or `Y`. It then removes the file and runs `rmdir -p` on empty parent directories.
- `add` prompts `Generate a password? [y/n]:`. `y` reads `/dev/urandom`, filters with `LC_ALL=C tr -dc` on `PASH_PATTERN` (default `_A-Z-a-z-0-9`), and keeps `PASH_LENGTH` bytes (default 50) via `dd`. `dd` closes the pipe early, so `tr` often prints `write error: Broken pipe` while the entry is still saved. `n` asks for the password twice with echo disabled.
- The encrypted file is the password plus a trailing newline. `show` prints that newline.
- With `PASH_KEYID` unset, encryption is `gpg -c` and GnuPG asks for a symmetric passphrase. With `PASH_KEYID` set, the arguments are `--trust-model always -aer "$PASH_KEYID"`. Decryption is always `gpg -dq`.
- `GPG_TTY` is exported from `tty` when it is unset. `gpg2` is chosen over `gpg` when both are on `PATH`.
- `umask 077` is applied after `mkdir`. With a typical umask of `022`, the store and category directories are mode `0755`. The `.gpg` file created afterwards is mode `0600`.
- `pash` honours `XDG_DATA_HOME` when `PASH_DIR` is unset. `pashmenu` uses `prefix=${PASH_DIR-~/.local/share/pash}`: only an unset `PASH_DIR` falls back, and `XDG_DATA_HOME` is ignored. An empty `PASH_DIR` stays empty.
- A name that starts with `/` is rejected. A name that matches both `*/*` and `*../*` is rejected with `Category went out of bounds`. The check is those globs, not a canonicalized path. A name containing `/` is a category; `pash` creates the parent directory with `mkdir -p`.
- `copy` writes the decrypted secret to `PASH_CLIP`. Unless `PASH_TIMEOUT` is `off`, it also starts a background `sleep` (default 15) and then runs `PASH_CLIP` on an empty stdin. The `EXIT` trap calls `stty echo icanon`. Dash and bash wait for that sleep before the process exits, so `pash copy` blocks for the whole timeout. `PASH_TIMEOUT=off` skips the timer and returns immediately. The default clipboard command is only the CLIPBOARD selection (`xclip -sel c`). A clipboard manager can still keep its own history.
- `xclip` needs a display. Without one it prints `Error: Can't open display: (null)` and `pash copy` exits 1.
- On a non-terminal stdin, every command that installs the trap prints `stty: 'standard input': Inappropriate ioctl for device` on stderr.
- `pash tree` exits 1 with `error: 'tree' command not found.` when `tree` is missing. Otherwise it runs `tree --noreport` and strips a trailing `.gpg`.
- `pashmenu` exits 1 with `Error: No Wayland or X11 display detected` when neither display variable is set. `--type` has to be the first argument; further arguments are passed to `dmenu` or `dmenu-wl`. Without `--type`, the helper runs `pash c <name>` with stderr discarded, and that call still blocks for `PASH_TIMEOUT`. With `--type`, the pipeline is `pash s` piped into `IFS= read -r pash t` and then `printf %s "$pass"`. `$pass` is never assigned, so `xdotool` or `ydotool` receives an empty string.
- Leave `set -x` off while a real store is in use. The script forces `set +x` before `main`.
- Renaming an entry is renaming its file (`mv old.gpg new.gpg` inside the store). There is no rename command.
- `pass` compatibility is not guaranteed and is not tested in this repository. To point `pash` at a `pass` store, set `PASH_DIR` first, then read `PASH_KEYID` from `$PASH_DIR/.gpg-id`.
