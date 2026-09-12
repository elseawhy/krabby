# 🦀 krabby

An aggressive, performance-obsessed wrapper around `cargo` that auto-tunes build profiles per binary. It probes whether C/C++ FFI dependencies actually benefit from cross-language LTO, caches the winning profile, and reuses it on future builds — so you don't pay probing costs more than once per crate.

Built for machines where you're willing to trade portability and compile time for the fastest binary the compiler can produce on *your* CPU.

## What it does

`krabby` is a single Bash script (`_krabby_main`) that replaces day-to-day `cargo build` / `cargo install` / `cargo update` usage with:

- **Native everything**: `-march=native`, `-C target-cpu=native`, full LTO, `lld` as the linker, and hardening flags (`FORTIFY_SOURCE=3`, stack protector, etc.) baked into every build.
- **Clang LTO by default**: Instead of letting rustc blindly link standard objects, it forces `linker-plugin-lto` for all builds, handing LLVM bitcode directly to Clang's `-O3` pipeline. This enables aggressive global vectorization and natively inlines C/C++ FFI boundaries, achieving measurable performance gains (up to 6%) even on pure Rust crates.
- **Crate update checking** (`krabby update`): checks `crates.io` and Git repositories for newer versions of every `cargo install`-ed binary and recompiles anything out of date. Supports holding packages back via `krabby hold`.

## Requirements

`krabby` targets **Linux** systems. The host target triple is now auto-detected via `rustc -vV` and `clang -print-target-triple`, making the core build pipeline completely portable across architectures (x86_64, AArch64, RISC-V, etc.).

### Required

