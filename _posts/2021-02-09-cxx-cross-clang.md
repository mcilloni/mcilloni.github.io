---
layout: post
title: Cross compiling made easy, using Clang and LLVM
---

Anyone who ever tried to cross-compile a C/C++ program knows how big a PITA the whole process could be. The main reasons for this sorry state of things are generally how byzantine build systems tend to be when configuring for cross-compilation, and how messy it is to set-up your cross toolchain in the first place.   

One of the main culprits in my experience has been the GNU toolchain, the decades-old behemoth upon which the POSIXish world has been built for years.
Like many compilers of yore, GCC and its `binutils` brethren were never designed with the intent to support multiple targets within a single setup, with he only supported approach being installing a full cross build for each triple you wish to target on any given host. 

For instance, assuming you wish to build something for FreeBSD on your Linux machine using GCC, you need:

- A GCC + binutils install for your host triplet (i.e., `x86_64-pc-linux-gnu` or similar);
- A GCC + binutils complete install for your target triplet (i.e. `x86_64-unknown-freebsd12.2-gcc`, `as`, `nm`, etc)
- A sysroot containing the necessary libraries and headers, which you can either build yourself or promptly steal from a running installation of FreeBSD.

This process is sometimes made simpler by Linux distributions or hardware vendors offering a selection of prepackaged compilers, but this will never suffice due to the sheer amount of possible host-target combinations. This sometimes means you have to build the whole toolchain yourself, something that, unless you rock a quite beefy CPU, tends to be a massive waste of time and power.

## Clang as a cross compiler

This annoying limitation is one of the reasons why I got interested in LLVM (and thus Clang), which is by-design a full-fledged cross compiler toolchain and is mostly compatible with GNU. A single install can output and compile code _for every supported target_, as long as a complete sysroot is available at build time.

I found this to be a game-changer, and, while it can't still compete in convenience with modern language toolchains (such as Go's gc and `GOARCH`/`GOOS`), it's night and day better than the rigmarole of setting up GNU toolchains. You can now just fetch whatever your favorite package management system has available in its repositories (as long as it's not extremely old), and avoid messing around with multiple installs of GCC.

