# 🦀 krabby

An aggressive, performance-obsessed wrapper around `cargo` that auto-tunes build profiles per binary. It probes whether C/C++ FFI dependencies actually benefit from cross-language LTO, caches the winning profile, and reuses it on future builds — so you don't pay probing costs more than once per crate.

Built for machines where you're willing to trade portability and compile time for the fastest binary the compiler can produce on *your* CPU.

## What it does

`krabby` is a single Bash script (`_krabby_main`) that replaces day-to-day `cargo build` / `cargo install` / `cargo update` usage with:

- **Native everything**: `-march=native`, `-C target-cpu=native`, full LTO, `mold` as the linker, and hardening flags (`FORTIFY_SOURCE=3`, stack protector, etc.) baked into every build.
- **Automatic profile selection** (`_krabby_compile`): on a cache miss, probes two build profiles per binary —
  - `crosslto` — cross-language LTO with clang/llvm-ar/llvm-ranlib as the C/C++ toolchain, useful when a crate has C/C++ FFI dependencies that benefit from being LTO'd together with the Rust code.
  - `rust` — pure Rust, no C/C++ toolchain involved.

  - For `install`, both profiles are compiled and the resulting binaries disassembled to count advanced x86 instructions (BMI2/AVX2 gather/scatter/permute ops) as a proxy for whether cross-LTO actually changed codegen. 
  - For `build`, instead of a second full compile, the incremental build is re-run with a 3-second timeout — if cargo rebuilds anything, C/C++ deps are involved and `crosslto` wins. If instruction counts are equal (or the incremental probe finishes instantly), `rust` is used. The winner is cached and all future builds skip straight to it.
- **Per-binary caching**: winning profiles are stored as individual files under `~/.cache/v3compile/<bin_name>` for `install` (e.g. `~/.cache/v3compile/ripgrep` contains just `rust` or `crosslto`), or in a local `.v3compile` file for project builds — so subsequent builds skip straight to the fast path instead of re-probing.
- **Crate update checking** (`krabby update`): checks `crates.io` for newer versions of every `cargo install`-ed binary and recompiles anything out of date.
- **Special-cased `sudo-rs` upgrades**: backs up the existing `sudo` binary, verifies the new one actually works before committing, and restores the backup automatically if compilation or verification fails.
- **Optional dotfile sync** (`KRABBY_SYNC_ENABLED`): can track the diff of installed cargo binaries into a pkglist file, similar to how Arch users track `pacman` package lists.

## Requirements

`krabby` is written for an **x86_64 Linux** system and assumes a fairly specific, performance-tuned toolchain. It is *not* portable out of the box — several flags (`-march=native`, the `x86_64-unknown-linux-gnu` target, LLVM tools) are hardcoded.

### Required

