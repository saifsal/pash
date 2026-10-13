# pash

`pash` is a password manager for the command line. Each password is a GnuPG-encrypted file in a directory you control. The password manager is a single POSIX `sh` script, so you can read every branch it takes. An optional Bash helper, `pashmenu`, picks an entry with `dmenu`.

It exists for people who want a small shell password manager: no `eval` on user input, a short code path, and GPG for encryption. Version `2.3.0` is the version the script prints. This repository is Saif Salih's copy of `pash`.

## Features

- `add`, `show`, `copy`, `del`, `list`, and `tree` for entries stored as `<name>.gpg`.
- Names with slashes become directories (`dev/github` is `dev/github.gpg`).
- Generate a password from `/dev/urandom`, or type one twice with echo turned off.
- Encrypt to a GPG key (`PASH_KEYID`) or with a symmetric passphrase.
- Copy an entry with a clipboard command, then run that command on empty input after a timeout.
- Optional `pashmenu` launcher: pick an entry in `dmenu` and copy it.
- Extend the command set with an ordinary shell function.

## Example

Entries are files in the store. A store with a few names looks like this:

```text
pash
├─ dev/
│  ├─ github
├─ internet/
│  ├─ graalians
│  ├─ pixeljoint
│  ├─ nixers
└──┘
```

`pash list` prints those names one per line (`dev/github`, `internet/nixers`, …). `pash tree` prints the same store through the `tree` command, with `.gpg` removed from the file names. This repository has no screenshots.

## Prerequisites

For `pash`:

- POSIX `sh` (`#!/bin/sh`)
- `gpg` or `gpg2`
- a terminal, for `add` and `del`

For clipboard copy, a clipboard program. The default is `xclip`. For `pash tree`, the `tree` command.

For the optional `pashmenu` helper:

- Bash
- `dmenu` on X11, or `dmenu-wl` on Wayland
- `pash` on `PATH`