Until a few years ago, the whole process wasn't as smooth as it could be. Due to LLVM not having a full toolchain yet available, you were still supposed to provide a `binutils` build specific for your target. While this is generally much more tolerable than building the whole compiler (`binutils` is relatively fast to build), it was still somewhat of a nuisance, and I'm glad that `llvm-mc` (LLVM's integrated assembler) and `lld` (universal linker) are finally stable and as flexible as the rest of LLVM.

With the toolchain now set, the next step becomes to obtain a sysroot in order to provide the needed headers and libraries to compile and link for your target.

## Obtaining a sysroot 
A super fast way to find a working system directory for a given OS is to rip it straight out of an existing system (a Docker container image will often also do).
For instance, this is how I used `tar` through `ssh` as a quick way to extract a working sysroot from a FreeBSD 13-CURRENT AArch64 VM [^1]:

```
$ mkdir ~/farm_tree
$ ssh FARM64 'tar cf - /lib /usr/include /usr/lib /usr/local/lib /usr/local/include' | bsdtar xvf - -C $HOME/farm_tree/
```

## Invoking the cross compiler

With everything set, it's now only a matter of invoking Clang with the right arguments:

```shell
$  clang++ --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree -fuse-ld=lld -stdlib=libc++ -o zpipe zpipe.cc -lz --verbose
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

In the snipped above, I have managed to compile and link a C++ file into an executable for AArch64 FreeBSD, all while using just the `clang` and `lld` I had already installed on my GNU/Linux system.

More in detail:
1. `--target` switches the LLVM default target (`x86_64-pc-linux-gnu`) to `aarch64-pc-freebsd`, thus enabling cross-compilation.
2. `--sysroot` forces Clang to assume the specified path as root when searching headers and libraries, instead of the usual paths. Note that sometimes this setting might not be enough, especially if the target uses GCC and Clang somehow fails to detect its install path. This can be easily fixed by specifying `--gcc-toolchain`, which clarifies where to search for GCC installations.
3. `-fuse-ld=lld` tells Clang to use `lld` instead whatever default the platform uses. As I will explain below, it's highly unlikely that the system linker understands foreign targets, while LLD can natively support _almost_ every binary format and OS [^2].
4. `-stdlib=libc++` is needed here due to Clang failing to detect that FreeBSD on AArch64 uses LLVM's `libc++` instead of GCC's `libstdc++`.
5. `-lz` is also specified to show how Clang can also resolve other libraries inside the sysroot without issues, in this case, `zlib`.

The final test is now to copy the binary to our target system (i.e. the VM we ripped the sysroot from before) and check if it works as expected:

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

Success! It's now possible to use this cross toolchain to build larger programs, and below I'll give a quick example to how to use it to build real projects. 

### Optional: creating an LLVM toolchain directory

LLVM provides a mostly compatible counterpart for almost every tool shipped by `binutils` (with the notable exception of `as` [^3]), prefixed with `llvm-`. 

The most critical of these is LLD, which is a drop in replacement for a plaform's system linker, capable to replace both GNU `ld.bfd` and `gold` on GNU/Linux or BSD, and Microsoft's `LINK.EXE` when targeting MSVC. It supports linking on (almost) every platform supported by LLVM, thus removing the nuisance to have multiple specific linkers installed.

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

### Optional: creating Clang wrappers to simplify cross-compilation

I happened to notice that certain build systems (and with _"certain"_ I mean some poorly written `Makefile`s and sometimes Autotools) have a tendency to misbehave when `$CC`, `$CXX` or `$LD` contain spaces or multiple parameters. This might become a recurrent issue if we need to invoke `clang` with several arguments. [^4] 

Given also how unwieldy it is to remember to write all of the parameters correctly everywhere, I usually write quick wrappers for `clang` and `clang++` in order to simplify building for a certain target:

```shell
$ cat ~/.local/bin/aarch64-pc-freebsd-clang
#!/usr/bin/env sh

exec /usr/bin/clang -B$HOME/.llvm/bin --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree "$@"
$ cat ~/.local/bin/aarch64-pc-freebsd-clang++
#!/usr/bin/env sh

exec /usr/bin/clang++ -B$HOME/.llvm/bin -stdlib=libc++ --target=aarch64-pc-freebsd --sysroot=$HOME/farm_tree "$@"	
```

If created in a directory inside $PATH, these script can used everywhere as standalone commands:

```shell
$ aarch64-pc-freebsd-clang++ -o tst tst.cc -static
$ file tst
tst: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), statically linked, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped

