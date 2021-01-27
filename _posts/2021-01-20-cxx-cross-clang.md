---
layout: post
title: Cross compiling made easy, using Clang and LLVM
---

Anyone who ever tried to cross-compile a C/C++ program knows how big a PITA the whole process could be. 
I've been building software for other platforms on my main GNU/Linux system for years, and I must say it has never been a very pleasant experience.
The main culprits are how byzantine build systems tend to be when configuring for cross-compilation, and how messy it is to set-up your cross toolchain in the first place.   

One of the main culprits in my experience has been the GNU toolchain, the decades-old behemoth upon which the Unix-like world has been building for years.
Like many compilers of yore, GCC and its `binutils` brethren were never designed with the intent to support multiple targets within a single setup. The only supported approach is to install a full cross build for each triple you wish to target on any given host. 

For instance, assuming you wish to build something for FreeBSD on your Linux machine using GCC, you need:

- A GCC + binutils install for your host triplet (i.e., x86_64-pc-linux-gnu or similar);
- A GCC + binutils complete install for your target triplet (i.e. x86_64-unknown-freebsd12.2-gcc, as, nm, etc)
- A sysroot containing the necessary libraries and headers, which you can promptly steal from a running installation of FreeBSD.

This process is sometimes made simpler by Linux distributions or hardware vendors offering a selection of prepackaged compilers, but this is often not enough due to the sheer amount of possible host-target combinations. This sometimes means build the whole toolchain yourself, something that, unless you rock a quite beefy CPU, tends to be a massive waste of time and power.

### Clang as a cross compiler

This annoying limitation is one of the reasons why I got interested in LLVM (and thus Clang), which is by-design a full-fledged cross compiler toolchain widely compatible with GNU. A single install can output and compile code _for every supported target_, as long as a complete sysroot is available at build time.

