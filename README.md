# 🦀 krabby

An aggressive, performance-obsessed wrapper around `cargo` that auto-tunes build profiles per binary. It probes whether C/C++ FFI dependencies actually benefit from cross-language LTO, caches the winning profile, and reuses it on future builds — so you don't pay probing costs more than once per crate.

Built for machines where you're willing to trade portability and compile time for the fastest binary the compiler can produce on *your* CPU.

## What it does

`krabby` is a single Bash script (`_krabby_main`) that replaces day-to-day `cargo build` / `cargo install` / `cargo update` usage with:

- **Native everything**: `-march=native`, `-C target-cpu=native`, full LTO, `mold` as the linker, and hardening flags (`FORTIFY_SOURCE=3`, stack protector, etc.) baked into every build.
- **Automatic profile selection** (`_krabby_compile`): on a cache miss, probes two build profiles per binary —
  - `crosslto` — cross-language LTO with clang/llvm-ar/llvm-ranlib as the C/C++ toolchain, useful when a crate has C/C++ FFI dependencies that benefit from being LTO'd together with the Rust code.
  - `rust` — pure Rust, no C/C++ toolchain involved.

  - For `install`, both profiles are compiled and the resulting binaries disassembled to count advanced instructions as a proxy for whether cross-LTO actually changed codegen (e.g., BMI2/AVX2 ops on x86_64). On non-x86_64 hosts, the script is designed to support any architecture by auto-detecting the host target, but the current instruction-counting heuristic is x86_64-specific (it will conservatively fall back to the `rust` profile). Contributors on ARM and other hardware are welcome to help expand these heuristics!
  - For `build`, instead of a second full compile, the incremental build is re-run with a 3-second timeout — if cargo rebuilds anything, C/C++ deps are involved and `crosslto` wins. If instruction counts are equal (or the incremental probe finishes instantly), `rust` is used. The winner is cached and all future builds skip straight to it.
- **Per-binary caching**: winning profiles are stored as individual files under `~/.cache/v3compile/<bin_name>` for `install` (e.g. `~/.cache/v3compile/ripgrep` contains just `rust` or `crosslto`), or in a local `.v3compile` file for project builds — so subsequent builds skip straight to the fast path instead of re-probing.
- **Crate update checking** (`krabby update`): checks `crates.io` for newer versions of every `cargo install`-ed binary and recompiles anything out of date.
- **Special-cased `sudo-rs` upgrades**: backs up the existing `sudo` binary, verifies the new one actually works before committing, and restores the backup automatically if compilation or verification fails.
- **Optional dotfile sync** (`KRABBY_SYNC_ENABLED`): can track the diff of installed cargo binaries into a pkglist file, similar to how Arch users track `pacman` package lists.

## Requirements

`krabby` targets **Linux** systems. The host target triple is now auto-detected via `rustc -vV`, making the core build pipeline portable across architectures (x86_64, AArch64, RISC-V, etc.). The current FFI probe heuristic inspects BMI2/AVX2 instructions and is **x86_64-specific** — on other architectures it always returns 0 and krabby conservatively defaults to the `rust` profile while still applying all other aggressive optimizations (`-march=native`, `-C target-cpu=native`, LLVM tools). **Contributors with ARM or other non-x86_64 hardware are highly encouraged to submit PRs to expand the FFI probing heuristics!**

### Required