| Tool | Why |
|---|---|
| `bash` (4+) | The script itself (`#!/usr/bin/env bash`) |
| `rustc` + `cargo` | Core build tooling. A **stable** toolchain is sufficient — the script uses `-Z build-std`, `-Z unstable-options`, and `-Zmir-opt-level`, and gates them via `RUSTC_BOOTSTRAP=1`, which coerces a stable `rustc` into accepting nightly-only flags without needing a nightly toolchain or `rustup`. Install via your distro's package manager (e.g. `sudo pacman -S rust`) |
| `rust-src` | Required for `-Z build-std=std,panic_abort`. On Arch: `sudo pacman -S rust-src` |
| `x86_64-unknown-linux-gnu` target | Explicit `--target` is passed on every build. On Arch this is bundled with the `rust` package and available out of the box |
| `clang` / `clang++` | Used as `CC`/`CXX` for the `crosslto` profile |
| `llvm-ar`, `llvm-ranlib` | Used as `AR`/`RANLIB` for the `crosslto` profile |
| [`mold`](https://github.com/rui314/mold) | Linker, invoked via `-fuse-ld=mold` in `LDFLAGS` |
| `objdump` (binutils) | Disassembles built binaries to count SIMD/BMI2 instructions when probing profiles |
| `curl` | Fetches latest crate versions from crates.io in `krabby update` |
| `awk` | Version/index parsing in `krabby update` and binary path resolution |
| `xargs` (findutils) | Copying staged binaries in the probe step |
| coreutils (`cp`, `rm`, `mkdir`, `mktemp`, `tr`, `timeout`) | General file ops, temp dir management, instruction count filtering, and the 3-second incremental probe timeout |

### Optional

- A CPU that actually benefits from `-march=native` (BMI2/AVX2) — the probing logic assumes recent x86_64 hardware; on older CPUs the `crosslto` vs `rust` comparison may just always come back even.
- `sudo-rs` installed via `cargo install` if you want the automatic backup/verify/rollback handling in `_handle_sudo_rs` to matter.
- `$HOME/dotfiles/pkglist-cargo.txt` tracking, if you set `KRABBY_SYNC_ENABLED=1`.

### Installation

```bash
# Install Rust toolchain + build dependencies (Arch)
sudo pacman -S rust rust-src clang llvm mold binutils curl

# Install the script
curl -o ~/.local/bin/krabby https://raw.githubusercontent.com/elseawhy/krabby/refs/heads/main/krabby
chmod +x ~/.local/bin/krabby
```

> On Debian/Ubuntu: `sudo apt install rustc cargo rust-src clang llvm-dev mold binutils curl`

> **No `rustup` needed.** `RUSTC_BOOTSTRAP=1` coerces the stable system `rustc` into accepting nightly-gated flags (`-Z build-std`, `-Zmir-opt-level`, etc.). A stable toolchain from your distro's package manager is all that's required.

## Usage

```bash
krabby                    # Build the local project (release, autodetects Cargo.toml)
krabby install <crate>[@version]  # Install/compile a crate from crates.io with FFI profiling
krabby uninstall <crate>  # Uninstall a crate
krabby update             # Check crates.io and upgrade all installed binaries
krabby list                # List all cargo-installed binaries
krabby help                # Show usage
```

### First build of a crate/project

On a cache miss, `krabby` runs both profiles, disassembles the resulting binaries, and picks a winner:

```
$ krabby install ripgrep
Starting Krabby build for: ripgrep (Mode: install)
No cache found. Initiating FFI probes...
 [-] Compiling profile: [crosslto]...
 [-] Compiling profile: [noc (canary)]...
--- PROBE RESULT ---
crosslto       : 12
noc (canary)   : 12
--------------------
No C/C++ FFI benefits detected. Compiling pure rust profile...
 [-] Compiling profile: [rust]...
```

The winning profile is then cached, so the next `krabby install ripgrep` (or rebuild of the same project) skips straight past probing:

```
$ krabby install ripgrep
Starting Krabby build for: ripgrep (Mode: install)
Found global cache: [rust]
Skipping FFI probes. Fast-tracking...
 [-] Compiling profile: [rust]...
Final binary built via fast-path [rust].
```

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `KRABBY_SYNC_ENABLED` | `0` | Set to `1` to enable pkglist-cargo.txt syncing to `~/dotfiles` on install/uninstall |
| `XDG_CACHE_HOME` | `~/.cache` | Where the per-binary profile cache (`v3compile/<bin_name>`) is stored |

Per-project overrides: drop a `.v3compile` file (containing just `rust` or `crosslto`) in a project's root to skip probing for that project permanently.

## ⚠️ Caveats

- **Not portable**: binaries built with `-march=native` will only run correctly on the same (or a very similar) CPU. Don't ship these artifacts elsewhere.
- **Nightly-gated flags via `RUSTC_BOOTSTRAP=1`**: this bypasses the stable/nightly gate, so a plain stable `rustc` (from `pacman`, `apt`, etc.) is sufficient — no `rustup` or nightly toolchain needed. That said, it is inherently fragile across `rustc` versions — expect occasional breakage when Rust changes internals of `-Z build-std` or other unstable flags.
- **`sudo-rs` upgrade path** modifies binary ownership/permissions (`chown root:root`, `chmod 4755`) — review `_handle_sudo_rs` before relying on it in an unattended context.
- Deleting `~/.cache/v3compile/<bin_name>` (or the per-project `.v3compile`) forces re-probing on the next build.

## License

[MIT License](https://github.com/elseawhy/krabby/blob/main/LICENSE)