```

## Cross-building with Autotools, CMake and Meson

Autotools, CMake, and Meson are arguably the most popular building systems for C and C++ open source projects (sorry, SCons).
All of three support cross-compiling out of the box, albeit with some caveats.

### Autotools

Over the years, Autotools has been famous for being horrendously clunky and breaking easily. While this reputation is definitely well earned, it's still widely used by most large GNU projects. Given it's been around for decades, it's quite easy to find support online when something goes awry (sadly, this is not also true when writing `.ac` files). When compared to its more modern breathren, it doesn't require any toolchain file or extra configuration when cross compiling, being only driven by command line options.

A `./configure` script (either generated by autoconf or shipped by a tarball alongside source code) _usually_ supports the `--host` flag, allowing the user to specify the triple of the _host_ on which the final artifacts are meant to be run. This flags activates cross compilation, and causes the _"auto-something"_ array of tools to try to detect the correct compiler for the target, which generally assumed to be `some-triple-gcc`.

For instance, let's try to configure `binutils` version 2.35.1 for `aarch64-pc-freebsd`, using the Clang wrapper introduced above:

```shell
$ tar xvf binutils-2.35.1.tar.xz
$ mkdir binutils-2.35.1/build # always create a build directory to avoid messing up the source tree
$ cd binutils-2.35.1/build
$ env CC='aarch64-pc-freebsd-clang' CXX='aarch64-pc-freebsd-clang++' AR=llvm-ar ../configure --build=x86_64-pc-linux-gnu --host=aarch64-pc-freebsd --
enable-gold=yes
checking build system type... x86_64-pc-linux-gnu
checking host system type... aarch64-pc-freebsd
checking target system type... aarch64-pc-freebsd
checking for a BSD-compatible install... /usr/bin/install -c
checking whether ln works... yes
checking whether ln -s works... yes
checking for a sed that does not truncate output... /usr/bin/sed
checking for gawk... gawk
checking for aarch64-pc-freebsd-gcc... aarch64-pc-freebsd-clang
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables...
checking whether we are cross compiling... yes
checking for suffix of object files... o
checking whether we are using the GNU C compiler... yes
checking whether aarch64-pc-freebsd-clang accepts -g... yes
checking for aarch64-pc-freebsd-clang option to accept ISO C89... none needed
checking whether we are using the GNU C++ compiler... yes
checking whether aarch64-pc-freebsd-clang++ accepts -g... yes
[...]
```

The invocation of `./configure` above specifies that I want autotools to:

1. Configure for building on an `x86_64-pc-linux-gnu` host (which I specified using `--build`);
2. Build binaries that will run on `aarch64-pc-freebsd`, using the `--host` switch;
3. Use the Clang wrappers made above as C and C++ compilers;
4. Use `llvm-ar` as the target `ar`.

I also specified to build the Gold linker, which is written in C++ and it's a good test for well our improvised toolchain handles compiling C++. 

If the configuration step doesn't fail for some reason (it shouldn't), it's now time to run GNU Make to build `binutils`:

```shell
$ make -j16 # because I have 16 theads on my system
[ lots of output]
$ mkdir dest
$ make DESTDIR=$PWD/dest install # install into a fake tree
```

There should now be executable files and libraries inside of the fake tree generated by `make install`. A quick test using `file` confirms they have been correctly built for `aarch64-pc-freebsd`:

```shell
$ file dest/usr/local/bin/ld.gold
dest/usr/local/bin/ld.gold: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped
```

### CMake

The simplest way to set CMake to configure for an arbitrary target is to write a _toolchain file_. These usually consist of a list of declarations that instructs CMake on how it is supposed to use a given toolchain, specifying parameters like the target operating system, the CPU architecture, the name of the C++ compiler, and such.

One reasonable toolchain file for the `aarch64-pc-freebsd` triple written as follows:

```CMake
set(CMAKE_SYSTEM_NAME FreeBSD)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_SYSROOT $ENV{HOME}/farm_tree)

set(CMAKE_C_COMPILER aarch64-pc-freebsd-clang)
set(CMAKE_CXX_COMPILER aarch64-pc-freebsd-clang++)
set(CMAKE_AR llvm-ar)

# these variables tell CMake to avoid using any binary it finds in 
# the sysroot, while picking headers and libraries exclusively from it 
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

In this file, I specified the wrapper created above as the cross compiler for C and C++ for the target. It should be possible to also use plain Clang with the right arguments, but it's much less straightforward and potentially more error-prone. 

In any case, it is _very_ important to indicate the `CMAKE_SYSROOT` and `CMAKE_FIND_ROOT_PATH_MODE_*` variables, or otherwise CMake could wrongly pick packages from the host with disastrous results.

It is now only a matter of setting `CMAKE_TOOLCHAIN_FILE` with the path to the toolchain file when configuring a project. To better illustrate this, I will now also build `{fmt}` (which is an amazing C++ library you should definitely use) for `aarch64-pc-freebsd`:

