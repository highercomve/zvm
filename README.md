# zvm

A lightweight Zig version manager. `zvm` is a single Bash script — no daemon, no
runtime, no plugins — that downloads, verifies, and switches between Zig
toolchains. It also handles ZLS (Zig Language Server) builds and can
auto-switch the active version when you `cd` into a project that pins
a `minimum_zig_version` in its `build.zig.zon`.

## Table of contents

- [What you get](#what-you-get)
- [Requirements](#requirements)
- [Installation](#installation)
- [How zvm lays things out on disk](#how-zvm-lays-things-out-on-disk)
- [Activating PATH (sourcing your shell)](#activating-path-sourcing-your-shell)
- [Subcommands](#subcommands)
  - [`zvm install`](#zvm-install)
  - [`zvm use`](#zvm-use)
  - [`zvm list`](#zvm-list)
  - [`zvm remove`](#zvm-remove)
  - [`zvm versions`](#zvm-versions)
  - [`zvm version`](#zvm-version)
  - [`zvm zls`](#zvm-zls)
  - [`zvm auto`](#zvm-auto)
  - [`zvm init`](#zvm-init)
- [Auto-switching from `build.zig.zon`](#auto-switching-from-buildzigzon)
- [Configuration](#configuration)
- [Supported platforms](#supported-platforms)
- [Troubleshooting](#troubleshooting)
- [Uninstalling](#uninstalling)
- [License](#license)

## What you get

- Install any released Zig toolchain (and `master`) for the detected
  architecture, with shasum verification against the official
  [`download/index.json`](https://ziglang.org/download/index.json).
- Switch the active toolchain by flipping a single symlink
  (`~/.zvm/current` → `~/.zvm/<version>`), so every shell picks it up
  with no per-shell `PATH` rewrites.
- Build and install [zls](https://github.com/zigtools/zls) matched to the
  active Zig version.
- Optional cd-time auto-switching driven by `build.zig.zon`'s
  `.minimum_zig_version`.

## Requirements

`zvm` is intentionally minimal. It needs:

- `bash`
- `curl`
- `jq`
- `sha256sum`
- `tar`, `find`, `grep`, `sed`, `awk` (always present on supported systems)
- `git` and `zig` itself (only for `zvm zls`)

If any of `curl`, `jq`, or `sha256sum` are missing, the script exits with an
explicit error pointing at the missing tool.

## Installation

Download the script and make it executable:

```bash
curl -o zvm https://raw.githubusercontent.com/highercomve/zvm/refs/heads/master/zvm
chmod +x zvm
```

Move it somewhere on your `PATH`:

```bash
sudo mv zvm /usr/local/bin/zvm
```

(On macOS with MacPorts, `/opt/local/bin` works too. On Homebrew systems,
`/usr/local/bin` is fine.)

Run it once with no arguments to print usage. The first invocation also
performs the one-time setup described below.

## How zvm lays things out on disk

```
~/.zvm/
├── current        → symlink to the active version, e.g. ~/.zvm/0.16.0
├── config         (optional; created by `zvm auto --save`)
├── 0.14.0/
│   ├── zig
│   ├── lib/...
│   └── zls        (if you ran `zvm zls`)
├── 0.16.0/
│   ├── zig
│   └── lib/...
└── master/
    └── ...
```

- Each installed toolchain lives in its own directory under `~/.zvm/`.
- `~/.zvm/current` is the single source of truth for "which version is
  active" — it's a symlink that gets flipped by `zvm use`, `zvm install`,
  or `zvm auto`.
- You can override the root directory by editing `ZVM_DIR` at the top of
  the script (the env-var override path is not auto-respected; the
  variable is set explicitly in the script).

## Activating PATH (sourcing your shell)

The first time you run `zvm`, it appends a small guarded block to your
rc files (only the ones that exist):

- `~/.bashrc`
- `~/.zshrc`
- `~/.config/fish/config.fish`

POSIX shells get:

```sh
# ZVM
if [ -d "$HOME/.zvm" ]; then
    case ":$PATH:" in
        *":$HOME/.zvm/current:"*) ;;
        *) export PATH="$PATH:$HOME/.zvm/current" ;;
    esac
fi
```

Fish gets the equivalent in its own syntax. The block is idempotent and is
skipped if `zvm` detects an existing reference (`.zvm/current`, `ZVM_DIR`,
`ZVM_PATH`, or a `zvm/{bin,env}` mention) in the rc file — so it won't
duplicate or fight a hand-edited setup.

After the first run, source your shell config (or open a new terminal):

```bash
source ~/.bashrc          # bash
source ~/.zshrc           # zsh
source ~/.config/fish/config.fish   # fish
```

## Subcommands

```
zvm [list|remove|install|use|auto|init|zls|versions|version]
```

### `zvm install`

Install a new Zig toolchain. Fetches `download/index.json`, verifies the
shasum, extracts into `~/.zvm/<version>/`, and switches to it
(by calling `select_run` internally).

```bash
zvm install [<version>] [<architecture>]
```

- `<version>` — e.g. `0.14.0`, `0.16.0`, `master`. Omit to pick interactively.
- `<architecture>` — e.g. `x86_64-linux`, `aarch64-macos`. Omit to auto-detect.

Examples:

```bash
zvm install                    # interactive, auto-detect arch
zvm install 0.16.0             # specific version, auto-detect arch
zvm install master x86_64-linux # explicit version + arch
```

Sample output:

```
🔍 Fetching Zig versions...
🔍 Detected system architecture: x86_64-linux
✅ Selected architecture: x86_64-linux
⬇️ Downloading x86_64-linux from https://ziglang.org/download/0.16.0/...
✅ Download complete.
🔍 Verifying signature for zig.tar.gz...
🔍 Verifying signature for zig.tar.gz expecting: <expected sha256>
✅ Signature verification successful.
🔧 Extracting zig.tar.gz...
✅ Extraction complete.
✅ Installed version 0.16.0.
🔧 zig version
0.16.0
```

If the requested version is already installed, `zvm install` exits with
an error rather than re-downloading.

### `zvm use`

Switch the active Zig toolchain.

```bash
zvm use <version>
zvm use                # interactive picker over installed versions
```

This only changes the `~/.zvm/current` symlink. It does **not** rewrite
`~/.bashrc` or any rc file (older versions of zvm did; that's been
removed because it clobbered user-customized blocks). If the symlink
isn't yet on `PATH` in your current shell, `zvm use` will tell you to
open a new shell or `export PATH=...`.

### `zvm list`

List installed versions. The active one is marked with `*`:

```
📜 Installed versions:
  0.14.0
* 0.16.0
  master
```

### `zvm remove`

Delete an installed version's directory.

```bash
zvm remove <version>
```

Safe-guarded: the `rm -rf` target uses `${ZVM_DIR:?}/${version:?}` so an
empty variable can never expand to `/`.

### `zvm versions`

List **all** versions available from `ziglang.org/download/index.json`,
newest first. This hits the network every time:

```bash
zvm versions
```

### `zvm version`

Print the zvm script's own version, plus the active Zig version if any.
Aliases: `--version`, `-v`.

```
zvm v0.0.1
zig 0.16.0 (/home/you/.zvm/current)
```

### `zvm zls`

Build [zls](https://github.com/zigtools/zls) against the currently active
Zig and install it next to it (`~/.zvm/current/zls`). Picks the zls git
branch that matches the active `zig version`.

```bash
zvm zls
```

Requires `git` and a working `zig` on `PATH`. Fails if `zls` is already
installed for the active version (delete `~/.zvm/current/zls` to retry).

### `zvm auto`

Read `./build.zig.zon` in the current directory, extract
`.minimum_zig_version`, and switch to it. If the version isn't installed,
download + install it first.

```bash
zvm auto                    # same as --global
zvm auto --global           # flip ~/.zvm/current (affects every shell)
zvm auto --local            # per-shell PATH; prints eval-able shell code
zvm auto --local bash       # explicit shell (bash|zsh|fish)
zvm auto --hook             # install the cd-time hook into your rc files
zvm auto --disable          # remove the cd hook
```

Modes (mutually exclusive — last one wins):

#### `--global` (default)

Flips the `~/.zvm/current` symlink to point at the resolved version. This
is the "set the system default" mode. Every shell that resolves `zig`
through `~/.zvm/current` (which is what the initial PATH setup arranges)
picks up the new version on its next command.

- **Already on target**: short-circuits with a one-line ack.
- **Installed but not active**: flips the symlink.
- **Not installed**: downloads, verifies, extracts, then flips.

#### `--local [shell]`

Per-shell mode. Instead of touching the symlink, prints shell code on
stdout that prepends `$ZVM_DIR/<version>` to `PATH` for the current
shell only. The injected path is tracked in `$ZVM_SHELL_PATH` so
subsequent calls in the same shell swap it out cleanly (no stale PATH
entries).

The shell defaults to whatever `$SHELL` reports (with a `ps`-based
fallback). Pass `bash`, `zsh`, or `fish` explicitly when scripting.

All log output goes to stderr in this mode so stdout stays clean for an
`eval`. That makes it naturally silent for the cd hook — errors are
discarded and `eval ""` is a no-op.

```bash
# Two terminals can sit in different projects without fighting:
#   terminal A: cd ~/project-a/   → 0.15.2 in *this* shell
#   terminal B: cd ~/project-b/   → 0.16.0 in *this* shell
#   ~/.zvm/current: unchanged, whatever you last set as default
```

**Self-applying**: when you've sourced the zvm shell integration (via
`zvm init <shell>` or `zvm auto --hook`), a `zvm()` function wraps the
binary and `eval`s the output automatically when `--local` is used. So:

```bash
zvm auto --local      # self-applies — no eval needed
```

Without the integration sourced, you'd run `eval "$(zvm auto --local)"`
to apply it. The wrapper falls through to the real binary for every
other subcommand, so it's transparent.

#### `--hook`

Installs the cd-time hook into your rc files (`.bashrc`, `.zshrc`,
`config.fish` — whichever exist) and writes `auto_hook=1` to
`~/.zvm/config`. The hook calls `zvm auto --local <shell>` and `eval`s
the result on every directory change.

Idempotent: re-running `--hook` won't duplicate entries. The hook block
is bracketed by a `# ZVM auto-switch hook` comment so `--disable` can
remove it precisely.

```bash
zvm auto --hook
exec $SHELL                       # or open a new terminal
cd ~/projects/my-zig-app          # → hook fires, this shell gets the pinned version
```

#### `--disable`

Removes the `auto_hook` line from `~/.zvm/config` and strips the hook
lines from your rc files. Only removes the exact lines zvm appended;
hand-edited blocks are untouched.

### `zvm init`

Print the shell integration block to source. It defines:

1. A `zvm()` function that wraps the real binary. When called as
   `zvm auto --local ...` it `eval`s the binary's output so the PATH
   change takes effect in your current shell. Every other call falls
   through to `command zvm`.
2. `__zvm_auto_hook`: runs `zvm auto --local <shell>` on every directory
   change, so cd into a project with a `build.zig.zon` switches that
   shell's Zig automatically.

```bash
zvm init bash | zsh | fish
```

You don't normally call this by hand — `zvm auto --hook` wires it into
your rc files for you. It's exposed so you can inspect what's being
installed, or pipe it into your config manually:

```bash
# bash / zsh
eval "$(zvm init bash)"
eval "$(zvm init zsh)"

# fish
zvm init fish | source
```

Implementation per shell:

- **bash**: prepends `__zvm_auto_hook` to `PROMPT_COMMAND`. Caches
  `$PWD`, so the hook only fires on an actual directory change — prompt
  latency stays flat.
- **zsh**: registers `__zvm_auto_hook` via `add-zsh-hook chpwd` and runs
  it once at sourcing time so the first shell starts on the right
  toolchain.
- **fish**: uses `function ... --on-variable PWD` and also fires once on
  sourcing.

## Auto-switching from `build.zig.zon`

Two flavors:

### Per-shell (cd hook) — recommended

```bash
zvm auto --hook
exec $SHELL    # or open a new terminal so the hook is loaded

cd ~/projects/my-zig-app
# build.zig.zon declares .minimum_zig_version = "0.16.0"
# → hook fires, runs `zvm auto --local <shell>`, evals the result
# → THIS shell's PATH now has .zvm/0.16.0 first
# → ~/.zvm/current is NOT touched; other terminals are unaffected
```

Two terminals can sit in two different projects with two different Zig
versions active at the same time — they don't fight, and nothing changes
the system default.

### Global (one-off) — for setting the system default

```bash
cd ~/projects/my-zig-app
zvm auto              # same as `zvm auto --global`
# → flips ~/.zvm/current to .zvm/0.16.0
# → every shell that uses .zvm/current picks it up on its next command
```

### Notes

- The parser is `grep -E | sed -E` over the `.minimum_zig_version` field.
  Multi-line and single-line zon formatting both work.
- `minimum_zig_version` is treated as an **exact** version, not a
  "minimum that any installed version >= satisfies". If you have 0.16.1
  installed and the project pins 0.16.0, zvm will install 0.16.0.
- Only `./build.zig.zon` (cwd) is checked. No walk-up-the-tree yet.
- `--local` mode is naturally silent — all log output goes to stderr,
  which the cd hook discards. Only directories that actually trigger a
  switch produce visible work (a one-line "installing" message if the
  pinned version isn't on disk yet).
- The injected path is tracked in `$ZVM_SHELL_PATH`. On the next switch
  in the same shell, the old entry is removed before the new one is
  prepended — so PATH doesn't grow with stale zvm entries.

## Configuration

`~/.zvm/config` is a plain shell-syntax key=value file. It's sourced by
`zvm` at startup. Today it holds at most one line:

```
auto_hook=1
```

This flag is a marker that `zvm auto --hook` has been run. You can edit
or delete it by hand. `zvm auto --disable` will remove just the
`auto_hook` line and leave anything else alone.

`ZVM_DIR` and `ZVM_CONFIG` are defined at the top of the script and can
be changed by editing it. `ZVM_VERSION` is exported by `select_run` for
use by sub-processes.

## Supported platforms

`zvm` detects the system architecture from `uname -s` / `uname -m`:

| OS         | Architecture            | Zig tarball name      |
|------------|-------------------------|------------------------|
| Linux      | x86_64                  | `x86_64-linux`         |
| Linux      | aarch64 / arm64         | `aarch64-linux`        |
| Linux      | armv7l                  | `armv7a-linux`         |
| Linux      | armv8l                  | `aarch64-linux`        |
| Linux      | riscv64                 | `riscv64-linux`        |
| Linux      | ppc64le                 | `powerpc64le-linux`    |
| Linux      | i686                    | `x86-linux`            |
| macOS      | x86_64                  | `x86_64-macos`         |
| macOS      | arm64                   | `aarch64-macos`        |
| FreeBSD    | x86_64                  | `x86_64-freebsd`       |
| Windows    | MINGW64 x86_64          | `x86_64-windows`       |
| Windows    | MINGW32 i686            | `x86-windows`          |

If `uname` doesn't match, you can still install by passing the
architecture explicitly: `zvm install <version> <arch>`.

## Troubleshooting

**`zvm: command not found`**
The script isn't on `PATH`. Move it into `/usr/local/bin` (or any
directory already on `PATH`), or add its location to `PATH` yourself.

**`sha256sum could not be found` / `jq could not be found` / `curl could not be found`**
Install the missing dependency. On Debian/Ubuntu: `sudo apt install
coreutils jq curl`. On macOS with Homebrew: `brew install jq coreutils`
(`sha256sum` ships in `coreutils`). On Fedora: `sudo dnf install jq curl
coreutils`.

**`Signature verification failed`**
The downloaded tarball doesn't match the shasum from
`download/index.json`. `zvm` deletes the partial download and exits. Try
again — usually a flaky connection. If it keeps failing on a specific
version, the upstream index may be inconsistent; pin to a different
version.

**`Version X already installed.`**
By design — `zvm install` refuses to overwrite. Use `zvm remove X` first,
or `zvm use X` to just switch to it.

**`No build.zig.zon in current directory.`**
You ran `zvm auto --global` (or plain `zvm auto`) in a directory without
one. Either `cd` into a Zig project, or use `--local` which exits
silently when there's no zon to read.

**`zvm auto` always reports the wrong version**
Make sure the `.minimum_zig_version = "..."` line in your
`build.zig.zon` is on a single line. The parser uses a simple regex and
won't follow line continuations.

**Hook installed but cd doesn't switch**
Open a fresh shell (or `source ~/.bashrc`) so the hook from
`zvm init <shell>` is loaded. The hook is added to your rc on
`zvm auto --save`, but it only takes effect in shells started after that
write.

**Re-running `zvm auto --hook` doesn't add hooks again**
That's the idempotency check — `zvm` skips appending when it sees
existing `zvm init <shell>` lines. If you removed them by hand, also
remove the surrounding `# ZVM auto-switch hook` comment, then re-run
`--hook`.

## Uninstalling

```bash
zvm auto --disable                # remove cd hook + config flag
rm -rf ~/.zvm                     # remove all installed toolchains
sudo rm /usr/local/bin/zvm        # remove the script
# then edit ~/.bashrc / ~/.zshrc / ~/.config/fish/config.fish to drop
# the `# ZVM` PATH block if you want to clear it up completely.
```

## License

Released under the [MIT License](./LICENSE). Copyright © 2024-2026 Sergio Marin.