| Tool | Why |
|---|---|
| `bash` (4+) | The script itself (`#!/usr/bin/env bash`) |
| `rustc` + `cargo` | Core build tooling. A **stable** toolchain is sufficient — the script uses `-Z build-std`, `-Z unstable-options`, and `-Zmir-opt-level`, gated via `RUSTC_BOOTSTRAP=1`. **On Arch Linux** (or any distro shipping the latest stable): `sudo pacman -S rust` (or your package manager's equivalent) is recommended. **On all other distros**: use [rustup](https://rustup.rs) — `apt`/`dnf` often lag months behind upstream and may ship a version too old for some flags. |
| `rust-src` | Required for `-Z build-std=std,panic_abort`. On Arch: `sudo pacman -S rust-src`. With rustup: `rustup component add rust-src` |
| Host target triple | Auto-detected at runtime via `rustc -vV` — no manual configuration needed. Bundled with the `rust` package on Arch; available by default with rustup. |
| `clang` / `clang++` | Used as `CC`/`CXX` for the `crosslto` profile |
| `llvm-ar`, `llvm-ranlib` | Used as `AR`/`RANLIB` for the `crosslto` profile |
| [`mold`](https://github.com/rui314/mold) | Linker, invoked via `-fuse-ld=mold` in `LDFLAGS` |
| `objdump` (binutils) | Disassembles built binaries to count SIMD/BMI2 instructions when probing profiles |
| `curl` | Fetches latest crate versions from crates.io in `krabby update` |
| `awk` | Version/index parsing in `krabby update` and binary path resolution |
| `xargs` (findutils) | Copying staged binaries in the probe step |
| coreutils (`cp`, `rm`, `mkdir`, `mktemp`, `tr`, `timeout`) | General file ops, temp dir management, instruction count filtering, and the 3-second incremental probe timeout |

### Optional

- A CPU that actually benefits from `-march=native` (e.g., AVX2/BMI2 on x86_64, or NEON/SVE on ARM). The current probing logic assumes recent x86_64 hardware; on older CPUs or other architectures, the `crosslto` vs `rust` comparison may just always come back even or fallback to `rust`.
- `sudo-rs` installed via `cargo install` if you want the automatic backup/verify/rollback handling in `_handle_sudo_rs` to matter.
- `$HOME/dotfiles/pkglist-cargo.txt` tracking, if you set `KRABBY_SYNC_ENABLED=1`.

### Installation

```bash
# Install Rust toolchain + build dependencies (Arch or other rolling-release distros)
sudo pacman -S rust rust-src clang llvm mold binutils curl

# Install the script
curl -o ~/.local/bin/krabby https://raw.githubusercontent.com/elseawhy/krabby/refs/heads/main/krabby
chmod +x ~/.local/bin/krabby
```

> **Users on distros without the latest stable Rust** — use [rustup](https://rustup.rs) instead of your distro's Rust package. `apt`/`dnf` often ship outdated stable versions. After installing rustup, add `rust-src` with `rustup component add rust-src`, then install the remaining native dependencies via your package manager:
> ```bash
> # Debian/Ubuntu
> sudo apt install clang llvm-dev mold binutils curl
> ```

> **`rustup` vs `RUSTC_BOOTSTRAP=1`** — these solve different problems. `rustup` is recommended to ensure you have a *recent* stable `rustc` (distro packages can lag months behind). `RUSTC_BOOTSTRAP=1` is a separate mechanism that coerces any stable `rustc` into accepting nightly-gated flags (`-Z build-std`, `-Zmir-opt-level`, etc.) — no nightly toolchain or `rustup override` is needed for that.

## Usage

- `krabby [args...]` - Build the local project (auto-detects Cargo.toml). Arguments are passed directly to `cargo build`. Note: `--release` and `--locked` are automatically enabled.
- `krabby install <crate> [args...]` - Install/compile a crate from crates.io with FFI profiling. You can pass any `cargo install` flags (like `--features`) after the crate name. Note: `--locked` is automatically enabled.
- `krabby uninstall <crate>` - Uninstall a crate
- `krabby update` - Check crates.io and upgrade all installed binaries
- `krabby list` - List all cargo-installed binaries
- `krabby help` - Show usage

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

## Build Flags Reference

This section documents every flag set by krabby. All flags are applied through environment variables passed to `cargo` during compilation.

---

### `LDFLAGS` — Linker flags (passed to the C/C++ linker)

```
-march=native -fuse-ld=mold -Wl,-O1,--sort-common,--as-needed,-z,relro,-z,now,-z,pack-relative-relocs,-plugin-opt=O3,-plugin-opt=mcpu=native -flto
```

| Flag | Effect |
|---|---|
| `-march=native` | Emits CPU instructions tuned for the exact CPU this machine has — enables AVX2, BMI2, etc. on x86_64, and equivalent extensions like NEON/SVE on ARM. Passed to the linker so that LTO-time code generation uses the same target arch as compilation. |
| `-fuse-ld=mold` | Selects [`mold`](https://github.com/rui314/mold) as the linker. mold is significantly faster than `ld` or `lld` and supports all modern ELF features. |
| `-Wl,-O1` | Tells the linker to do basic optimizations (e.g. merging identical sections). |
| `-Wl,--sort-common` | Sorts common symbols by alignment to reduce padding in the BSS section. |
| `-Wl,--as-needed` | Only links libraries that are actually referenced. Eliminates unused shared library dependencies from the final binary. |
| `-Wl,-z,relro` | Makes certain ELF segments (GOT, `.init_array`, etc.) read-only after the dynamic linker is done with them — part of the RELRO hardening technique. |
| `-Wl,-z,now` | Forces all PLT relocations to be resolved at startup ("full RELRO"). Combined with `-z,relro`, this makes the GOT fully read-only at runtime, thwarting GOT overwrite attacks. |
| `-Wl,-z,pack-relative-relocs` | Uses a compact encoding for relative relocations (RELR), reducing binary size and startup time on modern kernels/glibc. |
| `-Wl,-plugin-opt=O3` | Instructs the LTO plugin (LLVM) to optimize at O3 during the link step. This is LTO-time code-generation optimization, separate from per-TU compile-time `-O3`. |
| `-Wl,-plugin-opt=mcpu=native` | Tells LLVM's LTO backend to target the native CPU when generating machine code at link time — mirrors `-march=native` but for the LTO pass. |
| `-flto` | Enables Link-Time Optimization. Passes IR (LLVM bitcode) between translation units, allowing the linker to inline and optimize across object file boundaries. |

---

### `CFLAGS` — C compiler flags

```
-march=native -O3 -pipe -fno-plt -fexceptions -Wp,-D_FORTIFY_SOURCE=3 -Wformat -Werror=format-security -fstack-clash-protection -fstack-protector-strong -fcf-protection -flto=full
```

| Flag | Effect |
|---|---|
| `-march=native` | Generate code using all instruction set extensions available on the current CPU (SSE4, AVX2, BMI2, etc. on x86_64). Resulting binaries are not portable. |
| `-O3` | Maximum compiler optimization level. Enables auto-vectorization, aggressive inlining, loop unrolling, and more. |
| `-pipe` | Uses pipes between compilation stages instead of temporary files. Speeds up compilation on systems with slow I/O. |
| `-fno-plt` | Calls shared library functions directly through the GOT instead of going through the PLT stub, saving one indirect branch per cross-DSO call. Also a hardening measure since it reduces the attack surface for PLT-reuse exploits. |
| `-fexceptions` | Enables C++ style stack unwinding tables even in C code. Required for correct behavior when C code is called from C++ with exceptions, or to allow DWARF-based profilers/backtraces to unwind through C frames. |
| `-Wp,-D_FORTIFY_SOURCE=3` | Enables glibc's buffer-overflow detection wrappers (e.g. for `memcpy`, `sprintf`) at level 3 — the most aggressive level, which also checks some dynamic-size buffers that level 2 misses. Requires optimization (`-O1` or higher) to take effect. |
| `-Wformat` | Warns about mismatches between `printf`/`scanf` format strings and their arguments. |
| `-Werror=format-security` | Promotes format-string security warnings (e.g. `printf(user_str)` with no format argument) to hard errors, preventing a class of format-string exploits. |
| `-fstack-clash-protection` | Inserts probe code to touch each page as the stack grows, preventing "stack clash" attacks where a large allocation silently skips past the guard page into another memory region. |
| `-fstack-protector-strong` | Adds a stack canary to functions that have local buffers or take the address of a local variable, detecting stack smashing at runtime. `-strong` is the most complete variant short of `-all`. |
| `-fcf-protection` | Emits Intel CET (Control-flow Enforcement Technology) `ENDBR` instructions and shadow-stack metadata. On supported x86_64 CPUs/kernels this enforces that indirect calls and returns can only target valid code. |
| `-flto=full` | Enables LLVM full LTO for this translation unit — all C object files are emitted as LLVM bitcode, merged, and optimized together at link time. Pairs with the `-plugin-opt=O3` in `LDFLAGS`. |

---

### `CXXFLAGS` — C++ compiler flags

```
<all of CFLAGS> -D_GLIBCXX_ASSERTIONS
```

`CXXFLAGS` inherits every flag from `CFLAGS` (see above) and adds:

| Flag | Effect |
|---|---|
| `-D_GLIBCXX_ASSERTIONS` | Enables lightweight bounds-checking assertions in the C++ standard library (libstdc++). For example, `std::vector::operator[]` will abort on out-of-bounds access in debug builds. Has a small runtime cost but catches UB that would otherwise be silent. |

---

### `RUSTFLAGS` — Rust compiler flags

```
-C target-cpu=native -C opt-level=3 -Zmir-opt-level=4 -C codegen-units=1 -Z unstable-options -C panic=immediate-abort
```

Plus, when linked through the `crosslto` profile:

```
-C linker=clang -C linker-plugin-lto
```

And all `LDFLAGS` are forwarded as `-Clink-arg=<flag>` entries.

| Flag | Effect |
|---|---|
| `-C target-cpu=native` | Tells `rustc`/LLVM to emit code targeting all features of the current CPU — the Rust equivalent of `-march=native`. |
| `-C opt-level=3` | Sets LLVM optimization level to O3 for Rust code. Enables the same aggressive optimizations as `-O3` in C/C++. |
| `-Zmir-opt-level=4` | Runs rustc's own Mid-level IR (MIR) optimizer at its most aggressive level (0–4) before handing off to LLVM. Performs inlining, copy propagation, and simplification passes that help LLVM do more. Requires `RUSTC_BOOTSTRAP=1` (nightly-gated). |
| `-C codegen-units=1` | Compiles the entire crate as a single LLVM module instead of splitting it across parallel codegen units. Allows LLVM to optimize and inline across the entire crate at compile time (even without LTO). Slower to compile but produces better code. |
| `-Z unstable-options` | Opt-in flag required to unlock certain unstable `rustc` options used elsewhere in the flags (e.g. some `-C` and `-Z` combinations). Requires `RUSTC_BOOTSTRAP=1`. |
| `-C panic=immediate-abort` | Replaces Rust's panicking machinery (stack unwinding, `PanicInfo` formatting, etc.) with a direct `abort()` call. Eliminates the panic runtime, significantly reducing binary size and removing unwinding overhead. |
| `-C linker=clang` *(crosslto only)* | Uses `clang` as the linker driver instead of `cc`/`gcc`. Required for cross-language LTO because clang knows how to hand LLVM bitcode from both Rust and C/C++ object files to the LTO backend together. |
| `-C linker-plugin-lto` *(crosslto only)* | Tells `rustc` to emit LLVM bitcode rather than native object files, and to invoke the LLVM LTO plugin at link time. This is what enables cross-language LTO between Rust and C/C++ — both sides must emit bitcode for the linker to merge and optimize them jointly. |
| `-Clink-arg=<LDFLAGS>` | Each linker flag from `LDFLAGS` is forwarded verbatim to the linker via `rustc`'s `-Clink-arg=` mechanism, since `rustc` drives the linker itself and doesn't read `LDFLAGS` from the environment directly. |

---

### Profile summary

| Profile | CFLAGS/CXXFLAGS set? | Extra RUSTFLAGS | When used |
|---|---|---|---|
| `crosslto` | ✅ Full flags | `-C linker=clang -C linker-plugin-lto` | Crate has C/C++ deps that benefit from cross-LTO |
| `rust` | ❌ Not set | — | Pure-Rust crate, no FFI benefit detected |
| `noc` *(probe only)* | ❌ Not set | `-C linker=clang -C linker-plugin-lto` | Canary build used during FFI detection probing |

---

## License

[MIT License](https://github.com/elseawhy/krabby/blob/main/LICENSE)