```shell
$  git clone https://github.com/fmtlib/fmt
Cloning into 'fmt'...
remote: Enumerating objects: 45, done.
remote: Counting objects: 100% (45/45), done.
remote: Compressing objects: 100% (33/33), done.
remote: Total 24446 (delta 17), reused 12 (delta 7), pack-reused 24401
Receiving objects: 100% (24446/24446), 12.08 MiB | 2.00 MiB/s, done.
Resolving deltas: 100% (16551/16551), done.
$ cd fmt
$ cmake -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=$HOME/toolchain-aarch64-freebsd.cmake -DBUILD_SHARED_LIBS=ON -DFMT_TEST=OFF .
-- CMake version: 3.19.4
-- The CXX compiler identification is Clang 11.0.1
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /home/marco/.local/bin/aarch64-pc-freebsd-clang++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Version: 7.1.3
-- Build type: Release
-- CXX_STANDARD: 11
-- Performing Test has_std_11_flag
-- Performing Test has_std_11_flag - Success
-- Performing Test has_std_0x_flag
-- Performing Test has_std_0x_flag - Success
-- Performing Test SUPPORTS_USER_DEFINED_LITERALS
-- Performing Test SUPPORTS_USER_DEFINED_LITERALS - Success
-- Performing Test FMT_HAS_VARIANT
-- Performing Test FMT_HAS_VARIANT - Success
-- Required features: cxx_variadic_templates
-- Performing Test HAS_NULLPTR_WARNING
-- Performing Test HAS_NULLPTR_WARNING - Success
-- Looking for strtod_l
-- Looking for strtod_l - not found
-- Configuring done
-- Generating done
-- Build files have been written to: /home/marco/fmt/build
```

Compared with Autotools, the command line passed to `cmake` is very simple and doesn't need too much explanation. After the configuration step is finished, it's only a matter to compile the project and get `ninja` or `make` to install the resulting artifacts somewhere.

```shell
$ cmake --build build
[4/4] Creating library symlink libfmt.so.7 libfmt.so
$ mkdir dest
$ env DESTDIR=$PWD/dest cmake --build build -- install
[0/1] Install the project...
-- Install configuration: "Release"
-- Installing: /home/marco/fmt/dest/usr/local/lib/libfmt.so.7.1.3
-- Installing: /home/marco/fmt/dest/usr/local/lib/libfmt.so.7
-- Installing: /home/marco/fmt/dest/usr/local/lib/libfmt.so
-- Installing: /home/marco/fmt/dest/usr/local/lib/cmake/fmt/fmt-config.cmake
-- Installing: /home/marco/fmt/dest/usr/local/lib/cmake/fmt/fmt-config-version.cmake
-- Installing: /home/marco/fmt/dest/usr/local/lib/cmake/fmt/fmt-targets.cmake
-- Installing: /home/marco/fmt/dest/usr/local/lib/cmake/fmt/fmt-targets-release.cmake
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/args.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/chrono.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/color.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/compile.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/core.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/format.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/format-inl.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/locale.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/os.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/ostream.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/posix.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/printf.h
-- Installing: /home/marco/fmt/dest/usr/local/include/fmt/ranges.h
-- Installing: /home/marco/fmt/dest/usr/local/lib/pkgconfig/fmt.pc
$  file dest/usr/local/lib/libfmt.so.7.1.3
dest/usr/local/lib/libfmt.so.7.1.3: ELF 64-bit LSB shared object, ARM aarch64, version 1 (FreeBSD), dynamically linked, for FreeBSD 13.0 (1300136), with debug_info, not stripped
```

### Meson

Like CMake, Meson relies on toolchain files (here called _"cross files"_) to specify which tools should be used when building for a given target. Thanks to being written in a TOML-like language, they are very straightforward:

```meson
$ cat meson_aarch64_fbsd_cross.txt
[binaries]
c = '/home/marco/.local/bin/aarch64-pc-freebsd-clang'
cpp = '/home/marco/.local/bin/aarch64-pc-freebsd-clang++'
ld = '/usr/bin/ld.lld'
ar = '/usr/bin/llvm-ar'
objcopy = '/usr/bin/llvm-objcopy'
strip = '/usr/bin/llvm-strip'

[properties]
ld_args = ['--sysroot=/home/marco/farm_tree']

[host_machine]
system = 'freebsd'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'

```

