# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

bbcp is a C++ point-to-point network file copy tool. It was written by Andrew Hanushevsky at SLAC/Stanford and is licensed under LGPL (see the source headers and `src/COPYING.LESSER`).

This repo is a fork that tracks the official upstream at https://github.com/bbcp/bbcp, currently version `17.12.00.00.0`. It carries a few local build changes, listed under "Fork-specific changes" below.

C++ sources use the `.C` extension. The only C file is `NetLogger.c`.

Other top-level directories:
- `doc/` holds the full manual (`bbcp.htm` / `bbcp.pdf` / `bbcp.docx`). Use it as the reference for option semantics.
- `utils/bbcp.spec` is an RPM spec.
- `patches/` holds an old macOS patch.

## Build

The build is Unix-only: it uses pthreads, fork/exec of ssh, and `/bin/uname`. On this Windows machine, build in WSL (Ubuntu 24.04 is installed). MSVC is not supported.

```sh
# deps (Debian/Ubuntu): build-essential libssl-dev zlib1g-dev
cd src
make -j"$(nproc)"     # -> ../bin/<sysname>/bbcp, objects in ../obj/<sysname>/
make clean
../bin/amd64_linux/bbcp -#   # prints version
```

The working tree is checked out with CRLF line endings, and `make` and `/bin/sh` reject CRLF. Build from a clean LF copy instead:

```sh
git archive HEAD | tar -x -C /tmp/bbcpbuild
```

How the Makefile dispatches:
- `make` runs `make $(uname) OSVER=$(../MakeSname) RELEASE=$(../MakeSname -r)`. `MakeSname` prints a platform name such as `amd64_linux`, or uses `$DEST_SYSNAME` if that is set. It also creates `../obj/<name>` and `../bin/<name>`.
- The `Linux` target re-dispatches to `makeLinux$(uname -m)`. The existing targets are `makeLinuxi386`/`makeLinuxi686` (one rule), `makeLinuxx86_64` and `makeLinuxaarch64`. Any other architecture needs its name added to one of these rules.
- Linux and Darwin link OpenSSL MD5 (`bbcp_MD5_openssl`) through `MD5HEADER`/`OBJMD5`.
- `make OPT=-D...` adds extra flags on targets that honor `$(OPT)` (FreeBSD, Darwin, Sun).

The Makefile is hand-written. To add a source file, update `SOURCE` and `OBJECT`, and add an explicit `$(OBJDIR)/foo.o:` rule with the header dependencies listed. There is no automatic dependency generation.

There is no test suite and no lint configuration. To check a change, build it and run a real copy. When `bbcp` is not on PATH, use `-S`/`-T` to point the spawned source and sink at the built binary:

```sh
B=/tmp/bbcpbuild/bin/amd64_linux/bbcp
head -c 50000000 /dev/urandom > /tmp/src.bin
$B -v -s 4 -E c32c -S "$B -S" -T "$B -T" /tmp/src.bin /tmp/dst.bin && cmp /tmp/src.bin /tmp/dst.bin
```

The expected warnings are `-Wregister` in `bbcp_MD5_openssl.C` and OpenSSL 3 MD5 deprecations.

## Releases

The workflow is `.github/workflows/linux-release.yml`:
- It runs on every push or PR to `origin`. It builds and smoke-tests on `ubuntu-22.04` (x86_64) and `ubuntu-22.04-arm` (arm64), and uploads `bbcp-<ver>-linux-<arch>.tar.gz` plus a `.sha256` file as artifacts.
- Pushing a `v*` tag (for example `v17.12.00.00.0`) also publishes a GitHub Release with those archives.
- It builds on 22.04 because the resulting binaries need glibc 2.35+ and libcrypto.so.3; building on 22.04 keeps them working on Ubuntu 22.04 and newer.
- `DEST_SYSNAME=linux-<arch>` is set because `MakeSname` labels arm64 as `i386_linux`.

## Fork-specific changes (keep when merging upstream)

- **`uname -m`, not `uname -i`, in the `Linux` target.** Upstream uses `uname -i`, which prints `unknown` on Debian/Ubuntu.
- **No `-lnsl` in `LNXLIBS*`.** glibc 2.32+ no longer ships it, and bbcp uses no NIS/RPC symbols.
- **`~bbcp_Semaphore() noexcept(false)` in `bbcp_Pthread.h`.** Upstream's `throw(std::exception)` does not compile under C++17, which is GCC's default.

Remotes:
- `origin` is the fork (ServaboFidem/bbcp). All pushes go here.
- `upstream` is https://github.com/bbcp/bbcp. It is fetch-only; its push URL is deliberately invalid.

To pull new upstream work:

```sh
git fetch upstream
git merge upstream/master
```

On GitHub, this fork is still listed as a fork of the dead `eeertekin/bbcp`. Do not open PRs against that repo.

