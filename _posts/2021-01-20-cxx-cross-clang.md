---
layout: post
title: Cross compiling made easy, using Clang and LLVM
---

Anyone who ever tried to cross-compile a C/C++ program knows how big a PITA the whole process could be. 
I have been building software for other platforms on my main GNU/Linux system for years, and I must say it has never been a very pleasant experience.
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
For instance, this is how I used `tar` through `ssh` as a quick way to extract a working sysroot from a FreeBSD 13-CURRENT AArch64 VM [^1]:

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
3. `-fuse-ld=lld` tells Clang to use `lld` instead of the default. As I will explain in a short while, it's highly unlikely that the system linker understands foreign targets, while LLD can natively support almost every binary format and OS [^2].
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

LLVM provides a mostly compatible counterpart for almost every tool shipped by `binutils` (with the notable exception of `as` [^3]), prefixed with `llvm-`. 

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

exec /usr/bin/clang -B$HOME/.llvm/bin --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree "$@"
$ cat ~/.local/bin/aarch64-pc-freebsd-clang++
#!/usr/bin/env sh

exec /usr/bin/clang++ -B$HOME/.llvm/bin -stdlib=libc++ --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree "$@"
$ aarch64-pc-freebsd-clang++ -o tst tst.cc -static
$ file tst
tst: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), statically linked, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped

```

### Appendix: static linking with musl and Alpine Linux

Statically linking a C or C++ program can sometimes save you a lot of library compatibility headaches, especially when you can't control what's going to be installed on whatever you plan to target.
Building static binaries is however quite annoying on GNU/Linux, due to Glibc actively refraining people from linking it statically. [^5]

Musl is a very compatible standard library implementation for Linux that plays much nicer with static linking, and it is now shipped by most major distributions. These packages often suffice in building your code statically, at least as long as you plan to stick with plain C. 

The situation gets much more complicated if you plan to use C++, or if you need additional components. System libraries (like `libstdc++`, `libz`, `libffi` and so on) are usually only built for Glibc, so you must compile everything else yourself, including `libstdc++`, which means recompiling GCC or LLVM's `libc++`.

Thankfully, there are several distributions out there that target _"Musl-plus-Linux"_, everyone's favorite being Alpine Linux. It is thus possible to apply the same approach used with FreeBSD on AArch before to obtain a `x86_64-pc-linux-musl` sysroot, complete of libraries and packages built for Musl. 

#### Setting up an Alpine container

A good starting point are the minirootfs tarballs provided by Alpine, which are meant for containers and tend to be very small:

```shell
$ wget -qO - https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.1-x86_64.tar.gz | gunzip | sudo tar xfp - -C ~/alpine_tree
```

It is now possible to chroot inside the image in `~/alpine_tree` to set it up with all the packages you may need. 
I prefer in general to use `systemd-nspawn` instead then `chroot` because it is vastly better and less error prone. [^6]

```shell
$ $  sudo systemd-nspawn -D alpine_tree
Spawning container alpinetree on /home/marco/alpine_tree.
Press ^] three times within 1s to kill container.
alpinetree:~# 
```

We can now (optionally) switch to the `edge` branch of Alpine for newer packages by editing `/etc/apk/repositories`, and then install the required packages containing the static libraries we need:

```shell
alpinetree:~# cat /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/edge/main
https://dl-cdn.alpinelinux.org/alpine/edge/community
alpinetree:~# apk update
fetch https://dl-cdn.alpinelinux.org/alpine/edge/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/edge/community/x86_64/APKINDEX.tar.gz
v3.13.0-1030-gbabf0a1684 [https://dl-cdn.alpinelinux.org/alpine/edge/main]
v3.13.0-1035-ga3ac7373fd [https://dl-cdn.alpinelinux.org/alpine/edge/community]
OK: 14029 distinct packages available
alpinetree:~# apk upgrade
OK: 6 MiB in 14 packages
alpinetree:~# apk add g++ libc-dev
(1/14) Installing libgcc (10.2.1_pre1-r3)
(2/14) Installing libstdc++ (10.2.1_pre1-r3)
(3/14) Installing binutils (2.35.1-r1)
(4/14) Installing libgomp (10.2.1_pre1-r3)
(5/14) Installing libatomic (10.2.1_pre1-r3)
(6/14) Installing libgphobos (10.2.1_pre1-r3)
(7/14) Installing gmp (6.2.1-r0)
(8/14) Installing isl22 (0.22-r0)
(9/14) Installing mpfr4 (4.1.0-r0)
(10/14) Installing mpc1 (1.2.1-r0)
(11/14) Installing gcc (10.2.1_pre1-r3)
(12/14) Installing musl-dev (1.2.2-r1)
(13/14) Installing libc-dev (0.7.2-r3)
(14/14) Installing g++ (10.2.1_pre1-r3)
Executing busybox-1.33.0-r1.trigger
OK: 188 MiB in 28 packages
alpinetree:~# apk add zlib-dev zlib-static
(1/3) Installing pkgconf (1.7.3-r0)
(2/3) Installing zlib-dev (1.2.11-r3)
(3/3) Installing zlib-static (1.2.11-r3)
Executing busybox-1.33.0-r1.trigger
OK: 189 MiB in 31 packages
```

In this case I only installed `g++` and `libc-dev` in order to get a static `libstdc++` and the STL headers. I also installed `zlib-dev` to install zlib's headers and `zlib-static` to get `libz.a`. 
Many other libraries have also static versions available, along with their headers, inside `-dev` or `-static` packages. [^7]

#### Compiling static C++ programs

With everything now set, it's only a matter of running `clang++` with the right `--target` and `--sysroot`:

```shell
$ clang++ -B$HOME/.llvm/bin --gcc-toolchain=$HOME/alpine_tree/usr --target=x86_64-alpine-linux-musl --sysroot=$HOME/alpine_tree -L$HOME/alpine_tree/lib -std=c++17 -o zpipe zpipe.cc -lz -static
$ file zpipe
zpipe: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

The extra `--gcc-toolchain` is optional, but may help solving issues where clang the sysroot's GCC, which is needed for the C runtime startup files.
The extra `-L` for `/lib` is required because Alpine splits its libraries between `/usr/lib` and `/lib`; the latter is not automatically picked up by `clang` and `gcc`, which both usually expect libraries to be located in `$SYSROOT/usr/bin`.

Musl packages usually come with the upstream-provided shims `musl-gcc` and `musl-clang`, which wrap the system compilers in order to build and link with the alternative libc. I quite like using those, so, in order to make a similar 

[^1]: If the transfer is happening across a network and not locally, it's a good idea to compress the output tarball.

[^2]: Sadly, macOS is not supported anymore by LLD due to Mach-O support being largely unmaintained and left to rot over the last years. This leaves `ld64` (or a cross-build thereof, if you manage to build it) as the only way to link Mach-O executables (unless `ld.bfd` from `binutils` still supports it).

[^3]: `llvm-mc` can be used as a (very cumbersome) assembler but it's poorly documented. Like `gcc`, the `clang` frontend can act as an assembler, making `as` often redundant.   

[^4]: This is without talking about those criminals who hardcode `gcc` in their build scripts, but this is a rant better left for another day.

[^5]: Glibc's builtin name resolution system (NSS) is one of the main culprits, which heavily uses dlopen()/dlsym(). This is due to its heavy usage of plugins, which is meant to provide support for extra third-party resolvers such as mDNS.

[^6]: `systemd-nspawn` can also double as a lighter alternative to VMs, using the `--boot` option to spawn an init process inside the container. See [this very helpful gist](https://gist.github.com/sfan5/52aa53f5dca06ac3af30455b203d3404) to learn how to make bootable containers for distributions based on OpenRC, like Alpine.

[^7]: Sadly, Alpine for reasons unknown to me, does not ship the static version of certain libraries (like `libfmt`). Given that embedding a local copy of third party dependencies is common practice nowadays for C++, this is not too problematic. 