This cross-file can then be specified to `meson setup` using the `--cross-file` option [^5], with everything else remaining the same as with every other Meson build.

And, well, this is basically it: like with CMake, the whole process is relatively painless and foolproof. 
For the sake of completeness, this is how to build `dav1d`, VideoLAN's AV1 decoder, for `aarch64-pc-freebsd`:

```shell
$ git clone https://code.videolan.org/videolan/dav1d
Cloning into 'dav1d'...
warning: redirecting to https://code.videolan.org/videolan/dav1d.git/
remote: Enumerating objects: 164, done.
remote: Counting objects: 100% (164/164), done.
remote: Compressing objects: 100% (91/91), done.
remote: Total 9377 (delta 97), reused 118 (delta 71), pack-reused 9213
Receiving objects: 100% (9377/9377), 3.42 MiB | 54.00 KiB/s, done.
Resolving deltas: 100% (7068/7068), done.
$ meson setup build --cross-file ../meson_aarch64_fbsd_cross.txt --buildtype release
The Meson build system
Version: 0.56.2
Source dir: /home/marco/dav1d
Build dir: /home/marco/dav1d/build
Build type: cross build
Project name: dav1d
Project version: 0.8.1
C compiler for the host machine: /home/marco/.local/bin/aarch64-pc-freebsd-clang (clang 11.0.1 "clang version 11.0.1")
C linker for the host machine: /home/marco/.local/bin/aarch64-pc-freebsd-clang ld.lld 11.0.1
[ output cut ]
$ meson compile -C build
Found runner: ['/usr/bin/ninja']
ninja: Entering directory `build'
[129/129] Linking target tests/seek_stress
$ mkdir dest
$ env DESTDIR=$PWD/dest meson install -C build
ninja: Entering directory `build'
[1/11] Generating vcs_version.h with a custom command
Installing src/libdav1d.so.5.0.1 to /home/marco/dav1d/dest/usr/local/lib
Installing tools/dav1d to /home/marco/dav1d/dest/usr/local/bin
Installing /home/marco/dav1d/include/dav1d/common.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/include/dav1d/data.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/include/dav1d/dav1d.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/include/dav1d/headers.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/include/dav1d/picture.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/build/include/dav1d/version.h to /home/marco/dav1d/dest/usr/local/include/dav1d
Installing /home/marco/dav1d/build/meson-private/dav1d.pc to /home/marco/dav1d/dest/usr/local/lib/pkgconfig
$ file dest/usr/local/bin/dav1d
dest/usr/local/bin/dav1d: ELF 64-bit LSB executable, ARM aarch64, version 1 (FreeBSD), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 13.0 (1300136), FreeBSD-style, with debug_info, not stripped

```

## Bonus: static linking with musl and Alpine Linux

Statically linking a C or C++ program can sometimes save you a lot of library compatibility headaches, especially when you can't control what's going to be installed on whatever you plan to target.
Building static binaries is however quite complex on GNU/Linux, due to Glibc actively discouraging people from linking it statically. [^6]

Musl is a very compatible standard library implementation for Linux that plays much nicer with static linking, and it is now shipped by most major distributions. These packages often suffice in building your code statically, at least as long as you plan to stick with plain C. 

The situation gets much more complicated if you plan to use C++, or if you need additional components. Any library shipped by a GNU/Linux system (like `libstdc++`, `libz`, `libffi` and so on) is usually only built for Glibc, meaning that any library you wish to use must be rebuilt to target Musl. This also applies to `libstdc++`, which inevitably means either recompiling GCC or building a copy of LLVM's `libc++`.

Thankfully, there are several distributions out there that target _"Musl-plus-Linux"_, everyone's favorite being Alpine Linux. It is thus possible to apply the same strategy we used above to obtain a `x86_64-pc-linux-musl` sysroot complete of libraries and packages built for Musl, which can then be used by Clang to generate 100% static executables. 