| Tool | Why |
|---|---|
| `bash` (4+) | The script itself (`#!/usr/bin/env bash`) |
| `rustc` + `cargo` | Core build tooling. A **stable** toolchain is sufficient — the script uses `-Z build-std`, `-Z unstable-options`, and `-Zmir-opt-level`, gated via `RUSTC_BOOTSTRAP=1`. **On Arch Linux** (or any distro shipping the latest stable): `sudo pacman -S rust` (or your package manager's equivalent) is recommended. **On all other distros**: use [rustup](https://rustup.rs) — `apt`/`dnf` often lag months behind upstream and may ship a version too old for some flags. |
| `rust-src` | Required for `-Z build-std=std,panic_abort`. On Arch: `sudo pacman -S rust-src`. With rustup: `rustup component add rust-src` |
| Host target triple | Auto-detected at runtime via `rustc -vV` and `clang -print-target-triple` — no manual configuration needed. Bundled with the `rust` package on Arch; available by default with rustup. |
| `clang` / `clang++` | Used as `CC`/`CXX` for the `crosslto` profile |
| `llvm-ar`, `llvm-ranlib` | Used as `AR`/`RANLIB` for the `crosslto` profile |
| `lld` | Linker, invoked via `-fuse-ld=lld` in `LDFLAGS` |
| `curl` | Fetches latest crate versions from crates.io in `krabby update` |
| `git` | Used to check remote commit hashes for git-installed crates in `krabby update` |
| `awk` | Version/index parsing in `krabby update` and binary path resolution |
| coreutils (`cp`, `rm`, `mktemp`) | General file ops and temp log management |

### Optional

- A CPU that actually benefits from `-march=native` (e.g., AVX2/BMI2 on x86_64, or NEON/SVE on ARM). The current probing logic assumes recent x86_64 hardware; on older CPUs or other architectures, the `crosslto` vs `rust` comparison may just always come back even or fallback to `rust`.

### Installation

```bash
# Install Rust toolchain + build dependencies (Arch or other rolling-release distros)
sudo pacman -S rust rust-src clang llvm lld curl

# Install the script
curl -o ~/.local/bin/krabby https://raw.githubusercontent.com/elseawhy/krabby/refs/heads/main/krabby
chmod +x ~/.local/bin/krabby
```

> **Users on distros without the latest stable Rust** — use [rustup](https://rustup.rs) instead of your distro's Rust package. `apt`/`dnf` often ship outdated stable versions. After installing rustup, add `rust-src` with `rustup component add rust-src`, then install the remaining native dependencies via your package manager:
> ```bash
> # Debian/Ubuntu
> sudo apt install clang llvm-dev lld curl
> ```

> **`rustup` vs `RUSTC_BOOTSTRAP=1`** — these solve different problems. `rustup` is recommended to ensure you have a *recent* stable `rustc` (distro packages can lag months behind). `RUSTC_BOOTSTRAP=1` is a separate mechanism that coerces any stable `rustc` into accepting nightly-gated flags (`-Z build-std`, `-Zmir-opt-level`, etc.) — no nightly toolchain or `rustup override` is needed for that.

## Usage

- `krabby [args...]` - Build the local project (auto-detects Cargo.toml). Arguments are passed directly to `cargo build`. Note: `--release` and `--locked` are automatically enabled.
- `krabby install <crate_or_url>... [cargo_flags...]` - Install/compile crates from crates.io or Git repositories (auto-detected via `http(s)://`). Supports multi-binary installs and applies cargo flags (like `--features`) safely across all crates sequentially. Note: `--locked` is automatically enabled.
- `krabby inject <cmd>` - Execute an arbitrary command with Krabby's aggressive compiler environment variables (`CFLAGS`, `RUSTFLAGS`, etc.) injected.
- `krabby update` - Check crates.io and Git repositories and upgrade all installed binaries
- `krabby list` - List all cargo-installed binaries
- `krabby hold <crate>...` - Prevent one or more packages from being updated
- `krabby unhold <crate>...` - Allow a package to be updated again
- `krabby help` - Show usage


## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `XDG_CONFIG_HOME` | `~/.config` | Where the package hold list (`krabby/hold`) is stored |

## ⚠️ Caveats

- **Not portable**: binaries built with `-march=native` will only run correctly on the same (or a very similar) CPU. Don't ship these artifacts elsewhere.
- **Nightly-gated flags via `RUSTC_BOOTSTRAP=1`**: this bypasses the stable/nightly gate, so a plain stable `rustc` (from `pacman`, `apt`, etc.) is sufficient — no `rustup` or nightly toolchain needed. That said, it is inherently fragile across `rustc` versions — expect occasional breakage when Rust changes internals of `-Z build-std` or other unstable flags.

## Build Flags Reference

This section documents every flag set by krabby. All flags are applied through environment variables passed to `cargo` during compilation.

---

### `LDFLAGS` — Linker flags (passed to the C/C++ linker)

```
-march=native -fuse-ld=lld -Wl,-O1,--sort-common,--as-needed,-z,relro,-z,now,-z,pack-relative-relocs,-plugin-opt=O3,-plugin-opt=mcpu=native -flto
```

| Flag | Effect |
|---|---|
| `-march=native` | Emits CPU instructions tuned for the exact CPU this machine has — enables AVX2, BMI2, etc. on x86_64, and equivalent extensions like NEON/SVE on ARM. Passed to the linker so that LTO-time code generation uses the same target arch as compilation. |
| `-fuse-ld=lld` | Selects `lld` as the linker. lld is the LLVM linker and is generally faster than `ld` while offering excellent compatibility and stability. |
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
--target=$LLVM_TARGET -ffat-lto-objects -march=native -O3 -pipe -fno-plt -fexceptions -Wp,-D_FORTIFY_SOURCE=3 -Wformat -Werror=format-security -fstack-clash-protection -fstack-protector-strong -fcf-protection -flto=full
```

| Flag | Effect |
|---|---|
| `--target=$LLVM_TARGET` | Explicitly targets the LLVM triple (auto-detected via `clang -print-target-triple`) to prevent architecture string mismatches during C/C++ compilation. |
| `-ffat-lto-objects` | Emits both LLVM bitcode and native machine code into object files. Fixes compatibility issues with `cc` builds that invoke tools expecting standard ELF objects before the final link step. |
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
-C target-cpu=native -C opt-level=3 -Zmir-opt-level=4 -C codegen-units=1 -Z unstable-options -C panic=immediate-abort -C linker=clang -C linker-plugin-lto
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
| `-C linker=clang` | Uses `clang` as the linker driver instead of `cc`/`gcc`. Required for cross-language LTO because clang knows how to hand LLVM bitcode from both Rust and C/C++ object files to the LTO backend together. |
| `-C linker-plugin-lto` | Tells `rustc` to emit LLVM bitcode rather than native object files, and to invoke the LLVM LTO plugin at link time. This is what enables cross-language LTO between Rust and C/C++ — both sides must emit bitcode for the linker to merge and optimize them jointly. |
| `-Clink-arg=<LDFLAGS>` | Each linker flag from `LDFLAGS` is forwarded verbatim to the linker via `rustc`'s `-Clink-arg=` mechanism, since `rustc` drives the linker itself and doesn't read `LDFLAGS` from the environment directly. |

---

## License

[MIT License](https://github.com/elseawhy/krabby/blob/main/LICENSE)