`pashmenu --type` also looks for `xdotool` (X11) or `ydotool` (Wayland). That mode currently types nothing; see [pashmenu](#pashmenu).

## Install

Clone the repository and copy the scripts onto `PATH`:

```sh
git clone https://github.com/saifsal/pash.git
cd pash
mkdir -p ~/.local/bin
cp pash pashmenu ~/.local/bin/
chmod 755 ~/.local/bin/pash ~/.local/bin/pashmenu
```

`~/.local/bin` needs to be on `PATH`. Skip `pashmenu` if you only want the shell script. From the clone, `./pash` runs without installing.

Every subcommand creates that directory. `pash` alone and `pash -?` only print help:

```text
${PASH_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/pash}
```

When `XDG_DATA_HOME` is unset, that is `~/.local/share/pash`.

## Usage

```text
pash [ add|del|show|list|copy|tree ] [name]
```

| Command | What it does |
| --- | --- |
| `pash add <name>` | Create `<name>.gpg`. Asks whether to generate a password. |
| `pash show <name>` | Print the decrypted entry. |
| `pash copy <name>` | Write the entry to the clipboard, then clear it after the timeout. |
| `pash del <name>` | Ask `y/n`, then delete the entry. |
| `pash list` | Print every entry name. |
| `pash tree` | Print the store as a tree. Requires `tree`. |

The first letter is enough: `pash a`, `pash c`, `pash d`, `pash s`, `pash l`, and `pash t` call the same commands.

```sh
pash add dev/github
pash list
pash show dev/github
pash copy dev/github
pash del dev/github
```

`add` and `del` read a single `y` or `n` without waiting for Enter. Generating a password can print `tr: write error: Broken pipe` on stderr. `dd` stops reading after `PASH_LENGTH` bytes. The entry was still written when the next line is `Saved '…' to the store.`

`copy` prints `Clearing clipboard in "15" seconds.` and waits for that timeout before it exits. The clipboard command runs immediately; the wait is the clear timer. Disable it with `PASH_TIMEOUT=off`.

`show` and `copy` print or copy the stored bytes. A password created by `pash` includes a trailing newline.

To rename an entry, rename the file:

```sh
mv ~/.local/share/pash/old.gpg ~/.local/share/pash/new.gpg
```

`pash` with no arguments, or `pash -?`, prints the built-in help and exits 1.

## Configuration

Set these in the environment. A one-off prefix (`PASH_LENGTH=10 pash add github`) works the same way.

| Variable | Default | Role |
| --- | --- | --- |
| `PASH_DIR` | `$XDG_DATA_HOME/pash` or `~/.local/share/pash` | Store directory. |
| `PASH_KEYID` | unset | Recipient for `gpg --trust-model always -aer`. Unset means symmetric `gpg -c`, which asks for a passphrase. A key id or an email both work. |
| `PASH_LENGTH` | `50` | Generated password length, in bytes. |
| `PASH_PATTERN` | `_A-Z-a-z-0-9` | Character set passed to `tr -dc`. |
| `PASH_CLIP` | `xclip -sel c` | Command that reads the secret on stdin. Word-split on purpose. |
| `PASH_TIMEOUT` | `15` | Seconds before the clipboard command is run again with an empty stdin. `off` skips the clear. |

```sh
export PASH_KEYID=XXXXXXXX
export PASH_LENGTH=50
export PASH_PATTERN='_A-Z-a-z-0-9'
export PASH_DIR=~/.local/share/pash
export PASH_CLIP='xclip -sel c'
export PASH_TIMEOUT=15
```

`xclip -sel c` writes the X CLIPBOARD selection. A desktop clipboard manager may record that selection even after `pash` clears it. `PASH_CLIP` can be any command that reads stdin, for example a different `xclip` selection or `wl-copy`.

Encrypted files are created mode `0600`. The store directory itself uses your normal umask (`0755` when the umask is `022`).

## pashmenu

`pashmenu` lists every `*.gpg` file under the store and feeds the names to `dmenu`. Choosing one runs `pash copy` on it.

```sh
pashmenu
pashmenu -i -l 20          # extra arguments are passed to dmenu
```

It uses `dmenu-wl` when `WAYLAND_DISPLAY` is set, and `dmenu` when `DISPLAY` is set. With neither set it exits 1 and prints `Error: No Wayland or X11 display detected`.

Its default directory is `~/.local/share/pash` when `PASH_DIR` is unset. It does not read `XDG_DATA_HOME`. Export `PASH_DIR` if `pash` is using a different store.

`pashmenu --type` is meant to type the password with `xdotool` or `ydotool`. The script reads the decrypted password into the wrong variable and types an empty string, so `--type` does not insert the password.

## Migrating from pass

`pash` is not a tested drop-in replacement for [`pass`](https://www.passwordstore.org/). The layout is the same idea (one `.gpg` file per entry, a `.gpg-id` file in a `pass` store). To try an existing store, set the directory before reading the key id:

```sh
export PASH_DIR=${PASSWORD_STORE_DIR:-$HOME/.password-store}
read -r PASH_KEYID < "$PASH_DIR/.gpg-id"
export PASH_KEYID
```

## Extending

A shell function can add commands. This one handles `pash git …` inside the store and forwards everything else to the real script:

```sh
pash() {
    case $1 in
        g*)
            cd "${PASH_DIR:=${XDG_DATA_HOME:=$HOME/.local/share}/pash}"
            shift
            git "$@"
        ;;
        *)
            command pash "$@"
        ;;
    esac
}
```

The `g*` pattern matches any argument that starts with `g`, including `git`.

## Development and testing

There is no build and no test suite. From a clone:

```sh
sh -n pash
bash -n pashmenu
shellcheck -s sh pash
shellcheck -s bash pashmenu
```

`sh -n` and `bash -n` exit 0. ShellCheck 0.9.0 reports one info (SC2086) in `pash`, and warnings in `pashmenu` (including the `--type` variable mix-up). `shellcheck -S warning -s sh pash` exits 0.

To exercise GPG, use a temporary `GNUPGHOME` and `PASH_DIR`. The exact smoke test, and the rules for changing the scripts, are in [AGENTS.md](AGENTS.md).

`pash` is POSIX `sh` with four-space indentation. The password heredoc in `pw_add` is tab-indented because `<<-EOF` strips tabs. `pashmenu` is Bash with tab indentation. Keep those separate.

## Project status

The script identifies itself as `pash 2.3.0`. There is no changelog and no roadmap. The password manager commands are in place and can be run as documented above. There is no CI.

`pashmenu` can copy a selected entry. Its `--type` flag does not type the password.

## Licence

[MIT License](LICENSE.md). Copyright (c) 2016–2019 Dylan Araps. Copyright (c) 2022 Saif Salih.