### Setting up an Alpine container

A good starting point is the minirootfs tarball provided by Alpine, which is meant for containers and tends to be very small:

```shell
$ wget -qO - https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.1-x86_64.tar.gz | gunzip | sudo tar xfp - -C ~/alpine_tree
```

It is now possible to chroot inside the image in `~/alpine_tree` and set it up, installing all the packages you may need. 
I prefer in general to use `systemd-nspawn` in lieu of `chroot` due to it being vastly better and less error prone. [^7]

```shell
$ $  sudo systemd-nspawn -D alpine_tree
Spawning container alpinetree on /home/marco/alpine_tree.
Press ^] three times within 1s to kill container.
alpinetree:~# 
```

We can now (optionally) switch to the `edge` branch of Alpine for newer packages by editing `/etc/apk/repositories`, and then install the required packages containing any static libraries required by the code we want to build:

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

In this case I installed `g++` and `libc-dev` in order to get a static copy of `libstdc++`, a static `libc.a` (Musl) and their respective headers. I also installed `zlib-dev` and `zlib-static` to install zlib's headers and `libz.a`, respectively. 
As a general rule, Alpine usually ships static versions available inside `-static` packages, and headers as `somepackage-dev`. [^8]

Also, remember every once in a while to run `apk upgrade` inside the _sysroot_ in order to keep the local Alpine install up to date. 

### Compiling static C++ programs

With everything now set, it's only a matter of running `clang++` with the right `--target` and `--sysroot`:

```shell
$ clang++ -B$HOME/.llvm/bin --gcc-toolchain=$HOME/alpine_tree/usr --target=x86_64-alpine-linux-musl --sysroot=$HOME/alpine_tree -L$HOME/alpine_tree/lib -std=c++17 -o zpipe zpipe.cc -lz -static
$ file zpipe
zpipe: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

The extra `--gcc-toolchain` is optional, but may help solving issues where compilation fails due to Clang not detecting where GCC and the various crt*.o files reside in the sysroot.
The extra `-L` for `/lib` is required because Alpine splits its libraries between `/usr/lib` and `/lib`, and the latter is not automatically picked up by `clang`, which both usually expect libraries to be located in `$SYSROOT/usr/bin`.

### Writing a wrapper for static linking with Musl and Clang

Musl packages usually come with the upstream-provided shims `musl-gcc` and `musl-clang`, which wrap the system compilers in order to build and link with the alternative libc. 
In order to provide a similar level of convenience, I quickly whipped up the following Perl script:

```perl
#!/usr/bin/env perl

use strict;
use utf8;
use warnings;
use v5.30;

use List::Util 'any';

my $ALPINE_DIR = $ENV{ALPINE_DIR} // "$ENV{HOME}/alpine_tree";
my $TOOLS_DIR = $ENV{TOOLS_DIR} // "$ENV{HOME}/.llvm/bin";

my $CMD_NAME = $0 =~ /\+\+/ ? 'clang++' : 'clang';
my $STATIC = $0 =~ /static/;

sub clang {
	exec $CMD_NAME, @_ or return 0;
}

sub main {
	my $compile = any { /^\s*-c|-S\s*$/ } @ARGV;

	my @args = (
		 qq{-B$TOOLS_DIR},
		 qq{--gcc-toolchain=$ALPINE_DIR/usr},
		 '--target=x86_64-alpine-linux-musl',
		 qq{--sysroot=$ALPINE_DIR},
		 qq{-L$ALPINE_DIR/lib},
		 @ARGV,
	);

	unshift @args, '-static' if $STATIC and not $compile;

	exit 1 unless clang @args;
}

