bbcp
====

Securely and quickly copy data from source to target, using parallel TCP streams.

### About this fork

This repository is a fork of [eeertekin/bbcp](https://github.com/eeertekin/bbcp). For many years that was the only well-maintained bbcp fork on GitHub.

It has now been updated to the current official upstream, [bbcp/bbcp](https://github.com/bbcp/bbcp) (version 17.12.00.00.0). That brings in all upstream work since 2014:
- new options `-A`, `-@`, `-~`, `-+`, `-^` and `--omit-*`
- `c32z` and `c32c` checksums
- automatic IPv4 fallback when the IPv6 stack is broken
- an aarch64 build target
- the full manual in `doc/`

This fork also carries patches that make bbcp build cleanly and work correctly on current Ubuntu releases:

| Patch | Why |
|---|---|
| Linux build target chosen with `uname -m` instead of `uname -i` | `uname -i` prints `unknown` on Debian/Ubuntu, which breaks `make` |
| `noexcept(false)` instead of `throw(std::exception)` | C++17, GCC's default, rejects dynamic exception specifications |
| Drop `-lnsl` from the Linux link flags | glibc 2.32+ no longer ships libnsl, and bbcp doesn't need it |
| Allow hostnames up to 253 characters (was 64) | Copies failed on hosts with long fully qualified names, such as cloud VMs |
| Fix copies to a bare relative target (`bbcp f dst`) | An upstream regression failed these with "not in a known file system" |

The fork also adds GitHub Actions CI. Every change is built and smoke-tested on x86_64 and arm64, and tagged versions are published as binary releases.

### Installation

**Prebuilt binaries** (Ubuntu 22.04 and newer, x86_64 or arm64) are on the [Releases](https://github.com/ServaboFidem/bbcp/releases) page:

```sh
VER=17.12.00.00.0-2                                   # see Releases for the latest
ARCH=$(uname -m | sed 's/aarch64/arm64/')
curl -LO https://github.com/ServaboFidem/bbcp/releases/download/v$VER/bbcp-$VER-linux-$ARCH.tar.gz
tar xzf bbcp-$VER-linux-$ARCH.tar.gz
sudo install -m 755 bbcp-$VER-linux-$ARCH/bbcp /usr/local/bin/bbcp
bbcp --version
```

Each archive has a matching `.sha256` file, which you can check with `sha256sum -c`.

bbcp must be installed on **both** ends of a transfer, in a directory on the remote user's non-interactive `PATH`. Alternatively, point to it with `-S` and `-T`.

**Build from source** (Debian/Ubuntu):

```sh
sudo apt-get update
sudo apt-get install build-essential libssl-dev zlib1g-dev git
git clone https://github.com/ServaboFidem/bbcp.git
cd bbcp/src
make -j"$(nproc)"
sudo install -m 755 ../bin/*/bbcp /usr/local/bin/bbcp
bbcp --version
```

The binary is written to `bin/<platform>/bbcp`, for example `bin/amd64_linux/bbcp`.

### Examples
```sh
    > bbcp -P 2 -w 2M -s 10 my.big.file eee@moon:/top-secret-files/
    bbcp: Creating /top-secret-files/my.big.file
    bbcp: At 140506 17:33:18 copy 20% complete; 89998.2 KB/s
    bbcp: At 140506 17:33:28 copy 41% complete; 89910.4 KB/s
    bbcp: At 140506 17:33:38 copy 61% complete; 89802.5 KB/s
    bbcp: At 140506 17:33:48 copy 80% complete; 88499.3 KB/s
    bbcp: At 140506 17:33:58 copy 96% complete; 84571.9 KB/s
```

### Usage

Output of `bbcp -h`. The full manual is in `doc/` (`bbcp.pdf`, `bbcp.htm`).

    Usage:   bbcp [Options] [Inspec] Outspec
    Options: [-a [dir]] [-A] [-b [+]bf] [-B bsz] [-c [lvl]] [-C cfn] [-D] [-d path]
             [-e] [-E csa] [-f] [-F] [-g] [-h] [-i idfn] [-I slfn] [-k] [-K]
             [-L opts[@logurl]] [-l logf] [-m mode] [-n] [-N nio] [-o] [-O] [-p]
             [-P sec] [-r] [-R [args]] [-q qos] [-s snum] [-S srcxeq] [-T trgxeq]
             [-t sec] [-v] [-V] [-u loc] [-U wsz] [-w [=]wsz] [-x rate] [-y] [-z]
             [-Z pnf[:pnl]] [-4 [loc]] [-~] [-@ {copy|follow|ignore}] [-$] [-#] [--]
    I/Ospec: [user@][host:]file
    Function: Secure and fast copy utility.
    -a dir  append mode to restart a previously failed copy.
    -A      automatically create destination directory if it does not exist.
    -b bf   sets the read blocking factor (default is 1).
    -b +bf  adds additional output buffers to mitigate ordering stalls.
    -B bsz  sets the read/write I/O buffer size (default is wsz).
    -c lvl  compress data before sending across the network (default lvl is 1).
    -C cfn  process the named configuration file at time of encounter.
    -d path requests relative source path addressing and target path creation.
    -D      turns on debugging.
    -e      error check data for transmission errors using md5 checksum.
    -E csa  specify checksum alorithm and optionally report or verify checksum.
            csa: [%]{a32|c32|md5|c32z|c32c}[=[<value> | <outfile>]]
    -f      forces the copy by first unlinking the target file before copying.
    -F      does not check to see if there is enough space on the target node.
    -h      print help information.
    -i idfn is the name of the ssh identify file for source and target.
    -I slfn is the name of the file that holds the list of files to be copied.
            With -I no source files need be specified on the command line.
    -k      keep the destination file even when the copy fails.
    -K      do not rm the file when -f specified, only truncate it.
    -l logf logs standard error to the specified file.
    -L args sets the logginng level and log message destination.
    -m mode target file mode as [dmode/][fmode] but one mode must be present.
            Default dmode is 0755 and fmode is 0644 or it comes via -p option.
    -n      do not use DNS to resolve IP addresses to host names.
    -N nio  enable named pipe processing; nio specifies input and output state:
            i -> input pipe or program, o -> output pipe or program
    -s snum number of network streams to use (default is 4).
    -o      enforces output ordering (writes in ascending offset order).
    -O      omits files that already exist at the target node (useful with -r).
    -p      preserve source mode, ownership, and dates.
    -P sec  produce a progress message every sec seconds (15 sec minimum).
    -q lvl  specifies the quality of service for routers that support QOS.
    -r      copy subdirectories and their contents (actual files only).
    -R args enables real-time copy where args specific handling options.
    -S cmd  command to start bbcp on the source node.
    -T cmd  command to start bbcp on the target node.
    -t sec  sets the time limit for the copy to complete.
    -v      verbose mode (provides per file transfer rate).
    -V      very verbose mode (excruciating detail).
    -u loc  use unbuffered I/O at source or target, if possible.
            loc: s | t | st
    -U wsz  unnegotiated window size (sets maximum and default for all parties).
    -w wsz  desired window size for transmission (the default is 128K).
            Prefixing wsz with '=' disables autotuning in favor of a fixed wsz.
    -x rate is the maximum transfer rate allowed in bytes, K, M, or G.
    -y what perform fsync before closing the output file when what is 'd'.
            When what is 'dd' then the file and directory are fsynced.
    -z      use reverse connection protocol (i.e., target to source).
    -Z      use port range pn1:pn2 for accepting data transfer connections.
    -+      for recursive copies only copy readable/searchable items.
    -4      use only IPV4 stack; optionally, at specified location.
    -~      preserve atime and mtime only.
    -@      specifies how symbolic links are handled: copy recreates the symlink,
            follow copies the symlink target, and ignore skips it (default).
    -$      print the license and exit.
    -#      print the version and exit.
    -^      omit copying empty directories (same as --omit-emptydirs).
    --      allows an option with a defaulted optional arg to appear last.
    user    the user under which the copy is to be performed. The default is
            to use the current login name.
    host    the location of the file. The default is to use the current host.
    Inspec  the name of the source file(s) (also see -I).
    Outspec the name of the target file or directory (required if >1 input file.)
    ******* Complete details at: http://www.slac.stanford.edu/~abh/bbcp

### Credits and license

bbcp was written by Andrew Hanushevsky at SLAC National Accelerator Laboratory, Stanford University.

Lineage of this repository:
- SLAC's original repository (http://www.slac.stanford.edu/~abh/bbcp/bbcp.git/)
- [eeertekin/bbcp](https://github.com/eeertekin/bbcp)
- merged with the official [bbcp/bbcp](https://github.com/bbcp/bbcp)
- this fork

bbcp is licensed under the GNU LGPL v3 or later; see `src/COPYING.LESSER`. `LICENSE` contains the GNU GPL v3 text that the LGPL incorporates.