## Architecture

**One binary, three roles.** The user runs `bbcp` as the controller. The controller then uses ssh (`bbcp_Node::Run`) to start a copy of `bbcp` on the source host and another on the sink host. Those copies get hidden source/sink options (`bbcp_SRC`/`bbcp_SNK` in `bbcp_Config.Options`). In `main()` in `bbcp.C`, those options route a process to either `bbcp_Protocol::Process` (source side) or `bbcp_Protocol::Request` (sink side). The controller path groups source file specs by user@host and calls `bbcp_Protocol::Schedule` once per group.

**Control flow (`bbcp_Protocol`).** `Schedule` works in this order:
1. Start the first node and send it its arguments (`SendArgs`).
2. Read back the callback port (`getCBPort`).
3. Start the second node and give it the callback host and port so it connects to the first node.
4. Wait for both nodes.

`bbcp_Config.Options & bbcp_CON2SRC` sets which side listens, which matters for firewalls. After the connection is made, the source and sink exchange a small text protocol (login, flist, get, exit) through the `Process_*` and `Request_*` methods. If the source and sink bbcp versions differ, both ends must still speak this protocol; a protocol change only works once the new bbcp is installed on both hosts.

**Data path (`bbcp_Node::SendFile`/`RecvFile`).** The data moves through a threaded pipeline over N parallel TCP streams (`-s`):

```
file -> inFile -> CPool -> [ZCX compress stages] -> BPool -> data_link[i] -> net
```

- `bbcp_Link` wraps one data stream.
- `bbcp_BuffPool` instances are global queues of buffers (`bbcp_APool`, `bbcp_BPool`, `bbcp_CPool` in `bbcp_BuffPool.C`).
- `bbcp_ZCX` does zlib compression (`-c`).
- Threads are created through `bbcp_Pthread`.
- `bbcp_ProgMon` reports progress and `bbcp_ProcMon` watches the parent process.
- The window size and buffer size (`-w`, `-B`) are negotiated in `bbcp_Protocol::AdjustWS`.

**Checksums (`-e`/`-E csa`).** All checksum classes derive from `bbcp_ChkSum`:

| `-E` name | Class |
|---|---|
| `a32` | `bbcp_A32` / `bbcp_A32_zlib` (adler32) |
| `c32` | `bbcp_C32` (cksum-style CRC32) |
| `c32z` | `bbcp_C32_Zip` (zip-compatible CRC32) |
| `c32c` | `bbcp_C32C` (CRC32c) |
| `md5` | `bbcp_MD5` / `bbcp_MD5_openssl` |

Adding a new checksum touches three places:
- `bbcp_Config.C` (around `!strcmp("c32c", csName)`) maps the `-E` name to a `bbcp_cs*` enum value and size.
- The switch in `bbcp_ChkSum.C` creates the object for that enum value.
- The `SOURCE`, `OBJECT` and rule lists in the Makefile.

**Pluggable I/O and filesystem layers.**
- `bbcp_FileSystem` has three implementations: `FS_Unix` (normal files), `FS_Pipe` (named pipes and programs, `-N`), and `FS_Null` (`/dev/null`-style).
- `bbcp_IO` has matching variants: `IO`, `IO_Pipe`, and `IO_Null`.
- `bbcp_File` puts an `IO` together with buffering and ordering logic (`-o`).
- `bbcp_FileSpec` parses `[user@][host:]path` specs and expands directories for `-r`. Recursive copies honour the symlink mode (`-@`), readable-only (`-+`) and the `--omit-*` filters. `bbcp_Set` is a small set helper used during expansion.

**Options.** Short options are declared in `bbcp_VALIDOPT*` in `bbcp_Config.C`. Long options are mapped to short letters through `Args.Option(...)` calls in the same file, and those must be listed in increasing minimum-length order. The help text is the `H(...)`/`I(...)` block in `bbcp_Config::help()`. The README "Usage" section is copied from it, so update both together.

**Global singletons.** `bbcp_Config` (all parsed options, via `bbcp_Args`), `bbcp_OS` (`bbcp_System`), `bbcp_Net` (`bbcp_Network`), and `bbcp_Debug` are global objects declared `extern` across files. Most code reads options straight from `bbcp_Config` instead of taking them as parameters.

**Other pieces.**
- `bbcp_RTCopy` handles real-time copy of files that are still growing (`-R`).
- `bbcp_NetLogger` and `NetLogger.c` provide optional NetLogger instrumentation.
- `bbcp_Emsg` handles error messages.
- The version string is hard-coded in `bbcp_Version.C`.
- Platform differences are handled with `#ifdef LINUX` / `MACOS` / `SUN` / `FREEBSD` / `AIX`. These macros come from the Makefile targets, and `bbcp_Platform.h` handles some of them.