main;
```

This wrapper is more refined than the FreeBSD AArch64 wrapper above.
For instance, it can infer C++ if invoked as `clang++`, or always force `-static` if called from a _symlink_ containing `static` in its name:

```shell
$ ls -la $(which musl-clang++)
lrwxrwxrwx    10 marco marco 26 Jan 21:49  /home/marco/.local/bin/musl-clang++ -> musl-clang
$ ls -la $(which musl-clang++-static)
lrwxrwxrwx    10 marco marco 26 Jan 22:03  /home/marco/.local/bin/musl-clang++-static -> musl-clang
$ musl-clang++-static -std=c++17 -o zpipe zpipe.cc -lz # automatically infers C++ and -static
$ file zpipe
zpipe: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

It is thus possible to force Clang to only ever link `-static` by setting $CC to `musl-clang-static`, which can be useful with build systems that don't play nicely with statically linking. From my experience, the worst offenders in this regard are Autotools (sometimes) and poorly written Makefiles.

### Conclusions

Cross-compiling C and C++ is and will probably always be an annoying task, but it has got much better since LLVM became production-ready and widely available. Clang's `-target` option has saved me countless man-hours that I would have instead wasted building and re-building GCC and Binutils over and over again. 

Alas, all that glitters is not gold, as is often the case. There is still code around that only builds with GCC due to nasty GNUisms (I'm looking at you, Glibc). Cross compiling for Windows/MSVC is also bordeline unfeasible due to how messy the whole Visual Studio toolchain is. 

Furthermore, while targeting arbitrary triples with Clang is now definitely simpler that it was, it still pales in comparison to how trivial cross compiling with Rust or Go is. 

One special mention among these new languages should go to Zig, and its goal to also make C and C++ easy to build for other platforms.

The `zig cc` and `zig c++` commands have the potential to become an amazing swiss-army knife tool for cross compiling, thanks to Zig shipping a copy of `clang` and large chunks of projects such as Glibc, Musl, libc++ and MinGW. Any required library is then built _on-the-fly_ when required:

```shell
$ zig c++ --target=x86_64-windows-gnu -o str.exe str.cc
$ file str.exe
str.exe: PE32+ executable (console) x86-64, for MS Windows
```

While I think this is not yet perfect, it already feels almost like magic. I dare to say, this might really become a killer selling point for Zig, making it attractive even for those who are not interested in using the language itself. 

[^1]: If the transfer is happening across a network and not locally, it's a good idea to compress the output tarball.

[^2]: Sadly, macOS is not supported anymore by LLD due to Mach-O support being largely unmaintained and left to rot over the last years. This leaves `ld64` (or a cross-build thereof, if you manage to build it) as the only way to link Mach-O executables (unless `ld.bfd` from `binutils` still supports it).

[^3]: `llvm-mc` can be used as a (very cumbersome) assembler but it's poorly documented. Like `gcc`, the `clang` frontend can act as an assembler, making `as` often redundant.   

[^4]: This is without talking about those criminals who hardcode `gcc` in their build scripts, but this is a rant better left for another day.

[^5]: In the same fashion, it is also possible to tune the native toolchain for the current machine using a _native file_ and the `--native-file` toggle.

[^6]: Glibc's builtin name resolution system (NSS) is one of the main culprits, which heavily uses dlopen()/dlsym(). This is due to its heavy usage of plugins, which is meant to provide support for extra third-party resolvers such as mDNS.

[^7]: `systemd-nspawn` can also double as a lighter alternative to VMs, using the `--boot` option to spawn an init process inside the container. See [this very helpful gist](https://gist.github.com/sfan5/52aa53f5dca06ac3af30455b203d3404) to learn how to make bootable containers for distributions based on OpenRC, like Alpine.

[^8]: Sadly, Alpine for reasons unknown to me, does not ship the static version of certain libraries (like `libfmt`). Given that embedding a local copy of third party dependencies is common practice nowadays for C++, this is not too problematic. 

[^9]: The `zig cc` and `zig c++` commands are also good options for cross compiling, thanks to Zig bundling a whole copy of `clang` and parts of Glibc, Musl, libc++, MinGW, ... I find it still slightly clunkier than I'd like, but it feels very promising.