I found this to be a game-changer, and, while it can't still compete with modern language toolchains such as Go, it's night and day better than what we had before. You can now just fetch whatever your favorite package management system has available (as long as it's not extremely old), avoiding messing around with multiple installs of GCC.

Until a few years ago, the whole process wasn't as smooth as it could be. Due to LLVM not having a full toolchain yet available, you were still supposed to provide a `binutils` build specific for your target. While this was much more tolerable (`binutils` is relatively fast to build), it was still somewhat of a nuisance, and I'm glad that `llvm-mc` and `lld` are finally stable as flexible as the rest of LLVM.

With the toolchain now set, the next step becomes to obtain a sysroot in order to provide the needed headers and libraries to compile and link for your target.

### Obtaining a sysroot 
A super fast way to find a working system directory for a given OS is to rip it straight from an existing system. It could be from a Docker container image, or straight from a running machine.
For instance, this is how I used `tar` through `ssh` as a quick way to extract a working sysroot from a FreeBSD 13-CURRENT AArch64 VM [1^]:

```
$ mkdir ~/farm_tree
$ ssh FARM64 'tar cf - /lib /usr/include /usr/lib /usr/local/lib /usr/local/include' | bsdtar xvf - -C $HOME/farm_tree/
```

### Invoking the cross compiler

With everything set, it's now only a matter of invoking Clang with the right arguments:

```shell
$  clang++ --target=aarch64-pc-freebsd --sysroot=$HOME/Workspace/farm_tree -fuse-ld=lld -stdlib=libc++ -o zpipe zpipe.cc -lz --verbose
clang version 11.0.1
Target: aarch64-pc-freebsd
Thread model: posix
InstalledDir: /usr/bin
 "/usr/bin/clang-11" -cc1 -triple aarch64-pc-freebsd -emit-obj -mrelax-all -disable-free -disable-llvm-verifier -discard-value-names -main-file-name zpipe.cc -mrelocation-model static -mframe-pointer=non-leaf -fno-rounding-math -mconstructor-aliases -munwind-tables -fno-use-init-array -target-cpu generic -target-feature +neon -target-abi aapcs -fallow-half-arguments-and-returns -fno-split-dwarf-inlining -debugger-tuning=gdb -v -resource-dir /usr/lib/clang/11.0.1 -isysroot /home/marco/farm_tree -internal-isystem /home/marco/farm_tree/usr/include/c++/v1 -fdeprecated-macro -fdebug-compilation-dir /home/marco/dummies/cxx -ferror-limit 19 -fno-signed-char -fgnuc-version=4.2.1 -fcxx-exceptions -fexceptions -faddrsig -o /tmp/zpipe-54f1b1.o -x c++ zpipe.cc
clang -cc1 version 11.0.1 based upon LLVM 11.0.1 default target x86_64-pc-linux-gnu
#include "..." search starts here:
#include <...> search starts here:
 /home/marco/farm_tree/usr/include/c++/v1
 /usr/lib/clang/11.0.1/include
 /home/marco/farm_tree/usr/include
End of search list.
 "/usr/bin/ld.lld" --sysroot=/home/marco/farm_tree --eh-frame-hdr -dynamic-linker /libexec/ld-elf.so.1 --enable-new-dtags -o zpipe /home/marco/farm_tree/usr/lib/crt1.o /home/marco/farm_tree/usr/lib/crti.o /home/marco/farm_tree/usr/lib/crtbegin.o -L/home/marco/farm_tree/usr/lib /tmp/zpipe-54f1b1.o -lz -lc++ -lm -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /home/marco/farm_tree/usr/lib/crtend.o /home/marco/farm_tree/usr/lib/crtn.o
$ file zpipe
zpipe: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped
```

In the snipped above, I have managed to compile and link a C++ file into an executable for AArch64 FreeBSD, all while using just the `clang` and `lld` I had already installed on the system.

More in detail:
1. `--target` swaps the LLVM default target (`x86_64-pc-linux-gnu`) to `aarch64-pc-freebsd`, thus enabling cross-compilation.
2. `--sysroot` forces Clang to assume the specified path as the root to search for headers and libraries, instead of the standard paths. Note that sometimes this setting might not be enough, especially if the target uses GCC and Clang somehow fails to detect its install path. This can be easily fixed by specifying `--gcc-toolchain`, which clarifies where to search for GCC installations.
3. `-fuse-ld=lld` tells Clang to use `lld` instead of the default. As I will explain in a short while, it's highly unlikely that the system linker understands foreign targets, while LLD can natively support almost every binary format and OS [^3].
4. `-stdlib=libc++` is needed here due to Clang failing to detect that FreeBSD on AArch64 uses LLVM's `libc++` instead of GCC's `libstdc++`.
5. `-lz` is also specified to show how Clang can also resolve other libraries inside the sysroot without issues, in this case, `zlib`.

The last test is now to copy the binary to our target system and check if it works correctly:

```shell
$ rsync zpipe FARM64:"~"
$ ssh FARM64
FreeBSD-ARM64-VM $ chmod +x zpipe
FreeBSD-ARM64-VM $ ldd zpipe
zpipe:
        libz.so.6 => /lib/libz.so.6 (0x4029e000)
        libc++.so.1 => /usr/lib/libc++.so.1 (0x402e4000)
        libcxxrt.so.1 => /lib/libcxxrt.so.1 (0x403da000)
        libm.so.5 => /lib/libm.so.5 (0x40426000)
        libc.so.7 => /lib/libc.so.7 (0x40491000)
        libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x408aa000)
FreeBSD-ARM64-VM $ ./zpipe -h
zpipe usage: zpipe [-d] < source > dest
```

Success! It's now possible to use this cross toolchain to build larger programs, and below I'll quickly explain how to use it to build packages relying on Autotools or CMake. 

#### Optional: creating an LLVM toolchain directory

LLVM provides a mostly compatible counterpart for almost every tool shipped by `binutils` (with the notable exception of `as` [^1]), prefixed with `llvm-`. 

The most critical of these is LLD, which is a drop in replacement for a plaform's system linker, capable to replace both GNU `ld.bfd` and `gold` on GNU/Linux or BSD, and Microsoft's LINK.EXE when targeting MSVC. It supports linking on (almost) every platform supported by LLVM, thus removing the nuisance to have multiple specific linkers installed.

Both GCC and Clang support using `ld.lld` instead of the system linker (which may well be `lld`, like on FreeBSD) via the command line switch `-fuse-ld=lld`. 

In my experience, I found that Clang's driver might get confused when picking the right linker on some uncommon platforms, especially before version 11.0.
For some reason, `clang` sometimes decided to outright ignore the `-fuse-ld=lld` switch and picked the system linker (`ld.bfd` in my case), which does not support AArch64.

A fast solution to this is to create a toolchain directory containing symlinks that rename the LLVM utilities to the standard `binutils` programs:  

```shell
$  ls -la ~/.llvm/bin/
Permissions Size User  Group Date Modified Name
lrwxrwxrwx    16 marco marco  3 Aug  2020  ar -> /usr/bin/llvm-ar
lrwxrwxrwx    12 marco marco  6 Aug  2020  ld -> /usr/bin/lld
lrwxrwxrwx    21 marco marco  3 Aug  2020  objcopy -> /usr/bin/llvm-objcopy
lrwxrwxrwx    21 marco marco  3 Aug  2020  objdump -> /usr/bin/llvm-objdump
lrwxrwxrwx    20 marco marco  3 Aug  2020  ranlib -> /usr/bin/llvm-ranlib
lrwxrwxrwx    21 marco marco  3 Aug  2020  strings -> /usr/bin/llvm-strings
```

The `-B` switch can then be used to force Clang (or GCC) to search the required tools in this directory, stopping the issue from ever occurring:

```shell
$  clang++ -B$HOME/.llvm/bin -stdlib=libc++ --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree -std=c++17 -o mvd-farm64 mvd.cc
$ file mvd-farm64
mvd-farm64: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 13.0, FreeBSD-style, with debug_info, not stripped
```

#### Optional: creating Clang wrappers to simplify cross-compilation

I happened to notice that certain build systems (and with _"certain"_ I mean some poorly written `Makefile`s and sometimes Autotools) have a tendency to misbehave when the specified CC, CXX or LD commands containing spaces or multiple parameters, like the one we devised above. [^4] 

Given how unwieldy it is to remember to write all of the correct parameters everywhere, I like to write quick wrappers for `clang` and `clang++` in order to simplify building for a certain target:

```shell
$ cat ~/.local/bin/aarch64-pc-freebsd-clang
#!/usr/bin/env sh

exec /usr/bin/clang -B$HOME/.llvm/bin --target=aarch64-pc-freebsd --sysroot=$HOME/Workspace/farm_tree "$@"
$ cat ~/.local/bin/aarch64-pc-freebsd-clang++
#!/usr/bin/env sh

exec /usr/bin/clang++ -B$HOME/.llvm/bin -stdlib=libc++ --target=aarch64-pc-freebsd --sysroot=$HOME/Workspace/farm_tree "$@"
$ aarch64-pc-freebsd-clang++ -o tst tst.cc -static
 marco  Canis  ~  $  file tst
tst: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), statically linked, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped

```

[^1]: If the transfer is happening across a network and not locally, it's a good idea to compress the output tarball.
[^2]: `llvm-mc` can be used as an assembler, but it's cumbersome to use and not well documented. Like `gcc`, the `clang` frontend can act as an assembler, making `as` often redundant.   
[^3]: Sadly, macOS is not supported anymore by LLD due to Mach-O support being largely unmaintained and left to rot over the last years. This leaves `ld64` (or a cross-build thereof, if you manage to build it) as the only way to link Mach-O executables (unless `ld.bfd` from `binutils` still supports it).
[^4]: This is without talking about those criminals who hardcode `gcc` in their build scripts, but this is a rant better left for another day.