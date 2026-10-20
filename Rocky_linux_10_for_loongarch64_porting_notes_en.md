# Introduction

Rocky Linux is an operating system rebuilt from RHEL source code and binary-compatible with RHEL. After CentOS became a rolling release, Rocky Linux is the replacement operating system for CentOS.

Rocky Linux 10.0 was released on June 11, 2025, 10.1 on November 25, 2025, and 10.2 on May 28, 2026. The versions of the compiler, kernel, glibc and other toolchain components in the operating system are relatively new, so it should be quite convenient to port it to the LoongArch architecture:

- GCC 14.3
- glibc 2.39
- Annobin 12.99
- binutils 2.41
- kernel 6.12.0.

I want to learn how to port a LoongArch operating system. With reference to Sun Haiyong's "Exploring with 'Chip': Teaching You to Build a Linux System for the Loongson Platform", I began porting the operating system.

In Rocky Linux 10.1, the version baseline of the toolchain is already higher than the versions in the "Open Source Ecosystem Development Cooperation Initiative" proposed by the major domestic operating system vendors.

# Preparation

## Preparing the System Environment

There are already many other operating systems for the LoongArch architecture, such as Debian, AOSC, and Fedora Remix.

I plan to build the temporary operating system using native compilation rather than cross-compilation. I plan to use Debian as the host system and begin my build.

First, set up the system environment. Following Mr. Sun's book, under the Debian system, some necessary software needs to be installed

```bash
sudo apt install build-essential rpm librpm-dev git python3-dev dnf texinfo \
    zlib-dev gettext gettext-base libgettextp0-dev tcl libncurses-dev bc \
    wget meson ninja-build gperf help2man libssl-dev
```

Download the source package of Rock Linux 10.1.

Use rsync to download the source code repository. I chose the Nanjing University mirror.

```bash
 #!/bin/bash
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/BaseOS/source/tree/ BaseOS
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/AppStream/source/tree/ AppStream
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/devel/source/tree/ devel
```

Actually, downloading only the packages in devel should be enough. The source code is 24GB in size.

Create a dedicated user and configure it. In Mr. Sun's book, in order to build the cross toolchain, a small initial system was specially installed using the dnf command and run using chroot.

I plan to skip this step and use the native system directly, but create a new user to do this.
```bash
groupadd rocky
useradd -s /bin/bash -g rocky -m -k /dev/null rocky
usermod -a -G sudo rocky
```
Under the Debian system, there is no wheel group; the equivalent is the sudo group.

`-k /dev/null` ensures that the newly added user rocky is clean.

Set a password for the new user
```bash
passwd rocky
```

Set the password to loongarch

Create the necessary directories
```bash
mkdir -p rocky_loong64/
cd rocky_loong64
mkdsir sources build tools cross-tools
ln -sv /home/rocky/rocky_loong64/tools /
ln -sv /home/rocky/rocky_loong64/cross-tools /
mkdir -pv /tools/lib /tools/lib64 /tools/include
```

**Note**: The `/tools/lib` directory must be created manually, because one of the library search paths of the subsequently compiled gcc is set to `/home/rocky/rocky_loong64/tools/lib/../lib64/`. If `/tools/lib` does not exist, then `../lib64` cannot be found via the relative path.

**Remember!**

Regarding the three concepts `CROSS_HOST`, `CROSS_TARGET`, and `CROSS_BUILD`:

`CROSS_BUILD` is the platform currently running.

`CROSS_HOST` is the platform on which the cross-compilation chain runs. The prefix set by `CROSS_HOST` will be used for the cross toolchain.

`CROSS_TARGET` is the platform on which the binary programs produced by the toolchain are to run.

Take two cases as examples

|                      |   Cross-compiling MIPS on an X86 platform   |   "Cross"-compiling loongarch64 on Loongarch|
|----------------------| ----------------------------| ------------------------------------|
|  CROSS_HOST          | x86_64-cross-linux-gnu      |  loongarch64-cross-linux-gnu        |
|  CROSS_BUILD         |  x86_64-unkown-linux-gnu    |   loongarch64-debian-linux-gnu      |
|  CROSS_TARGET        | mips64el-unknown-linux-gnu  |  loongarch64-unknown-linux-gnu      |

The files in /cross-tools run on the cross-compilation platform. For the x86_64 platform, CROSS_HOST is x86_64-cross-linux-gnu

Set `~/.bash_profile` and `~/.bashrc`; for the specific contents, refer to the [scripts](scripts/cross/prepare-cross-chain.sh)

Put all the source rpms packages downloaded earlier into one directory
```bash
find ./ -name *.rpm |xargs mv -v -t ~/rocky_loong64/sources/
```

# Overview of the Build Process

The porting of the Rocky Linux operating system can be simply divided into three stages: the compilation of the toolchain, the compilation of the temporary operating system, and the compilation of the operating system that includes a package management system. The third stage can further be divided into two sub-stages: the bootstrap stage and the full compilation stage.

The first stage mainly requires completing the compilation of the toolchain, such as binutils, glibc, and gcc, to prepare for the subsequent compilation of the temporary system. This stage can be completed on an X86 machine using cross-compilation; the compiled toolchain runs on the X86 platform and produces LoongArch executable programs; it can also be completed directly on a LoongArch machine, running on the LoongArch platform and producing LoongArch executable programs.

The second stage uses the previously compiled toolchain to complete the compilation of the temporary operating system. The entire process is very similar to Linux From Scratch. The final product is an operating system that can run on LoongArch hardware. For a simple operating system port, the task is already considered complete at this point. The problem with the temporary operating system is that it lacks a package management system, has no soul, and is too streamlined in functionality. This stage also requires compiling the toolchain once, and this toolchain runs on the LoongArch platform.

The third stage needs to be completed within the temporary operating system. First, use the package management system to build a toolchain, and use this toolchain together with the temporary system to complete a new operating system in a bootstrap manner, gradually replacing the temporary operating system. During this process, in order to be able to compile software packages, a lot of simplification is done to the compilation contents, resulting in a crippled operating system. Then, within this crippled operating system, the toolchain and the entire operating system are built completely once again.

The above is my shallow understanding of the entire porting and build process.

# Building the Cross Toolchain

Next, begin building an independent toolchain.

Although cross-compilation is not needed, building an independent, standard toolchain is still necessary. This avoids various strange problems caused by using the toolchain in the host system.

**Note**: In operating systems such as Debian/Ubuntu, `sh` points to `dash`, not `bash`. This will cause some scripts in Fedora to fail to run correctly. Therefore, it is recommended to adopt the approach used in LFS to ensure that `sh` points to `bash`.

## Kernel Header Files

Here, let's still not use Fedora's kernel for now, and instead use Debian's built-in kernel. When building the cross-compilation chain, this step can use the kernel source package provided by Fedora.

I used the linux-source source code of 6.17.11, to which the necessary kernel patches have already been added.

```bash
make mrproper
make ARCH=loongarch headers
find usr/include -name “.*” -delete
rm usr/include/Makefile
mkdir -pv /tools/include
cp -rv usr/include/* /tools/include
```

## binutils Cross Tool

Since I am not using the Fedora system here, the --nodeps parameter needs to be added when running rpmbuild.

```bash
rpmbuild -bp rpmbuild/SPECS/binutils.spec --nodeps
```

If you encounter such a build error when running this command:
```text
+ sed -i -e s/%{release}/58.2/g bfd/Makefile{.am,.in}
sed: can't read bfd/Makefile{.am,.in}: No such file or directory
```
it means the `sh` you are using is `dash`, not `bash`. **Please ensure that `sh` points to `bash`**.

After compilation is complete, perform a correctness check. I set CROSS_TARGET to `loongarch64-unknown-linux-gnu`, so the prefix of the binary files in /cross-tools/bin is also `loongarch64-unknown-linux-gnu`.

There are 16 generated binary files, including nm, ld, ar, as, strip, ld, etc.

Check the search path of the ld program:
```bash
/cross-tools/bin/loongarch64-unknown-linux-gnu-ld --verbose | grep SEARCH_DIR
```
Output:
```text
SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib");
```

As expected.

## gmp

Key: The two files `config.guess` and `config.sub` need to be replaced.

Check: In `/cross-tools/lib`, files such as `libgmp.so` and `libgmpxx.so` have been produced.

## MPFR

Check: In `/cross-tools/lib`, the produced file is `libmpfr.so`

## libmpc

Check: In `/cross-tools/lib`, the produced file is `libmpc.so`

## ISL

Rock Linux 10 does not provide an ISL source package, but the source of isl 0.24 is included in gcc. Compile isl from here.

Check: In `/cross-tools/lib`, the produced file is `libisl.so`

## CLooG

Rock Linux 10 does not provide a CLooG source package; it is presumably no longer needed.

## GCC-PASS1

The first compilation of GCC.

When directly running `rpmbuild -bp rpmbuild/SPECS/gcc.spec`, a build error is encountered:
```text
error: invalid syntax in lua scriptlet: [string "%postun"]:10: unexpected symbol near '%''
```
I don't know where this problem comes from, but after adding a space before %post and %postun, the problem disappeared.
```bash
%post -n libgcc -p <lua>
if posix.access ("/sbin/ldconfig", "x") then
  local pid = posix.fork ()
  if pid == 0 then
    posix.exec ("/sbin/ldconfig")
  elseif pid ~= -1 then
    posix.wait (pid)
  end
end

 %postun -n libgcc -p <lua>
if posix.access ("/sbin/ldconfig", "x") then
  local pid = posix.fork ()
  if pid == 0 then
    posix.exec ("/sbin/ldconfig")
  elseif pid ~= -1 then
    posix.wait (pid)
  end
```

Next, modify the GCC source code to change the library search path from the default `/lib/` etc. to `/tools/lib/`

The modification is divided into two parts

1. Modify /lib

```bash
for file in $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
    sed -i.orig -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' -e 's@/usr@/tools@' $file
done

#ATTENTION: Do not forget to update this file!
sed -i.orig 's@"\/lib" ABI_GRLEN_SPEC "\/ld-linux-loongarch-" ABI_SPEC "\.so\.1"@"\/tools/lib" ABI_GRLEN_SPEC "\/ld-linux-loongarch-" ABI_SPEC "\.so\.1"@g' gcc/config/loongarch/gnu-user.h

```
Note that one purpose of the above modification is to ensure that DYNAMIC_LINKER uses the `ld-*.so` in `/tools/lib*`/. However, for the LoongArch architecture, the definition related to DYNAMIC_LINKER is in the `gcc/config/loongarch/gnu-user.h` file, so in the above sed script, a statement to modify the gnu-user.h file should be added.

2. Modify the `STANDARD_STARTFILE_PREFIX_*` macros

    The `gcc/gcc.cc` file and `gcc/config/loongarch/linux.h` need to be modified

    Since in `loongarch/linux.h`, the way `STANDARD_STARTFILE_PREFIX_*` is defined is already different from MIPS,

    `gcc/config/loongarch/linux.h` is written like this
```c
#define STANDARD_STARTFILE_PREFIX_1 "/tools/" ABI_LIBDIR "/"
#define STANDARD_STARTFILE_PREFIX_2 ""
```
3. configure, compile.

 Here is the specific build [script](scripts/cross/build-cross-gcc-pass1.sh). At this point, gcc cannot compile executable programs that can run normally; it can only compile libraries.

    The `--disable-shared` option was used during compilation, so libgcc.a was produced, and there is no libgcc.so file.

    In /cross-tools/bin/, there are files such as `loongarch64-unknown-linux-gnu-gcc`.

    Check the library search path of the gcc program:
  ```bash
    /cross-tools/bin/loongarch64-unknown-linux-gnu-gcc  -print-search-dirs
  ```
Output:
```text
install: /home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/
programs: =/home/rocky/rocky_loong64/cross-tools/bin/../libexec/gcc/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../libexec/gcc/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/bin/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/bin/
libraries: =/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/../lib64/:/home/rocky/rocky_loong64/tools/lib/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/tools/lib/../lib64/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/:/home/rocky/rocky_loong64/tools/lib/
```
Note the `home/rocky/rocky_loong64/tools/lib/../lib64/` path among them; it is precisely this path that requires the `tools/lib` directory to exist in the system.

## glibc

Modify the spec file and comment out some statements about libmvec.

Note that `echo "slibdir=/tools/lib64" >>configparms`
this statement is no longer needed; it has been replaced by `libc_cv_slibdir`.

In glibc 2.39, crypt support has been removed, so libxcrypt needs to be compiled additionally.

The upstream Glibc compiles normally, but the glibc 2.39 in Rocky Linux does not compile normally.

After confirmation, the glibc-2.39.tar.xz in the glibc 2.39 rpm package is completely identical to upstream, but the glibc in Rocky Linux has 420 patches applied, so the problem must come from one or more of them. As to exactly which patch brings about the problem, try to use binary search to confirm.

|  Step  |   Operation        |         Effect|     Conclusion|
|--------|---------------|-------------|---------|
| 1      |  Comment out 211–420   |      OK    |   |
| 2      |  Comment out 306–420    |     Fail |    The problematic patch is in 211~306|
| 3       |  Comment out 250–420  |       OK    |   The problematic patch is in 250~306|
| 4      |  Comment out 276–420    |     Fail  |   The problematic patch is in 250~276|
| 5      |  Comment out 264–420  |       Fail  |   The problematic patch is in 250~263|
| 6      |  Comment out 257–420     |    OK   |    The problematic patch is in 257~263|
| 7       | Comment out 260–420   |      OK   |    The problematic patch is in 261~263|
| 8     |   Comment out 262–420   |      Fail   |  The problematic patch is in 260~261|
| 9     |   Comment out 261–420     |    OK   |    The problematic patch is 261|

Okay, the problematic patch `glibc-RHEL-101754-2.patch` has been found!

According to the content of the patch, LoongArch may need to add a `dl-tunables.list` file.

The solution is not to apply Patch 261. We need to take a look at what this patch does.

After compilation is complete, check: the libc library is located in /tools/lib64; inspect it with the file command.

```bash
file /tools/lib64/libc.so.6
/tools/lib64/libc.so.6: ELF 64-bit LSB shared object, LoongArch, version 1 (GNU/Linux), dynamically linked, interpreter /tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with debug_info, not stripped
```

In `/tools/lib64`, there are also files such as `ld-linux-loongarch-lp64d.so.1`, `librt.so`,
`crt1.o`, `crti.o`, and `crtn.o`.

Note: At this point, locales can be compiled, otherwise zh_CN.utf8 cannot be set as the locale later.

## GCC-PASS2

The second compilation of GCC. During compilation, I encountered this problem:
```text
/home/rocky/rpmbuild/BUILD/gcc-14.3.1-build/gcc-14.3.1-20250617/build/prev-loongarch64-unknown-linux-gnu/libstdc++-v3/include/ext/concurrence.h:252:32: error: cannot convert '<brace-enclosed initializer list>' to 'unsigned int' in initialization
  252 |     __gthread_cond_t _M_cond = __GTHREAD_COND_INIT;
      |
```

After a series of searches, I found that others had encountered a similar problem, and the solution is:

<https://sourceware.org/bugzilla/show_bug.cgi?id=32625>

This bug points to the next bug:

<https://sourceware.org/bugzilla/show_bug.cgi?id=32621>

The solution is here:

<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=118009>

<https://gcc.gnu.org/cgit/gcc/diff/?id=ea2798892de373b14f9fc7ae8a0d820eaddca98c>

Download the gcc source code, find this commit, make it into a patch, and try to apply it to this version of gcc.
```bash
git format-patch -1 ea2798892de373b14f9fc7ae8a0d820eaddca98c
```
Produce the patch file: `0001-fixincludes-Skip-pthread_incomplete_struct_argument-.patch`

Copy it to `~/rpmbuild/SOURCE`

Modify the gcc spec file and add the patch file to the patches.

Here is the specific [build script](scripts/cross/build-cross-gcc-pass2.sh)

## Checking the Toolchain

Write a test program:

```c
#include <stdio.h>
int main() {
  printf("OK!\n");
  return 0;
}
```

Compile
```bash
loongarch64-unknown-linux-gnu-gcc test-gcc-pass2.c -v -Wl,--verbose &> dummy.log
```

Run: `file a.out`

Output
```text
a.out: ELF 64-bit LSB pie executable, LoongArch, version 1 (SYSV),
dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
Run: ` readelf -l a.out |grep "tools"`

Output
```text
[Requesting program interpreter: /tools/lib64/ld-linux-loongarch-lp64d.so.1]
```
Run `grep -o "/tools/lib.*/crt.*succeeded" dummy.log`

Output
```text
/tools/lib/../lib64/crti.o succeeded
/tools/lib/../lib64/crtn.o succeeded
```

File search path:
```text
grep -B1 "${SYSDIR}/.*/include" dummy.log
GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
ignoring duplicate directory "/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/../../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/include"
ignoring duplicate directory "/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/../../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/include-fixed"
ignoring duplicate directory "/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/../../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/include"
ignoring duplicate directory "/home/rocky/rocky_loong64/tools/include"
--
#include <...> search starts here:
 /home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/include
 /home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/include-fixed
 /home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/include
 /home/rocky/rocky_loong64/tools/include
```

Whether the paths of the linked library files are correct:
```text
grep "SEARCH_DIR" dummy.log
SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib");
```
Whether the C library location is correct:
```text
grep "libc.so.6" dummy.log
attempt to open /home/rocky/rocky_loong64/tools/lib64/libc.so.6 succeeded
/home/rocky/rocky_loong64/tools/lib64/libc.so.6
ld-linux-loongarch-lp64d.so.1 needed by /home/rocky/rocky_loong64/tools/lib64/libc.so.6
```

Whether the ld library location is correct:
```text
grep found dummy.log
found ld-linux-loongarch-lp64d.so.1 at /home/rocky/rocky_loong64/tools/lib64/ld-linux-loongarch-lp64d.so.1
```

**Warning**: All the above checks must be done; they must not be skipped, and they must not be taken lightly! If anything does not match up, it must be fixed, otherwise all the programs compiled later will have problems and **everything** will have to be redone. Don't ask me how I know!

**Remember! Remember! Remember**

## pkgconf

Build normally.

Check:
```bash
type pkg-config
pkg-config is /cross-tools/bin/pkg-config
```

## grub

The grub version bundled in Rocky Linux is 2.12, whose support for the LoongArch architecture is not yet complete enough, so I switched to the grub 2.14 source code for compilation. However, after testing the result is still quite bad:
```text
============================================================================
Testsuite summary for GRUB 2.14
============================================================================

# TOTAL: 90
# PASS:  20
# SKIP:  9
# XFAIL: 0
# FAIL:  38
# XPASS: 0
# ERROR: 23
============================================================================
See ./test-suite.log
Please report to bug-grub@gnu.org
```
What specific problems there are is unknown, and I don't have more time to analyze it. However, it seems this grub is still usable.
At this point, the compilation of the entire toolchain is complete, and the first stage is over!

# Building the Temporary System

Moving on to the next part, building the temporary system. This part is very close to LFS, except that the source of the source code is restricted to Rocky Linux source code.

## Setting Up the Build Environment

Update .bashrc and create meson-cross.txt. You can refer to this [script](scripts/tmp/prepare-tmp-os.sh)

Next, complete the building of the temporary system toolchain. The build script for each software package is in the [script directory](scripts/tmp/).

## gmp

Check: In the /tools/lib64 directory, the files libgmp.a and libgmp.so exist.

## MPFR

Check: In the /tools/lib64 directory, the files libmpfr.so and libmpfr.a exist

## MPC

Check: In the /tools/lib64 directory, the files libmpc.so and libmpc.a exist

## ISL

ISL is part of gcc.

Check: In the /tools/lib64 directory, the files libisl.so and libisl.a exist

## CLooG

New versions of GCC no longer need CLooG.

## Zlib

zlib needs to be downloaded from here

<https://mirror.nju.edu.cn/rocky/9/devel/source/tree/Packages/z/zip-3.0-35.el9.src.rpm>

Check: In the /tools/lib64 directory, the files libz.so and libz.a exist

## Binutils

Check: In the /tools/bin directory, programs such as ld, nm, as, and ld-new exist.

Check the interpreter
```bash
file /tools/bin/ld
```

Output
```text
/tools/bin/ld: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
Check the search path:
```bash
tools/bin/ld --verbose|grep -i search
```
Output
```text
SEARCH_DIR("/tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("/tools/loongarch64-unknown-linux-gnu/lib");
```

## GCC

An important component.

Check: In /tools/bin/, files such as gcc, g++, gfortran, and cpp exist.

Check the interpreter
```bash
file /tools/bin/gcc
```

Output:
```text
/tools/bin/gcc: ELF 64-bit LSB executable, LoongArch, version 1
(GNU/Linux), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```

## ncurses

Before compiling, modify the SPEC file and comment out this line:
```bash
%{gpgverify} --keyring=%{SOURCE2} --signature=%{SOURCE1} --data=%{SOURCE0}
```
The `gpgverify` macro has not been implemented for now; skipping it is fine.

Check:

In `/tools/lib64/`, library files such as libncurses.so, libncursesw.a,
libncursesw.so exist.

In the `/tools/lib64/pkgconfig` directory, the files ncurses++w.pc, ncursesw.pc, and ncurses.pc exist.

## BZIP2

Check: In `/tools/bin`, files such as bzip2, bunzip2, bcat, bzmore, bzless,
bzgrep exist.

In `/tools/lib64`, libbz2.so exists.

## XZ

Check:
```bash
ls /tools/bin/xz* /tools/lib64/liblzma.*
```
Output:
```text
/tools/bin/xz /tools/bin/xzcmp /tools/bin/xzdiff /tools/bin/xzfgrep
/tools/bin/xzless /tools/bin/xzcat /tools/bin/xzdec /tools/bin/xzegrep
/tools/bin/xzgrep /tools/bin/xzmore
/tools/lib64/liblzma.a /tools/lib64/liblzma.la /tools/lib64/liblzma.so
/tools/lib64/liblzma.so.5 /tools/lib64/liblzma.so.5.6.2
```

## Readline

Check:
```bash
ls /tools/lib64/libreadline.* /tools/lib64/libhistory.*
```
Output:
```text
/tools/lib64/libhistory.a /tools/lib64/libhistory.so.8
/tools/lib64/libreadline.a /tools/lib64/libreadline.so.8
/tools/lib64/libhistory.so /tools/lib64/libhistory.so.8.2
/tools/lib64/libreadline.so /tools/lib64/libreadline.so.8.2
```

## OpenSSL

When compiling openssl, I encountered this bug

<https://www.findbugzero.com/operational-defect-database/vendors/rh/defects/RHEL-93168>

It looks related to certain macro definitions. For now, comment out some code, and compilation can proceed normally. As for what the underlying principle is, I'll think about it later.

It seems related to this patch: `0014-RH-Export-two-symbols-for-OPENSSL_str-n-casecmp.patch`

It is not that simple. The final resort: I wrote sed statements to comment out part of the code

Check:
```bash
ls /tools/bin/openssl
```
Output:
```text
/tools/bin/openssl
```
Check:

```bash
file /tools/bin/openssl
```

Output:
```text
/tools/bin/openssl: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
Check:
```bash
ls /tools/lib64/libssl.*
```
Output:
```text
/tools/lib64/libssl.a /tools/lib64/libssl.so /tools/lib64/libssl.so.
```

## pcre2

Comment out the gpgverify line in the spec file.

Check:
```bash
ls /tools/lib64/libpcre2-*
```
Output:
```text
/tools/lib64/libpcre2-8.a /tools/lib64/libpcre2-8.so.0
/tools/lib64/libpcre2-posix.la /tools/lib64/libpcre2-posix.so.3.0.5
/tools/lib64/libpcre2-8.la /tools/lib64/libpcre2-8.so.0.13.0
/tools/lib64/libpcre2-posix.so
/tools/lib64/libpcre2-8.so /tools/lib64/libpcre2-posix.a
/tools/lib64/libpcre2-posix.so.3
```

## libsepol

Check:
```bash
ls /tools/lib64/libsepol.*
```
Output:
```text
/tools/lib64/libsepol.a /tools/lib64/libsepol.so
/tools/lib64/libsepol.so.2
```

## libselinux

`PKG_CONFIG_PATH=/tools/lib64/pkgconfig USE_PCRE2=y` must be set

Otherwise pcre2-8 will not be linked automatically.

Check:
```bash
ls /tools/lib64/libselinux.*
```

Output:
```text
/tools/lib64/libselinux.a /tools/lib64/libselinux.so
/tools/lib64/libselinux.so.1
```

## nspr

The nspr source code is in nss.

Check:
```bash
ls /tools/lib64/{libnspr4.*,libplc*}
```
Output:
```text
/tools/lib64/libnspr4.a /tools/lib64/libnspr4.so /tools/lib64/libplc4.a
/tools/lib64/libplc4.so
```

## sqlite

Check:
```bash
ls /tools/lib64/libsqlite3.*
```
Output:
```text
/tools/lib64/libsqlite3.a /tools/lib64/libsqlite3.la
/tools/lib64/libsqlite3.so /tools/lib64/libsqlite3.so.0
/tools/lib64/libsqlite3.so.0.8.6
```

## nss

Need to add `NSS_DISABLE_DSA=1 NSS_ENABLE_ML_DSA=1`

Check:
```bash
ls /tools/lib64/{libnss*,libsoft*}
```
Output:
```text
/tools/lib64/libnss3.so /tools/lib64/libnss_db.so.2
/tools/lib64/libnss_hesiod.so.2 /tools/lib64/libnsssysinit.so
/tools/lib64/libnss_compat.so /tools/lib64/libnss_dns.so.2
/tools/lib64/libnssckbi-testlib.so /tools/lib64/libnssutil3.so
/tools/lib64/libnss_compat.so.2 /tools/lib64/libnss_files.so.2
/tools/lib64/libnssckbi.so /tools/lib64/libsoftokn3.so
/tools/lib64/libnss_db.so /tools/lib64/libnss_hesiod.so
/tools/lib64/libnssdbm3.so
```

## popt

Check:
```bash
ls /tools/lib64/libpopt.*
```

Output:
```text
/tools/lib64/libpopt.a /tools/lib64/libpopt.la /tools/lib64/libpopt.so
/tools/lib64/libpopt.so.0 /tools/lib64/libpopt.so.0.0.2
```

## libarchive

The aclocal.m4 and configure scripts need to be modified, because the aclocal version in the system is 1.18, not 1.16.

The method I used was to write this modification into the build script. You can also try updating the script with `autoconf -fiv`.

Check:
```bash
ls /tools/lib64/libarchive.*
```

Output:
```text
/tools/lib64/libarchive.a /tools/lib64/libarchive.so
/tools/lib64/libarchive.so.13.7.7

/tools/lib64/libarchive.la /tools/lib64/libarchive.so.13
```

## libdb

rocky 10 no longer provides the libdb package. No longer needed.

## libcap

Comment out the statements related to gpgverify.

Check:
```bash
ls /tools/lib64/libcap.*
```

Output:
```text
/tools/lib64/libcap.a /tools/lib64/libcap.so /tools/lib64/libcap.so.2
/tools/lib64/libcap.so.2.69
```

## libmicrohttpd

Check:
```bash
ls /tools/lib64/libmicrohttpd.*
```

Output:
```text
/tools/lib64/libmicrohttpd.a /tools/lib64/libmicrohttpd.so
/tools/lib64/libmicrohttpd.so.12.62.0
/tools/lib64/libmicrohttpd.la /tools/lib64/libmicrohttpd.so.12
```

## libpsl

curl depends on libpsl, so psl needs to be built first.

Commented out the statements related to %py3_shebang_fix in the spec file

Check:
```bash
ls /tools/lib64/libpsl.*
```
Output:
```text
/tools/lib64/libpsl.a /tools/lib64/libpsl.la /tools/lib64/libpsl.so
/tools/lib64/libpsl.so.5 /tools/lib64/libpsl.so.5.3.5
```

## curl

Comment out the statements related to %{gpgverify}

Check
```bash
file /tools/bin/curl
```

Output:
```text
/tools/bin/curl: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
```bash
ls /tools/lib64/libcurl.*
```

Output:
```text
/tools/lib64/libcurl.a /tools/lib64/libcurl.la /tools/lib64/libcurl.so
/tools/lib64/libcurl.so.4 /tools/lib64/libcurl.so.4.8.0
```

## elfutils

Done.

Check:
```bash
ls /tools/lib64/{libelf*,libdw*}
```

Output:
```text
/tools/lib64/libdw-0.193.so /tools/lib64/libdw.so
/tools/lib64/libelf-0.193.so /tools/lib64/libelf.so
/tools/lib64/libdw.a /tools/lib64/libdw.so.1 /tools/lib64/libelf.a
/tools/lib64/libelf.so.1
```

## lz4

Done.

Check:
```bash
ls /tools/bin/lz4* /tools/lib64/liblz4*
```

Output:
```text
/tools/bin/lz4 /tools/bin/lz4cat /tools/lib64/liblz4.so
/tools/lib64/liblz4.so.1.9.4
/tools/bin/lz4c /tools/lib64/liblz4.a /tools/lib64/liblz4.so.1
```

## zstd

Modify the spec file, change %patchN to %patch N

Check:
```bash
ls /tools/bin/zstd* /tools/lib64/libzstd.*
```

Output:
```text
/tools/bin/zstd /tools/bin/zstdgrep /tools/bin/zstdmt
/tools/lib64/libzstd.so /tools/lib64/libzstd.so.1.5.5
/tools/bin/zstdcat /tools/bin/zstdless /tools/lib64/libzstd.a
/tools/lib64/libzstd.so.1
```

## expat

Comment out the statements related to %{gpgverify}

Add `--without-docbook` when running configure

An error will occur during the build. Some way should be found to skip doc

Modify Makefile.in

Check:
```bash
ls /tools/lib64/libexpat.*
```

Output:
```text
/tools/lib64/libexpat.a /tools/lib64/libexpat.la
/tools/lib64/libexpat.so /tools/lib64/libexpat.so.1
/tools/lib64/libexpat.so.1.10.2
```

## bash

Done.

Check:
```bash
 ls /tools/bin/bash
```

Output:
```text
/tools/bin/bash
```

## bash-complete

Done.

## coreutils

Done.

Check:
```bash
ls /tools/bin/{ls,cp,hostname,wc,tail,uniq}
```

Output:
```text
/tools/bin/cp /tools/bin/hostname /tools/bin/ls /tools/bin/tail
/tools/bin/uniq /tools/bin/wc
```

## file

Once again encountered the gpgverify problem. Preparing to solve the problem once and for all.

Install the `redhat-rpm-config-293` source package, put the gpgverify script in
`/usr/lib/rpm/redhat`, and put `macros.fedora-misc` in `/usr/lib/rpm/macros.d`.

The gpgverify script needs gpgv2; simply make a symbolic link:
```bash
ln -s gpgv gpgv2
```

Check:
```bash
ls /tools/bin/file
```

Output:
```text
/tools/bin/file
```

## findutils

Done.

Check:
```bash
ls /tools/bin/find
```

Output:
```text
/tools/bin/find
```

## gawk

The automake version needs to be set.

Done.

Check:
```bash
ls /tools/bin/gawk
```

Output:
```text
/tools/bin/gawk
```

## gettext

Only need to add
```bash
sed -e 's/\(gl_cv_libxml_force_included=\)no/\1yes/' -i libtextstyle/configure
```
and that's it. Done.

Check:
```bash
ls /tools/bin/{gettext,msg*}
```

Output:
```text
/tools/bin/gettext /tools/bin/msgcmp /tools/bin/msgen /tools/bin/msgfmt
/tools/bin/msgmerge
/tools/bin/msgattrib /tools/bin/msgcomm /tools/bin/msgexec
/tools/bin/msggrep /tools/bin/msgunfmt
/tools/bin/msgcat /tools/bin/msgconv /tools/bin/msgfilter
/tools/bin/msginit /tools/bin/msguniq
```

## grep

The automake version needs to be set

Done.

Check:
```bash
ls /tools/bin/grep
```

Output:
```text
/tools/bin/grep
```

## gzip

Done.

Check:
```bash
ls /tools/bin/gz*
```

Output:
```text
/tools/bin/gzexe /tools/bin/gzip
```

## sed

Done.

Check:
```bash
ls /tools/bin/sed
```

Output:
```text
/tools/bin/sed
```

## util-linux

Set PKG_CONFIG_PATH, run autoreconf -fv

An error will occur during the build, so --disable-schedutils was added
Not sure whether this will have any bad effects.

Done.

Check:
```bash
ls /tools/bin/{mount,umou*}
```

Output:
```text
/tools/bin/mount /tools/bin/umount
```

## kmod

Need to set PKG_CONFIG_PATH, and run autoreconf -fv

Done.

Check:
```bash
ls /tools/bin/*mod
```

Output:
```text
/tools/bin/chmod /tools/bin/depmod /tools/bin/insmod /tools/bin/kmod
/tools/bin/lsmod /tools/bin/rmmod
```

## vim

Done. Referenced the book and the CLFS documentation.

One small difference: I set `set mouse=a`

Check:
```bash
ls /tools/bin/vi*
```

Output:
```text
/tools/bin/vi /tools/bin/view /tools/bin/vim /tools/bin/vimdiff
/tools/bin/vimtutor
```

## nano

Although the build passed, I always feel it is not quite reasonable.

When running configure, some parameters were added: `LIBS="-lncurses -ltinfo" LDFLAGS="-L/tools/lib64"`
Check:
```bash
ls /tools/bin/nano
```

Output:
```text
/tools/bin/nano
```

## which

Done.

Check:
```bash
ls /tools/bin/which
```

Output:
```text
/tools/bin/which
```

## iproute

Set `LIBDIR=/tools/lib64/`

Done.

Check:
```bash
ls /tools/sbin/{ifstat,ip}
```

Output:
```text
/tools/sbin/ifstat /tools/sbin/ip
```

## dhcpcd

Done.

Check:
```bash
ls /tools/sbin/dhcpcd
```

Output:
```text
/tools/sbin/dhcpcd
```

## fipscheck

In Rocky Linux 10, the fipscheck package is now provided by libkcapi-1.5.0-3.el10.src.rpm.

## libxcrypt

Both tcsh and pam need libcrypt; use the replacement from libxcrypt. Done.

Note, `--libdir` needs to be set to `/tools/lib64`

Check:
```bash
ls /tools/lib64/libcrypt.*
```

Output:
```text
/tools/lib64/libcrypt.a /tools/lib64/libcrypt.la
/tools/lib64/libcrypt.so /tools/lib64/libcrypt.so.1
/tools/lib64/libcrypt.so.1.1.0
```
Note: libcrypt.so and libcrypto.so are two completely different things!

## pam

After installation, the `pam_appl.h` file is not in the security directory. includedir needs to be adjusted.

Key point: `--includedir=/tools/include/security`

Done.

Check:
```bash
ls /tools/lib64/libpam*
```

Output:
```text
/tools/lib64/libpam.la /tools/lib64/libpam_misc.la
/tools/lib64/libpamc.la
/tools/lib64/libpam.so /tools/lib64/libpam_misc.so
/tools/lib64/libpamc.so
/tools/lib64/libpam.so.0 /tools/lib64/libpam_misc.so.0
/tools/lib64/libpamc.so.0
/tools/lib64/libpam.so.0.85.1 /tools/lib64/libpam_misc.so.0.82.1
/tools/lib64/libpamc.so.0.82.1
```

**Note**:

After installing pam, a series of settings need to be made in /tools/etc/pam.d, otherwise the system will fail to boot. This step can be performed when configuring the temporary system.

## openssh

Missing the `security/pam_appl.h` file. An additional program, pam, needs to be installed

Added the `--with-pam` parameter, Done

Check:
```bash
ls /tools/bin/ssh* /tools/sbin/ssh*
```

Output:
```text
/tools/bin/ssh /tools/bin/ssh-agent /tools/bin/ssh-keyscan
/tools/bin/ssh-add /tools/bin/ssh-keygen /tools/sbin/sshd
```

## sudo

Done
```bash
ls /tools/bin/sudo
```

Output:
```text
/tools/bin/sudo
```

## e2fsprogs

Add `LDFLAGS="-L/tools/lib64 -luuid -lblkid"`

Done
```bash
ls /tools/sbin/{mkfs.*,fsck.*}
```

Output:
```text
/tools/sbin/fsck.cramfs /tools/sbin/fsck.ext4 /tools/sbin/mkfs.cramfs
/tools/sbin/mkfs.ext4
/tools/sbin/fsck.ext2 /tools/sbin/fsck.minix /tools/sbin/mkfs.ext2
/tools/sbin/mkfs.minix
/tools/sbin/fsck.ext3 /tools/sbin/mkfs.bfs /tools/sbin/mkfs.ext3
```

## inih

Following the scripts from <https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md>

, build and install

Check:
```bash
ls /tools/lib64/libinih.so*
```

Output:
```text
/tools/lib64/libinih.so /tools/lib64/libinih.so.0
```

## uuid

Need to update the `config.guess` and `config.sub` files

Check:
```bash
ls /tools/lib64/libossp-uuid.*
```

Output:
```text
/tools/lib64/libossp-uuid.a /tools/lib64/libossp-uuid.so
/tools/lib64/libossp-uuid.so.16.0.22
/tools/lib64/libossp-uuid.la /tools/lib64/libossp-uuid.so.16
```

## userspace-rcu

For version 0.14.0, a patch adding LoongArch support needs to be added:

<https://gitee.com/zhong-wei-shen/userspace-rcu/blob/master/add-loongarch.patch>

automake and other versions don't match; need `autoreconf -fv`

Done.

Check:
```bash
ls /tools/lib64/liburcu*
```

Output:
```text
/tools/lib64/liburcu-bp.a /tools/lib64/liburcu-common.so.8.1.0
/tools/lib64/liburcu-qsbr.so.8
/tools/lib64/liburcu-bp.la /tools/lib64/liburcu-mb.a
/tools/lib64/liburcu-qsbr.so.8.1.0
/tools/lib64/liburcu-bp.so /tools/lib64/liburcu-mb.la
/tools/lib64/liburcu-signal.a
/tools/lib64/liburcu-bp.so.8 /tools/lib64/liburcu-mb.so
/tools/lib64/liburcu-signal.la
/tools/lib64/liburcu-bp.so.8.1.0 /tools/lib64/liburcu-mb.so.8
/tools/lib64/liburcu-signal.so
/tools/lib64/liburcu-cds.a /tools/lib64/liburcu-mb.so.8.1.0
/tools/lib64/liburcu-signal.so.8
/tools/lib64/liburcu-cds.la /tools/lib64/liburcu-memb.a
/tools/lib64/liburcu-signal.so.8.1.0
/tools/lib64/liburcu-cds.so /tools/lib64/liburcu-memb.la
/tools/lib64/liburcu.a
/tools/lib64/liburcu-cds.so.8 /tools/lib64/liburcu-memb.so
/tools/lib64/liburcu.la
/tools/lib64/liburcu-cds.so.8.1.0 /tools/lib64/liburcu-memb.so.8
/tools/lib64/liburcu.so
/tools/lib64/liburcu-common.a /tools/lib64/liburcu-memb.so.8.1.0
/tools/lib64/liburcu.so.8
/tools/lib64/liburcu-common.la /tools/lib64/liburcu-qsbr.a
/tools/lib64/liburcu.so.8.1.0
/tools/lib64/liburcu-common.so /tools/lib64/liburcu-qsbr.la
/tools/lib64/liburcu-common.so.8 /tools/lib64/liburcu-qsbr.so
```

## xfsprogs

ini.h was used, so the libinih development package needs to be installed; uuid.h was used, so uuid needs to be installed; urcu.h is needed, so userspace-rcu needs to be installed

**Note**: the uuid used by xfsprogs is provided by util-linux, not ossp-uuid.

Done

Check:

```bash
ls /tools/sbin/*xfs
```

Output:
```text
/tools/sbin/fsck.xfs /tools/sbin/mkfs.xfs
```

## dosfstools

Done

Check:
```bash
ls /tools/sbin/*vfat
```

Output:
```text
/tools/sbin/fsck.vfat /tools/sbin/mkfs.vfat
```

## bison

Done

Check:
```bash
ls /tools/bin/{bison,yacc}
```

Output:
```text
/tools/bin/bison /tools/bin/yacc
```

## check

When running configure, add the `--disable-build-docs` option.

Build normally.

Check:
```bash
ls /tools/bin/checkmk
```

Output:
```text
/tools/bin/checkmk
```

## diffutils

Done.

Check:
```bash
ls /tools/bin/diff*
```

Output:
```text
/tools/bin/diff /tools/bin/diff3
```

## make

Done.
```bash
ls /tools/bin/make
```

Output:
```text
/tools/bin/make
```

## patch

`config.sub` and `config.guess` need to be updated. Done.

Check:
```bash
ls /tools/bin/patch
```

Output:
```text
/tools/bin/patch
```

## tar

Done.

Check:
```bash
ls /tools/bin/tar
```

Output:
```text
/tools/bin/tar
```

## textinfo

Done. The Makefile.am file didn't even need to be modified.
Check:
```bash
ls /tools/bin/info
```

Output:
```text
/tools/bin/info
```

## m4

Done.
```bash
ls /tools/bin/m4
```

Output:
```text
/tools/bin/m4
```

## pkgconf

Done.

Check:
```bash
ls /tools/bin/pkg-config
```

Output:
```text
/tools/bin/pkg-config
```

## autoconf

Done. Check:
```bash
ls /tools/bin/autoconf
```

Output:
```text
/tools/bin/autoconf
```

## automake

Done.

Check:
```
ls /tools/bin/automake
```

Output:
```text
/tools/bin/automake
```

## libtool

Done.
```bash
ls /tools/bin/libtool
```

Output:
```text
/tools/bin/libtool
```

## flex

Done.
Check:
```bash
ls /tools/bin/flex
```

Output:
```text
/tools/bin/flex
```

## tcl

Done.

Check:
```bash
ls /tools/bin/tclsh /tools/lib64/libtcl*
```

Output:
```text
/tools/bin/tclsh /tools/lib64/libtcl.so /tools/lib64/libtcl8.6.so
/tools/lib64/libtclstub8.6.a
```

## lua

Done.

Check:

```bash
ls /tools/bin/lua
```

Output:
```text
/tools/bin/lua
```

## cpio

Done.

Check:
```bash
ls /tools/bin/cpio
```

Output:
```text
/tools/bin/cpio
```

## tcsh

Build error encountered: `undefined reference to 'crypt'`

After installing libxcrypt, then install tcsh, and it's done.
Check:
```bash
ls /tools/bin/tcsh
```

Output:
```text
/tools/bin/tcsh
```

## attr

acl depends on attr. Done.
Check:
```bash
ls /tools/bin/attr
```

Output:
```text
/tools/bin/attr
```

## audit

You need to write a build script based on the spec file, removing some features. Done.

Check:
```bash
ls /tools/lib64/libaudit.*
```

Output:
```text
/tools/lib64/libaudit.a /tools/lib64/libaudit.so
/tools/lib64/libaudit.so.1.0.0
/tools/lib64/libaudit.la /tools/lib64/libaudit.so.1
```

## acl

Installing rpm requires acl.

Done. Check:
```bash
ls /tools/lib64/libacl.*
```

Output:
```text
/tools/lib64/libacl.la /tools/lib64/libacl.so /tools/lib64/libacl.so.1
/tools/lib64/libacl.so.1.1.2302
```

## nettle

You need `autoreconf -ifv`, and when running configure, remove sm3 and sm4 support.

Done:
```bash
ls /tools/lib64/libnettle.*
```

Output:
```text
/tools/lib64/libnettle.a /tools/lib64/libnettle.so
/tools/lib64/libnettle.so.8 /tools/lib64/libnettle.so.8.10
```

## rpm

rpm 4.15 uses configure to generate the Makefile, but 4.19 switched to cmake. Therefore, the cross-compilation approach needs to change.

You need to create [loongarch64-toolchain.cmake](sources/loongarch64-toolchain.cmake).

Modify the build script to use cmake for compilation.

You need to set the PKG_CONFIG_PATH directory.

You need to compile and install libacl and audit.

Remove support for dbus.

Remove some of the changes in the patch rpm-4.19.x-pqc-algo.patch.

Remove support for python.

Done.

Check:
```bash
ls /tools/bin/rpm*
```

Output:
```text
/tools/bin/rpm /tools/bin/rpmbuild /tools/bin/rpmkeys /tools/bin/rpmsign
/tools/bin/rpmverify
/tools/bin/rpm2archive /tools/bin/rpmdb /tools/bin/rpmlua
/tools/bin/rpmsort
/tools/bin/rpm2cpio /tools/bin/rpmgraph /tools/bin/rpmquery
/tools/bin/rpmspec
```

## Configure rpm

Refer to the [Configuration script](scripts/tmp/config-tmp-rpm.sh).
Done

## systemd

Follow CLFS and the Leopard book. You need to set PKG_CONFIG_PATH.

You need to set version_no_tilde to 257
```bash
rpmbuild -D "version_no_tilde 257" -bp ~/rpmbuild/SPECS/systemd.spec --nodeps
```
Done

Check:
```bash
ls /tools/sbin/init /tools/bin/systemctl
```

Output:
```text
/tools/bin/systemctl /tools/sbin/init
```

## dbus

You need to set the PKG_CONFIG_PATH environment variable, and do not install the docs.

Add `autoreconf -fv`

Check
```bash
ls /tools/bin/dbus-daemon
```

Output:
```text
/tools/bin/dbus-daemon
```

## shadow-utils

Done
```bash
ls /tools/bin/login
```

Output:
```text
/tools/bin/login
```

## Linux kernel

Use Debian's 6.17.11 kernel.

When compiling the kernel, you need to make sure the necessary modules are all built in, so that initrd.img is no longer needed.

This includes the ext4 file system, the xfs file system,

The kernel modules that need to be built in include

- CONFIG_VIRTIO_BLK=y

- CONFIG_SCSI_VIRTIO=y

- CONFIG_HW_RANDOM_VIRTIO=y

- CONFIG_DRM_VIRTIO_GPU=y

    In the `make menuconfig` interface, make good use of "/" to search for modules.

  The final configuration file used is [here](sources/config-6.17.11).

## grub

Done.
The first attempt used grub 2.12

Check:
```bash
ls /tools/bin/grub-*
```

Output:
```text
/tools/bin/grub-editenv   /tools/bin/grub-kbdcomp      /tools/bin/grub-mknetdir		/tools/bin/grub-mkstandalone
/tools/bin/grub-file	  /tools/bin/grub-menulst2cfg  /tools/bin/grub-mkpasswd-pbkdf2	/tools/bin/grub-render-label
/tools/bin/grub-fstest	  /tools/bin/grub-mkimage      /tools/bin/grub-mkrelpath	/tools/bin/grub-script-check
/tools/bin/grub-glue-efi  /tools/bin/grub-mklayout     /tools/bin/grub-mkrescue		/tools/bin/grub-syslinux2cfg
```

Note, after compilation is complete, you should run make check to verify whether the compilation is correct. The actual test results were disastrous.
```text
Testsuite summary for GRUB 2.12
============================================================================

# TOTAL: 87
# PASS:  15
# SKIP:  8
# XFAIL: 0
# FAIL:  43
# XPASS: 0
# ERROR: 21
```

So we switched to upstream [grub 2.14](https://ftp.gnu.org/gnu/grub/grub-2.14.tar.xz) instead. Compile following Sun Haiyong's script.

The test results were still pretty grim:
```text

Testsuite summary for GRUB 2.14
============================================================================

# TOTAL: 90
# PASS:  20
# SKIP:  9
# XFAIL: 0
# FAIL:  38
# XPASS: 0
# ERROR: 23
```
Let's leave it like this for now; it doesn't seem to affect usage.

## wget

Done.

Check:
```bash
ls /tools/bin/wget
```

Output:
```text
/tools/bin/wget
```

## openjdk

You can download the LoongArch version of openjdk from here to solve the self-dependency problem.

<https://www.loongnix.cn/zh/api/java>

## rust

You need to modify the spec file, adding loongarch64 to rust_arches.

You need to set PKG_CONFIG_PATH.

Build error encountered during compilation:
```text
make: *** [Makefile:19: all] Error 1
Building bootstrap
   Compiling bootstrap v0.0.0 (/home/rocky/rpmbuild/BUILD/rust-1.88.0-build/rustc-1.88.0-src/src/bootstrap)
error: trait `FormatShortCmd` is never used
   --> src/bootstrap/src/utils/exec.rs:335:11
    |
335 | pub trait FormatShortCmd {
    |           ^^^^^^^^^^^^^^
    |
    = note: `-D dead-code` implied by `-D warnings`
    = help: to override `-D warnings` add `#[allow(dead_code)]`

error: could not compile `bootstrap` (lib) due to 1 previous error
```
Adding `export RUSTFLAGS="-A dead_code"` works around the above problem.

To compile rust 1.88, you need to use rust 1.88/1.87! There is also a self-dependency problem here.

Download and install rust 1.88

<https://static.rust-lang.org/dist/rust-1.88.0-loongarch64-unknown-linux-gnu.tar.gz>

After downloading and extracting, run
```bash
./install.sh --prefix=~/build/rust/
```

In fact, installing rust can be done later, since there are already ready-made rust installation packages that can solve the self-dependency problem during rpm package compilation.

The src/llvm-prject/compiler-rt directory is missing; llvm-project is a submodule that rust depends on.

Download the code and try to complete it:

<https://github.com/rust-lang/llvm-project/archive/refs/tags/rustc-1.88.0.tar.gz>

Refer to the spec file, modify the build script, and set llvm-root. (You need to confirm which modifications are necessary and which are superfluous.)

In the end, I decided not to do it, because compiling now requires supplementing many packages, such as llvm, and the cost is too high, so I'll compile it later.

Not completed.

Refer to this LFS Chinese document <https://lfs.xry111.site/zh_CN/12.4-systemd>

<https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md>

## dracut

Not completed. It doesn't affect usage, so I won't compile it for now.

# Configuring the Temporary System

After completing the compilation of the above software packages, you can configure the temporary system, which can already meet the boot requirements.

The configuration includes: copying files, creating basic directories, creating necessary symbolic links, etc.

You can refer to this configuration [script](scripts/tmp/create-tmp-os.sh).

There are two ways to use the temporary system: one is to boot it as a container, and the other is to make it directly into a virtual machine for use. The former has a lower barrier to entry, but cannot guarantee the integrity of the system; the latter has a slightly higher barrier to use and build difficulty, but is closer to a real operating system.

## Booting the temporary system as a container

Usage
```bash
sudo systemd-nspawn -b -D ./rootfs/ --private-users=1 --bind=/run/dbus
```
Here, -b means to boot this system, which will invoke systemd.

-D means the root directory of the system.

`--private-users=1` means systemd's PID is 1,

`--bind=/run/dbus` is to allow logind.service to access dbus.

## Booting the temporary system as a virtual machine

Create a qcow2 virtual machine image.

The manual operation procedure is as follows:

- Create the qcow2 image
```bash
qemu-img create -f qcow2 rocky_vm.qcow2 50G
```

- Prepare the nbd module
```bash
sudo modprobe nbd max_part=16
```

- Attach the qcow2 file
```bash
sudo qemu-nbd -c /dev/nbd0 rocky_vm.qcow2
```

- Partition the qcow2 file. Here you can operate directly with the graphical gparted.
```bash
sudo /sbin/gparted /dev/nbd0
```
First, create a gpt partition table.

Device->Create Partition Table->gpt

Simply create two partitions: a 100MiB FAT32 EFI partition, and use the rest of the space as the root partition, without separately partitioning boot and other partitions.

At this point, you will find that nbd0 already has two partitions, p1 and p2.

```bash
ls /dev/nbd0*
```
```text
/dev/nbd0 /dev/nbd0p1 /dev/nbd0p2
```

Prepare the mount point for the partition.

```bash
mkdir rocky_vm
```

Mount the second partition.
```bash
sudo mount /dev/nbd0p2 rocky_vm
```

Copy the previous rootfs into the rocky_vm folder.
```bash
sudo rsync -av rootfs/* rocky_vm/
```
Inside rocky_vm, create the efi mount point.
```bash
mkdir -pv boot/efi/
```
Mount the efi partition to /boot/efi.
```bash
sudo mount /dev/nbd0p1 boot/efi
```
Installing grub with grub-mkimage failed, and I encountered this problem:
```text
/cross-tools/bin/loongarch64-unknown-linux-gnu-grub-mkimage: error:
undefined symbol grub_arch_dl_min_alignment
```
When you encounter a problem, find a way to solve it.

In the `./grub-core/kern/loongarch64/dl.c` file, the `grub_arch_dl_min_alignment` function does not exist.

/cross-tools/bin/loongarch64-unknown-linux-gnu-grub-mkimage: error:
undefined symbol start.

Final solution: directly use the grub provided by Debian to achieve booting.

Completed the writing of the virtual machine build [script](scripts/tmp/create-tmp-vm.sh).

Use nspawn to enter the virtual machine.

On March 30, 2026, the virtual machine finally booted smoothly, and both the serial port and the spice interface worked.

## Virtual machine configuration

All subsequent porting will be done inside the virtual machine. My build machine is a laptop using a Loongson 3A6000M (2.0GHz), with only 4 cores and 8 threads. I allocated 3 processor cores to the virtual machine.

- Set up fstab

Two partitions, namely the efi partition vda1 and the root partition vda2.

- Set the boot mode
```bash
systemctl set-default multi-user
```

- Complete some network function services.

During the configuration process, I kept encountering new problems, and the content in the book was no longer enough to solve my problems. Now I have one foot in uncharted territory.

- Using the mount program.

For /etc/fstab to take effect, the mount program must be able to work properly. One of the prerequisites for the mount program to work properly is that the owner must be root.

- Kernel settings.

You need to set CONFIG_NLS_CODEPAGE_437 to y, to build the nls_cp437 module in.

Set CONFIG_NLS_ASCII to y.

Rebuild the kernel again.

- Set the root password, set to loongarch

- Create a new user. Create a user called rocky.
```bash
useradd -m -s /bin/bash -U -G wheel rocky
```
- Set the user password
```bash
passwd rocky
```

- Enable dhcpd

Obtain an IP address
```bash
ip addr
```

- Enable remote access:
```bash
ssh-keygen -A
/tools/sbin/sshd
```
Now you can remotely access the virtual machine via ssh.

- Set sudo permissions, so that an ordinary user can execute commands with sudo. Note, the sudo program must be setuid.

- For the newly created user rocky, set up .bash_profile and .bashrc.

- Using visuo, modify Defaults secure_path in the /etc/sudoers file, adding /tools/sbin,
/tools/bin to it.

Finally, I can move on to the next step!

# Completing the Temporary System

## Perl

Encountered some new problems; the `MB_LEN_MAX` macro definition is wrong. The correct value should be 16, but on my Rocky
system the value is 1. This prevents compilation from continuing.

I need to figure out why they differ.

After comparison, the `/tools/lib64/gcc/loongarch64-unknown-linux-gnu/14.3.1/include/limits.h` file really does differ from
the `/usr/lib/gcc/loongarch64-linux-gnu/14/include/limits.h` file provided by the Debian
system. Using the version provided by Debian, let's see whether the bug can be fixed.

After confirmation, the bug has been fixed. **When compiling GCC later, this patch must be added.**

Continue compiling Perl.

After compilation is complete, test:
```bash
/tools/bin/perl --version
```
Output:
```
This is perl 5, version 40, subversion 2 (v5.40.2) built for loongarch64-linux

Copyright 1987-2025, Larry Wall

Perl may be copied only under the terms of either the Artistic License or the
GNU General Public License, which may be found in the Perl 5 source kit.

Complete documentation for Perl, including FAQ lists, should be found on
this system using "man perl" or "perldoc perl".  If you have access to the
Internet, point your browser at https://www.perl.org/, the Perl Home Page.
```

## Python

Replace `config.guess` and `config.sub`

## Git

Build normally.

## Grep

The compilation of grep needs to use the rpm command. Make sure the modifications to the macros in /tools/lib/rpm are correct. This includes the replacement of /tools, and the settings for sysconfdir, localstatedir, var, sharedstatedir, etc.

There is no gpgverify, so comment out the related lines for now.

## GDB

Build normally.

## Xxhash

dwz depends on xxhash.h, so it needs to be completed first.

## DWZ

Depends on xxhash.h.

## Procps-ng

Replace `config.guess` and `config.sub`

At this point, the first backup can be made for the system.

rocky_vm-stage1.qcow2

## help2man

The perl-related modules are not installed yet, so when compiling, first `--disable-nls`

## debugedit

Depends on help2man. Comment out the gpgverify statement.

## ~~gpg2~~

~~The gpgverify script used in the rpm package will call gpg2. There are too many dependencies, so leave it alone for now.~~

## libkcapi

libxcrypt depends on fipscheck,
which is already provided by libkcapi. So, libkcapi needs to be compiled.

## python-setuptools

You cannot directly use python setup.py install, as this will not automatically call the shim, and distutils cannot be imported directly.

Finally, this approach was used:
```bash
python3 -m build --wheel
sudo pip3 install dist/setuptools-69.0.3-py3-none-any.whl
```

## redhat-rpm-config

From here on, begin compiling rpm packages.

- Modify `~/rpmbuild/SOURCES/rpmrc`, add the optflags for loongarch:

`optflags:loongarch64 %{__global_compiler_flags} -march=loongarch64 -msimd=lsx`

- Modify `~/rpmbuild/SOURCES/macros.nodejs-srpm`, add loongarch64

Modify the Release number of redhat-rpm-confg.spec

- Regenerate the source package
```bash
rpmbuild -bs ~/rpmbuild/SPECS/redhat-rpm-config.spec
```
- Compile the source package
```bash
rpmbuild --rebuild ~/rpmbuild/SRPMS/redhat-rpm-config-293-2.src.rpm --nodeps
```
Generated a new rpm package: `~/rpmbuild/RPMS/noarch/redhat-rpm-config-293-2.noarch.rpm`

- Forcibly install

- Configure redhat-rpm-config

## python-srpm-macros

 Many rpm packages require the `__os_install_post_python` macro provided by python-srpm-macros during compilation.
```bash
 rpmbuild --rebuild /opt/srpms/Packages/python-rpm-macros-3.12-10.el10.src.rpm
 ```

Compilation generated the following files:
```text
python3-rpm-macros-3.12-10.el10.loongarch.noarch.rpm
python-rpm-macros-3.12-10.el10.loongarch.noarch.rpm
python-srpm-macros-3.12-10.el10.loongarch.noarch.rpm
```
Install all:
```bash
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/python* --nodeps
```
After testing, rpm will not load macros from the `/usr/lib/rpm/macros.d` folder.

Because rpm is installed in tools, it will only load macros from /tools/lib/rpm/macros.d.

Solution: symbolically link the contents of `/usr/lib/rpm/macrod.d` into `/tools/lib/rpm/macros.d`.

## Distribution information packages

A package similar to fedora-repos cannot be found.

## rocky-release

The following modifications were made: comment out mkdir ./docs

It is not clear for now how these problems should be fixed. Leave them alone for now. The srpm package needs to be regenerated, and the rpm package compiled.

## iso-codes

Depends on the find-debuginfo program provided by debugedit.

Build normally, generating two rpm packages:
```
iso-codes-devel-4.16.0-6.el10.loongarch.noarch.rpm
iso-codes-4.16.0-6.el10.loongarch.noarch.rpm
```
Install these packages.
```bash
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/iso-codes-* --nodeps
```
No code was modified, so the srpm package does not need to be regenerated.

## setup

Dependency: `systemd-rpm-macros: required to use _tmpfilesdir macro`

So, systemd-rpm-macros needs to be installed first. This package is provided by systemd. Let's see whether these things were installed when compiling systemd.
```bash
rpmbuild -D "version_no_tilde 257" -bp ~/rpmbuild/SPECS/systemd.spec --nodeps
```

When compiling systemd, copy macros.systemd from the build folder into /tools/lib/rpm/macrod.d.
```bash
rpmbuild --bb ~/rpmbuild/SPECS/setup.spec --nodeps
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/setup-2.14.5-7.el10.loongarch.noarch.rpm
```
No changes made.

## Filesystem

There is no setting for loongarch, so it needs to be added manually.

Wherever aarch64 or riscv64 is included, add loongarch64.

Two rpm packages were generated:
```
filesystem-content-3.18-17.el10.loongarch.loongarch64.rpm
filesystem-3.18-17.el10.loongarch.loongarch64.rpm
```
The version number needs to be changed, and the srpm package regenerated.

## Basesystem

```bash
rpmbuild --rebuild /opt/srpms/Packages/basesystem-11-22.el10.src.rpm
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/basesystem-11-22.el10.loongarch.noarch.rpm
```
No changes made.

## less

Compiling this package is purely out of personal preference, and also to test whether the compiler works well.

The dependencies of the less package are relatively simple, and it is easy to compile.
```bash
rpmbuild -bb ~/rpmbuild/SPECS/less.spec --nodeps
sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/loongarch64/less-661-3.el10.loongarch.loongarch64.rpm --nodeps
```

## rsync

When directly compiling kernel header files, rsync is depended upon. Trying to compile the rpm package. Five packages were generated:
Only installing rsync-3.4.1-2.el10.loongarch.loongarch64.rpm is enough.

## Kernel header files kernel-header

There is no such package as kernel-headers. There is only the kernel-6.12 package.

Modify the kernel.spec file:

1. Add loongarch64 at the end of the line containing ExclusiveArch
2. Add support for the LoongArch architecture:

```text
%ifarch loongarch64
%define asmarch loongarch
%define hdarch loongarch
%define make_target vmlinuz.efi
%define kernel_image arch/loongarch/boot/vmlinuz.efi
%endif
```
Compilation succeeded, generating 3 files:
```text
kernel-modules-extra-matched-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
kernel-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
kernel-cross-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
```
There are warnings:
```text
RPM build warnings:
File not found:
/home/rocky/rpmbuild/BUILDROOT/kernel-6.12.0-124.21.1.el10.loongarch.loongarch64/usr/include/cpufreq.h
File not found:
/home/rocky/rpmbuild/BUILDROOT/kernel-6.12.0-124.21.1.el10.loongarch.loongarch64/usr/include/ynl
```
Regenerate the source package once, to avoid it being deleted.

Install the compiled rpm package:
```bash
sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/loongarch64/kernel-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
```

## glibc

The following modifications were made:

Changed `--enable-systemtap` to `--disable-systemtap`

Added some support for loongarch:
```text
%ifarch loongarch64
%global glibc_ldso /lib64/ld-linux-loongarch-lp64d.so.1
%global glibc_has_libnldbl 1
%global glibc_has_libmvec 0
%endif
```
Settings:

BuildFlagsNonshared=""

If the architecture is LoongArch, do not use Patch261.

Some files that failed to be generated, leave them alone for now.

When compiling, without testsuite, benchtests, and valgrind.

More than 200 files were generated.

Install files, and adjust the toolchain.

Test the toolchain.
```bash
readelf -l a.out |grep ": /lib"
```
Output:
```text
[Requesting program interpreter: /lib64/ld-linux-loongarch-lp64d.so.1]
```
The toolchain was adjusted correctly, it is no longer the ld in /tools/lib64!

## Zlib

Build normally, no need to adjust the spec file.

Install:
```bash
sudo rpm -Uvh $(ls ~/rpmbuild/RPMS/loongarch64/{minizip,zlib}-* |grep -v -e "debug") --nodeps
```

## libxcrypt

Since gpg2 is not installed, you need to comment out the gpgverify line, or add the with bootstrap option.

Encountered this error:
```bash
fipshmac -d
/home/rocky/rpmbuild/BUILDROOT/libxcrypt-4.4.36-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/fipscheck
/home/rocky/rpmbuild/BUILDROOT/libxcrypt-4.4.36-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/libcrypt.so.2.0.0
Allocation of hmac(sha256) cipher failed (ret=-97)
```

During the process of compiling libxcrypt, the fipshmac program needs to be used. The operation of this program depends on a series of encryption-related kernel modules, so the following commands need to be run:
```bash
modprobe crypto_user
modprobe crypto_hmac
modprobe af_alg
modprobe crypto_hash
modprobe algif_hash
```
Reference:

<https://gitee.com/openeuler/community/issues/I1KQMK>

Encountered a new bug: when fipshmac runs, it automatically adds a dot in front of the generated .hmac file, which causes the final filename to not match.

Modify the spec file, at:
```bash
fipshmac -d $fipsdir                        \\\
  $libdir/libcrypt.so.%{sov}                \
```

two lines were added after it:
```bash
mv -v $fipsdir/.libcrypt.so.%{sov}.hmac     \\\
   $fipsdir/libcrypt.so.%{sov}.hmac         \
```

Compilation method:
```bash
rpmbuild -bb ~/rpmbuild/SPECS/libxcrypt.spec --nodeps --with bootstrap --without new_api --nocheck --without compat_pkg --without staticlib
```

Since I added `--without compat_pkg` and `--without staticlib`, far fewer packages were generated by the compilation, only four:
```text
libxcrypt-devel-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-debuginfo-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-debugsource-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
```
Install the first two packages:
```bash
sudo rpm -ivh libxcrypt-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm \
              libxcrypt-devel-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm \
              --nodeps
```
libxcrypt needs libssp.so, which is why libssp.so had to be symbolically linked earlier.

## binutils

To be safe, disable gold for LoongArch.

Modify the spec file

Add loongarch64 on the line before %bcond_without gold

Regenerate the source package
```bash
rpmbuild -bs rpmbuild/SPECS/binutils.spec
```
Compile the rpm package:
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/binutils-2.41-58.el10.loongarch.2.src.rpm --with bootstrap --nodeps --nocheck --without debuginfod
```
4 new rpm packages were generated.

Install:
```bash
sudo rpm -ivh binutils-2.41-58.el10.loongarch.2.loongarch64.rpm \
       binutils-devel-2.41-58.el10.loongarch.2.loongarch64.rpm \
       --nodeps
```
Since there is no `/usr/sbin/alternatives`, the symbolic link needs to be set manually:

```bash
sudo ln -sv ld.bfd /bin/ld
```

## GMP

When compiling, a libzstd.so.1 file is needed. In /usr/lib64, create a symbolic link pointing to /tools/lib64/libzstd.so.1

After installing the gmp source package, modify `~/rpmbuild/SOURCES/gmp.h`

At an appropriate place, add:
```c
#elif defined(__loongarch64)
#include "gmp-loongarch64.h"
```
Modify the Release number.

Repackage the source package
```bash
rpmbuild -bs ~/rpmbuild/SPECS/gmp.spec
```
Recompile the packaged source package
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/gmp-6.2.1-12.el10.loongarch.1.src.rpm --nodeps
```

Install:
```bash
sudo rpm -ivh gmp-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-c++-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-devel-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-static-6.2.1-12.el10.loongarch.loongarch64.rpm \
            --nodeps
```

## MPFR

No special handling is needed, just build and install directly.

## libMPC

Compile directly.

Install:
```bash
sudo rpm -ihv libmpc-devel-1.3.1-7.el10.loongarch.loongarch64.rpm \
  libmpc-1.3.1-7.el10.loongarch.loongarch64.rpm \
  ../noarch/libmpc-doc-1.3.1-7.el10.loongarch.noarch.rpm \
  --nodeps
```

## XZ

Modify the spec file, do not perform gpgverify

Install:
```bash
sudo rpm -ivh xz-devel-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-libs-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-lzma-compat-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-static-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-5.6.2-4.el10.loongarch.loongarch64.rpm \
  --nodeps
```

## meson

Depends on python-setuptools.

Generated file: meson-1.4.1-4.el10.loongarch.noarch.rpm

**Note**
- After installing meson, it will create the file macros.meson in /usr/lib/rpm/macros.d. But the rpm on the system is still the one in /tools/, so a symbolic link needs to be made.
- meson is a python program, so its installation location needs attention. The system's python is still in /tools/, so a symbolic link needs to be made when necessary!
- I also installed another meson with pip. The one that finally takes effect might be this meson.

The version_no_tilde macro should be defined in rust-srpm-macros.

## ninja-build

When meson runs, it depends on ninja-build
```bash
rpmbuild --rebuild /opt/srpms/Packages/ninja-build-1.11.1-9.el10.src.rpm --nodeps --with bootstrap
```

## LZ4

Compilation needs meson.

meson depends on python-setuptools

Compile and install the 4 packages that do not contain debug.

## ZSTD

When compiling, it reported that a command is missing:
```text
line 46: execstack: command not found
```
Trying to compile execstack, encountered a new error:
```text
configure: error: libelf does not properly convert Elf64_Sxword
quantities.
If you are using libelf-0.7.0, please use patches/libelf-0.7.0.patch.
```
Skip the execution of the execstack command for now
```bash
rpmbuild --rebuild /opt/srpms/Packages/zstd-1.5.5-9.el10.src.rpm \
        --nodeps \
        --nocheck
```
7 rpm files were generated.
Install
```bash
sudo rpm -ivh libzstd-1.5.5-9.el10.loongarch.loongarch64.rpm \
  libzstd-devel-1.5.5-9.el10.loongarch.loongarch64.rpm \
  zstd-1.5.5-9.el10.loongarch.loongarch64.rpm \
  libzstd-static-1.5.5-9.el10.loongarch.loongarch64.rpm \
  --nodeps
```

## GCC

The spec file needs to be modified, adding a patch.
`0001-fixincludes-Skip-pthread_incomplete_struct_argument-.patch`

There are still many places that need to be modified.

The loongarch-related header files need to be added.

For now, the documentation cannot be fully compiled, so skip the content related to the documentation.

The annobin plugin-related parts, skip for now.

Copy the homework from openEuler's source code.

<https://atomgit.com/src-openeuler/gcc/blob/master/gcc.spec>

Note, the GCC version on openEuler is 12, and the homework for this version cannot all be copied.

gcc 14.3 was released on May 23, 2025, and already includes support for libquadmath.

Start trying to compile GCC.

The AS program cannot run properly. It seems that binutils failed to correctly create the symbolic link for libbfd.so.

Make a symbolic link for now, to see whether the problem can be fixed.

Compiling once takes a very long time, about 8 hours.

Encountered a new problem again:
```text
+ cp -pr '/home/rocky/rpmbuild/BUILD/gcc-14.3.1-20250617/rpm.doc/libgccjit-devel/*' /home/rocky/rpmbuild/BUILDROOT/gcc-14.3.1-2.1.el10.loongarch.1.loongarch64/usr/share/doc/libgccjit-devel
cp: cannot stat '/home/rocky/rpmbuild/BUILD/gcc-14.3.1-20250617/rpm.doc/libgccjit-devel/*': No such file or directory
+ :
+ cp -pr /home/rocky/rpmbuild/BUILD/gcc-14.3.1-20250617/gcc/jit/docs/examples /home/rocky/rpmbuild/BUILDROOT/gcc-14.3.1-2.1.el10.loongarch.1.loongarch64/usr/share/doc/libgccjit-devel
+ RPM_EC=0
++ jobs -p
+ exit 0
error: File not found: /home/rocky/rpmbuild/BUILDROOT/gcc-14.3.1-2.1.el10.loongarch.1.loongarch64/usr/share/doc/libgccjit-devel/*
```

Modify the spec file, in the doc of libgccjit-devel, comment out the statements related to rpm.doc/libgccjit-devel/.

Compilation succeeded, taking a total of 462 minutes, 7.7 hours, and 46 files were generated.

The final warning information of packaging:

```text
 Installed (but unpackaged) file(s) found:
   /usr/bin/gnatgcc
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libasan.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libatomic.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libcaf_single.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libgcc.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libgcc_eh.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libgcov.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libgfortran.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libgomp.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libitm.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/liblsan.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libstdc++.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libstdc++exp.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libstdc++fs.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libsupc++.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libtsan.a
   /usr/lib/debug/usr/lib/gcc/loongarch64-redhat-linux/14/libubsan.a
   /usr/lib/gcc/loongarch64-redhat-linux/14/include-fixed/README
   /usr/lib/gcc/loongarch64-redhat-linux/14/include-fixed/pthread.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/include/ssp/ssp.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/include/ssp/stdio.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/include/ssp/string.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/include/ssp/unistd.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/fixinc_list
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/gsyslimits.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/include/README
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/include/limits.h
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/macro_list
   /usr/lib/gcc/loongarch64-redhat-linux/14/install-tools/mkheaders.conf
   /usr/lib64/libasan.so
   /usr/lib64/libatomic.so
   /usr/lib64/libgcc_s.so
   /usr/lib64/libgfortran.so
   /usr/lib64/libgomp.so
   /usr/lib64/libitm.so
   /usr/lib64/liblsan.so
   /usr/lib64/libstdc++.so
   /usr/lib64/libtsan.so
   /usr/lib64/libubsan.so
   /usr/libexec/gcc/loongarch64-redhat-linux/14/install-tools/fixinc.sh
   /usr/libexec/gcc/loongarch64-redhat-linux/14/install-tools/mkinstalldirs
   /usr/share/locale/de/LC_MESSAGES/libstdc++.mo
   /usr/share/locale/fr/LC_MESSAGES/libstdc++.mo
   /usr/share/man/man7/fsf-funding.7.gz
   /usr/share/man/man7/gfdl.7.gz
   /usr/share/man/man7/gpl.7.gz
```
It can be said that the compilation and packaging process is not very perfect, but it is still usable.
Install some of the rpm packages.

Clean up and finish.

Compile test.

## glibc, 2nd pass

Starting now, recompile the toolchain.

The glibc spec file has been modified, so it cannot be compiled directly from the source. Use the script written earlier for compiling glibc to compile.

Make sure the file /usr/lib/rpm/redhat/redhat-annobin-cc1 exists, and that this file is empty. Because the current GCC under LoongArch does not support annbin.

Compilation succeeded, install the packages, 442 packages in total.

## binutils, 2nd pass

Use the previous build script to compile.

Install:
```bash
sudo rpm -Uvh binutils-2.41-58.el10.loongarch.2.loongarch64.rpm \
              binutils-devel-2.41-58.el10.loongarch.2.loongarch64.rpm \
              --nodeps \
              --force
```
When installing, this error will still be encountered:

```text
/usr/sbin/alternatives: No such file or directory
```
Make the link manually.
```bash
sudo ln -sv ld.bfd /bin/ld
```

## ZLIB

The source code of this package has not been adjusted, so just use the original src.rpm package to build and install directly.
```bash
rpmbuild --rebuild /opt/srpms/Packages/zlib-1.2.11-40.el9.src.rpm --nodeps
```

## libxcrypt

Use the spec file left over from before to compile.

Note, many kernel modules still need to be modprobed.

## GMP

Normal operation.

## MPFR

Normal operation.

## LibMPC

Normal operation.

## XZ

Normal operation.

## LZ4

Normal operation.

## ZSTD

Normal operation.

## GCC, 2nd pass

Normal operation. Compilation took 447m.

Install gcc.

Clean up the unneeded library files.

Ensure that under /lib64, there are no symbolic links pointing to /tools.

# The Incomplete Target System

Continue completing the target system.

Set the default bootstrap.

At the end of `/tools/lib/rpm/platform/loongarch64-linux/macros`, add: `%with_bootstrap 1`
First, compile the system interactive environment.

## ncurses

Comment out the gpgverify line in the SPEC file

11 files were generated; install the 7 that are unrelated to debug.

## readline

Build normally.

## BASH

The gpgverify statement also needs to be commented out

Compilation generates 5 files; install the 3 that are unrelated to debug.

## bash-completion

Build and install normally.

## bzip2

Comment out gpgverify

After compilation, 7 files were generated.
Install the 4 that are unrelated to debug.

## TCL

Build and install directly.

## libdb

No longer needed.

## gdbm

Build normally

## Perl

The check fails; add `--nocheck` when compiling.

A very large number of files were generated, 229 in total; 188 of them were installed.

Adjust the perl-related part of the rpm macros.

## Perl-srpm-macros

Building and installing the package is very simple. 1 package was generated: `perl-srpm-macros-1-57.el10~bootstrap.loongarch.noarch.rpm`.

## Perl-Fedora-VSP

Build normally.

## Perl-generators

Build normally.

## Python-rpm-macros

Build normally, install.

**Note**: remember to copy the files to `/tools/lib/rpm/macros.d/`

## Unzip

When actually packaging, you still need to add `--nodeps`

## Zip

Add `--nodeps` when packaging.

## help2man

  Build normally.

## M4

Build normally.

## Autoconf

  When compiling, remove support for emacs.
  ```bash
  rpmbuild --rebuild /opt/srpms/Packages/autoconf-2.71-13.el10.src.rpm \
            --without autoconf_enables_emacs \
            --nodeps \
            --nocheck
  ```

## Automake

```bash
rpmbuild --rebuild /opt/srpms/Packages/automake-1.16.5-20.el10.src.rpm \
        --without check \
        --nodeps
```

## Texinfo

Modify the spec file

5 files were generated; install the 3 that are unrelated to debug.

## libtool

Build normally.

## Pkgconf

10 rpm packages were generated. The compile command is as follows:
```bash
rpmbuild --rebuild /opt/srpms/Packages/pkgconf-2.1.0-3.el10.src.rpm --nocheck --nodeps
```

## Lua

Build normally.

## diffutils

Build normally.

## attr

Build normally.

## acl

Build normally.

## tar

```bash
rpmbuild --rebuild /opt/srpms/Packages/tar-1.35-9.el10_1.src.rpm \
        --without check \
        --without selinux \
        --nodeps
```

## cpio

  Build normally.

## bison

Build normally. The tests take a very long time.

## FLEX

Done.

## Ed

The gpgverify statement in the spec file needs to be commented out.

## patch

Modify the selinux-patch-related part of the spec file; do not apply this patch.

## Check

It needs to depend on cmake to compile. Skip it for now.

## CMake

Give it a try and see whether forcing the cmake build succeeds.

It depends on openssl. Skip it for now; come back to compile cmake after openssl is built.

## make

Build normally.

## tcsh

Do not run check

## sed

Build normally.

## grep

Build normally.

## findutils

Build normally.

## file

Build script
```bash
rpmbuild -bb ~/rpmbuild/SPECS/file.spec --nodeps --without python3
```

## NSPR

The nspr source code is in NSS; it is no longer compiled separately.

## sqlite

The tests take a relatively long time.

## pcre2

There is no pcre anymore, only pcre2.

It depends on readline.h; this package needs to be installed.

The check did not pass; skip the check.
```bash
rpmbuild -bb ~/rpmbuild/SPECS/pcre2.spec --nodeps --nocheck
```

## libsepol

Build normally.

## libgpg-error

Build normally.

## libgcrypt

Build error encountered:
```text
lto1: fatal error: inaccessible plugin file plugin/annobin.so expanded
from short plugin name annobin: No such file or directory
```

Comment out patch1, the patch related to annobin.

The tests take quite a lot of time. The check needs the annocheck command, which is not installed, so skip the check.

## chrpath

Build normally.

## expect

It needs to depend on chrpath. Already installed.

## screen

Source package not found.

## dejagnu

Build normally.

## libffi

Modify the spec file

Modify `~/rpmbuild/SOURCES/ffi-multilib.h`, add
```c
#elif defined(__loongarch64)
#include "ffi-loongarch64.h"
```

Modify ffitarget-multilib.h, add
```c
#elif defined(__loongarch64)
#include "ffitarget-loongarch64.h"
```
A compilation error was encountered
```text
../src/loongarch64/ffi.c:525:7: error: implicit declaration of function 'ffi_tramp_set_parms' [-Wimplicit-function-declaration]
```

After searching, this problem has already been fixed by xry111:

<https://github.com/libffi/libffi/pull/825>

Download <https://github.com/libffi/libffi/pull/825.patch>,
modify the spec file and add this patch.

## lksctp-tools

Done

## openssl

Modify the spec file, add architecture support for loongarch64

Repackage the source code and build the package.

It takes quite a while.

During packaging, this error was encountered:
```text
+ rename so.3 so.3.5.1 '/home/rocky/rpmbuild/BUILDROOT/openssl-3.5.1-4.el10~bootstrap.loongarch.1.loongarch64/usr/lib64/*.so.3'
rename: /home/rocky/rpmbuild/BUILDROOT/openssl-3.5.1-4.el10~bootstrap.loongarch.1.loongarch64/usr/lib64/*.so.3: not accessible: No such file or directory
error: Bad exit status from /var/tmp/rpm-tmp.9zb2oM (%install)
```
Because the so files end up installed to `/usr/lib/`, `_lib` should be set to `/usr/lib64/`

Modify the SPEC file and set the libdir path for the LoongArch architecture
```bash
%ifarch riscv64 loongarch64
--libdir=%{_lib} \
%endif
```

## libxml2

Modify the spec file, remove the handling and packaging related to the `__pycache` directory.

The PKG_CONFIG_PATH environment variable needs to be set.

During compilation, this error was encountered:
```text
tree.c:227:17: error: 'SIZE_MAX' undeclared (first use in this function)
  227 |     if (lenn >= SIZE_MAX - lenp - 1)
      |                 ^~~~~~~~
tree.c:49:1: note: 'SIZE_MAX' is defined in header '<stdint.h>'; this is probably fixable by adding '#include <stdint.h>'
   48 | #include "private/tree.h"
  +++ |+#include <stdint.h>
   49 |
```

After checking, someone has already mentioned this problem:

<https://gitlab.gnome.org/GNOME/libxml2/-/commit/3773bb3f89426297dd82d8ad5998059898d5172a>

Learned from the large model how to download patch files: just download <https://gitlab.gnome.org/GNOME/libxml2/-/commit/3773bb3f89426297dd82d8ad5998059898d5172a.patch>
to get the patch.

Modify the spec file, manually create a patch, and apply it.

Another new problem was encountered
```text
ERROR   0002: file '/usr/lib64/libxml2.so.2.12.5' contains an invalid rpath '/tools/lib64' in [/tools/lib64]
```

Following the program's hint, add `QA_RPATHS=$(( 0x0002|0x0004 ))` to work around the above problem.

## libxslt

Build normally.

## sgml-common

Build normally.

## docbook-dtds

Build normally.

## docbook-style-xls

Build normally.

## xmlto

Build normally.

## expat

Build normally.

## mpdecimal

Build normally.

## python3

It depends on mpdecimal-devel; install this first.

Modify it as required in the book.

## libxml2

After resolving the dependencies, rebuild libxml2

## NSS

Following the hints in the book, modify the spec file and comment out part of the code.

`config.sub` and `config.guess` are too old and need to be updated.

Use `autoreconf -ifv` to update them.

## PCRE2

Already installed

## LZO

Build normally.

## icu

Do not generate documentation.

## Boost

Remove some feature modules.

Some build errors were encountered:
```text
Installed (but unpackaged) file(s) found:
/usr/lib/debug/usr/lib64/libboost_python312.so.1.83.0-1.83.0-5.el10~bootstrap.loongarch.loongarch64.debug
/usr/lib64/libboost_python312.so
/usr/lib64/libboost_python312.so.1.83.0
```

For now, add the parameter `--define _unpackaged_files_terminate_build 0` to rpmbuild to work around this problem

## Cmake

Remove some feature modules.

cmake depends on vim-filesystem.

Manually added the definition of the vimfiles_root macro: `--define "vimfiles_root %{_datadir}/vim/vimfiles/"`

When encountering unpackaged files, do not terminate; add the statement: `--define '_unpackaged_files_terminate_build 0'`

The test takes a very long time; if it does not pass, add `--nocheck`

## Hostname

Build normally.

## Coreutils

Comment out the gpg verification part in the spec. Two cases will fail; never mind. Add `--nocheck`.

The coreutils-single and coreutils packages conflict; don't install the former.

## Python-rpm-generators

Build and install normally. It also depends on python-packaging,
pyproject-rpm-macros, pyproject-srpm-macros,
python-packaging, all of which need to be built and installed.

## Python-setuptools

It depends on python-rpm-generators.

## Re2c

Source code not found; do not compile.

## Ninja-build

Build normally. There is no need to fix rpmmacrodir, because this macro is no longer used in the current spec file.

## Meson

Encountered the version_no_tilde macro. Just define it.
```bash
rpmbuild --rebuild /opt/srpms/Packages/meson-1.4.1-4.el10.src.rpm --define " version_no_tilde 1.4.1"
```
Copy macros.meson to /tools/lib/rpm/

## Dos2unix

Comment out the gpgverify statement in the spec file.

## Swig

Add the `--without testsuite` parameter

## Less

Build normally.

## Gzip

Build normally.

## libpaper

Build normally.

## chrpath

Following the approach in the book, modify the spec file.

Because subsequent builds need chrpath to perform some checks, we need to skip these checks.
```bash
rpmbuild -ba ~/rpmbuild/SPECS/chrpath.spec --define '_unpackaged_files_terminate_build 0'
```

## Psutils

The tests do not pass. Add `--nocheck`.

## libedit

Build normally.

## multilib-rpm-config

Build normally.

## llvm

This error was encountered:
```text
错误：lua 脚本执行失败：[string "python_provide"]:2: module 'fedora.srpm.python' not found:
```
python-rpm-macros needs to be rebuilt and installed.

## python-lit

Source package not found; cannot install.

## clang

Related to llvm; skip for now.

## ccache

Source package not found; cannot install.

## nasm

Build normally.

## Gpref

Build normally.

## libutempter

Build normally.

## libpipeline

Build normally.

## giflib

Start compiling graphics-related packages.

The giflib package depends on mingw64-gcc, mingw64-filesystem

Modify the spec file to make a version that does not depend on mingw at all.

## libpng

Build normally.

## libjpeg-turbo

Build normally.

## Jbigkit

Build normally.

## libtiff

Build normally.

## lcms2

Build normally.

## libmng

Build normally.

## Xorg-X11-utils-macros

Build normally.

## Xorg-X11-Proto-devel

Build normally.

## libXau

Build normally.

## libXdmcp

Build normally.

## xcb-proto

Build normally.

## libxcb

Build normally.

## xorg-x11-xtrans-devel

Build normally.

## libx11

Build normally.

## brotli

Build normally.

## freetype

Depends on brotli

## forge-srpm-macros

Add `--nocheck` when packaging
Build script:
```bash
rpmbuild --rebuild /opt/srpms/Packages/forge-srpm-macros-0.4.0-6.el10.src.rpm --nodeps --nocheck
```

## fonts-rpm-macros

It depends on forge-srpm-macros; install it along the way.

Copy the installed macro definition files to `/tools/lib/rpm/macrod.d/`

## fontconfig

Just follow the book's instructions.

## Pixman

Build normally.

## libimagequant

Source code not found.

## libICE

Build normally.

## libSM

Modify the spec file; change `--with-libuuid` to `--without-libuuid`

## libXrender

Build normally.

## libXext

Build normally.

## libXt

Build normally.

## libXpm

Build normally.

## libXmu

Build normally.

## libXaw

Build normally.

## libXfixes

Build normally.

## libXi

Build normally.

## libXft

Build normally.

## libXinerama

Build normally.

## libXtst

Build normally.

## python-wheels

Build normally.

## python-pip

Build normally.

## marshalparser

Build normally.

## glib2

depends on g-ir-scanner, try to build and install gobject-introspection. gobject-introspection also depends on glib2. A circular dependency has appeared. Add `-Dintrospection=disabled`
to meson.

depends on sysprof-capture-4, try to satisfy it, and it is another circular dependency. Continue to break the dependency: `-Dsysprof=disabled`

depends on libelf, continue adding `-Dlibelf=disabled`

It also depends on rst2man, which needs python3-docutils; continue to break the dependency with `-Dman-pages=disabled`

It is still missing some things, missing marshalparser, so add it. Also added the python-pip,
python-wheels packages.

During check there are several errors:
```text
 16/375 glib:glib+core+slow / gdatetime                                        ERROR            0.01s   killed by signal 6 SIGABRT
 22/375 glib:glib+core / hmac                                                  ERROR            0.01s   killed by signal 5 SIGTRAP
109/375 glib:glib+core / autoptr                                               ERROR            0.01s   killed by signal 5 SIGTRAP
217/375 glib:gio / contenttype                                                 ERROR            0.01s   killed by signal 6 SIGABRT
282/375 glib:gio / file                                                        ERROR            0.04s   killed by signal 5 SIGTRAP
304/375 glib:gio / desktop-app-info                                            ERROR            0.01s   killed by signal 5 SIGTRAP
```
These errors are understandable. Add `--nocheck`

There are still errors:
```text
 Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GIRepository-3.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GLib-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GLibUnix-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GModule-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GObject-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/Gio-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/lib64/girepository-1.0/GioUnix-2.0.typelib
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gio.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gio-querymodules.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/glib-compile-schemas.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gsettings.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gdbus.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gapplication.1*
```
Continue to modify the spec file, do not want these files

There are still errors:
```text

File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/glib-genmarshal.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/glib-gettextize.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/glib-mkenums.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gi-compile-repository.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gi-decompile-typelib.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gi-inspect-typelib.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gobject-query.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gtester-report.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gtester.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gdbus-codegen.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/glib-compile-resources.1*
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/man/man1/gresource.1*
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GIRepository-3.0.gir
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GLib-2.0.gir
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GLibUnix-2.0.gir
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GModule-2.0.gir
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GObject-2.0.gir
    File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/Gio-2.0.gir
File not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/gir-1.0/GioUnix-2.0.gir
```
Continue to handle the missing directories
```text
  Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gio-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gio-unix-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/girepository-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/glib-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/glib-unix-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gmodule-2.0
Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gobject-2.0
```
The build succeeded; install the four non-debug packages.

## Cairo

The new version of cairo changed its build method to meson.

First try to build it and see what errors appear
Modify the spec file, the following changes are needed:

- `-Dgtk_doc=false`

- remove `gtk-doc/html/cairo`

## gd

During compilation add `--nodeps --nocheck`

## libcroco

Source package not found.

## Jbig2dec

Build normally.

## OpenJPEG2

Build normally.

## graphite2

The new version of graphite2 no longer builds docs.

## Python-mako

```bash
rpmbuild --rebuild /opt/srpms/Packages/python-mako-1.2.3-9.el10.src.rpm --nodeps --nocheck
```

## gobject-introspection

Build normally.

## Harfbuzz

This package is also built with meson, it is missing gtk-doc, add parameters to meson
```text
-Ddocs=disabled -Ddoc_tests=false -Dtests=disabled
-Dintrospection=disabled
```

## T1lib

Source package not found.

## T1utils

Source package not found.

## Xaw3d

Comment out the gpgverify statement

## libXComposite

Build normally.

## zziplib

Build normally.

## teckit

Build normally.

## potrace

Build normally.

## Texlive

texlive depends on zziplib, teckit, potrace, javac

Remove all content that depends on javac, that is, tex4ht.

In the end, 398 packages were generated.

## gettext

Build normally.

## adobe-mapping-cmap

Build normally.

## adobe-mapping-pdf

Build normally.

## symlinks

Build normally.

## urw-base35-fonts

Build normally.

## xorg-x11-font-utils

Build normally.

## libidn2

Build normally.

## libijs

Build normally.

## ghostscript

Build normally.

After compilation, the ps2pdf command has problems running, and for now it is not known how to solve it. Try to complete the dependencies as much as possible and rebuild.

After completing the dependencies such as libijs, libidn2, urw-base35-fonts, the ps2pdf command could run normally.

## google-droid-font

Build normally.

## libsigsegv

Build normally.

## poppler-data

Build normally.

## poppler

depends on Gpgmepp. Do not want this, just disable it. Add `-DENABLE_GPGME=OFF` to the SPEC file

Currently no Qt-related packages have been built, so disable Qt as well. Add `--without qt` to rpmbuild

curl is not found, disable it for now. The following changes need to be made in the spec file:
```text
-DENABLE_GTK_DOC=OFF \\
-DENABLE_GPGME=OFF \\
-DENABLE_LIBCURL=OFF \\
-DENABLE_GOBJECT_INTROSPECTION=OFF \\
```

Build script

```bash
rpmbuild -bb rpmbuild/SPECS/poppler.spec  --nodeps --nocheck --without qt
```

## ZZiplib

Already installed.

## Teckit

Already installed.

## FFCall

Source package not found.

## clisp

Source package not found.

## Potrace

Already installed.

## perl-devel-checklib

Build normally.

## perl-XML-parser

Build normally.

## perl-XML-XPath

Build normally.

## texlive-base

Source package not found.

## gawk

Build normally.

## groff

Build normally.

## Xapian-core

Build normally.

## QPdf

Modify the CMake settings
```text
-DREQUIRE_CRYPTO_GNUTLS=0\\
-DALLOW_CRYPTO_NATIVE=1 \\
-DREQUIRE_CRYPTO_NATIVE=1 \\
```

## TK

Build normally.

## Graphviz

Build script
```bash
rpmbuild -bb ~/rpmbuild/SPECS/graphviz.spec --without php --without ocaml --nodeps
```

## Doxygen

Build script:
```bash
rpmbuild -bb rpmbuild/SPECS/doxygen.spec --define "_module_build 1" --nodeps --nocheck
```

## symlinks

Already installed.

## asciidoc

 Build normally.

## opensp

Build normally.

## openjade

Build normally.

## linuxdoc-tools

Build normally.

## elinks

Build normally.

## python3-libxml2

Rebuild. Note, use the repackaged libxml2.

## itstool

Build normally.

## docbook-style-dssl

Build normally.

## perl-sgmlspm

Build normally.

## dockbook-utils

Build normally.

## docbook5-style-xsl

Build normally.

## oniguruma

Build normally.

## slang

Build normally.

## telnet

Build normally.

## lynx

depends on telnet, add it.

## words

Build normally.

## cracklib

Build normally.

## libtirpc

Build normally.

## libnsl2

Source package not found.

## libyaml

Build normally.

## procps-ng

Build normally.

## which

Build normally.

## checksec

Source package not found.

## rubypick

Source package not found.

## ruby

During testing, many tests related to TZ will fail.
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/ruby-3.3.10-12.el10~bootstrap.loongarch.src.rpm --without systemtap --define "with_hardening_test 0" --nodeps --nocheck
```

## Important reminder

The software packages that will be built starting below are related to the system's boot and login process.

In the current state, the system's login depends on the files in the /tools/ directory; as compilation proceeds, some executable files in /tools will be replaced by files provided by the newly built rpm packages, causing the system to be in a chaotic mixed state: the files provided by the rpm packages and the files in /tools work together, but they are not fully compatible with each other.

**Note**: From now on, if you reboot the virtual machine, it may cause the virtual machine to be unable to log in! It is recommended that from now on you do not shut down the virtual machine, and use the "save virtual machine" method instead of powering off. It is best to complete the compilation of all login-related software packages in one go, and then you can safely reboot.

**Remember!**

If you have already encountered the problem of being unable to log in to the virtual machine, there is no need to panic; the solution will be introduced later.

## libselinux

Build normally.

## libcap-ng

Build normally.

## pam

The dependency needs to be resolved.

Modify the spec file, change `--enable-audit` to `--disable-audit`

When installing the documentation, remove the txt files for adg mwg sag. Build script:
```bash
rpmbuild -bb rpmbuild/SPECS/pam.spec --define '_unpackaged_files_terminate_build 0' --nodeps
```

## libpwquality

Build normally.

## audit

Build script
```bash
rpmbuild -bb rpmbuild/SPECS/audit.spec --nodeps --define '_unpackaged_files_terminate_build 0'
```

## lmdb

Build normally.

## popt

Build normally.

## libuser

needs to use the gtkdocize command from gtk-doc. Skip this for now, just build.

In the spec file, add

```bash
sed -i "s/gtkdocize/#gtkdocize/g" autogen.sh
```

change `--enable-gtk-doc` to `--disable-gtk-doc`

remove the `--html-doc` statement

Modify `docs/reference/Makefile.am`,
remove the statements related to `gtk-doc.make`, and remove the judgments related to `ENABLE_GTK_DOC`.

## libsemanage

Build normally.

## shadow-utils

Build normally.

## libutempter

Build normally.

## rubygem-asciidoctor

Build normally.

## util-linux

depends on `rubygem-asciidoctor`

change `--with-udev` to `--without-udev`, `--with-systemd` to `--without-systemd`

## libev

Build normally.

## libevent

Build normally.

## libverto

Build normally.

## fuse

Build normally.

## e2fsprogs

add `-D_FILE_OFFSET_BITS=64`

## keyutils

Build normally.

## krb5

depends on libcom_err, this package is provided by e2fsprogs.

## libksba

Build normally.

## libassuan

Build normally.

## npth

Build normally.

## json-c

Build normally.

## gnupg2

Modify the SPEC file, remove the dependency on `tpm2-tss`

Do not produce the dirmngr file and dirmngr-client file

## python-setuptools_scm

add `--nodeps --nocheck` parameters

## python3-pluggy

Build normally.

## python-iniconfig

Build normally.

## libtasn1

Build normally.

## libcap

Build normally.

## bash-completion

Build normally.

## newt

Build normally.

## beakerlib

Source package not found.

## chkconfig

Build script
```bash
rpmbuild -bb rpmbuild/SPECS/chkconfig.spec --nodeps  --define '_unpackaged_files_terminate_build 0'
```

## p11-kit

 Done.

## fipscheck

 Source package not found.

## nettle

Build normally.

## ca-certificates

Build normally.

## libunistring

Build normally.

## gc

Source package not found.

## guile

Source package not found.

## autogen

Source package not found.

## trousers

Source package not found.

## pytest

Make up your mind and add this very basic gtk-doc package. Complete the dependencies little by little.

```bash
rpmbuild --rebuild /opt/srpms/Packages/pytest-7.4.3-5.el10.src.rpm --without tests --without timeout --without docs --nocheck
```
Encountered this error:
```text
 FileExistsError: %pyproject_install has found more than one *.dist-info/RECORD file. Currently, %pyproject_save_files supports only one wheel → one file list mapping. Feel free to open a bugzilla for pyproject-rpm-macros and describe your usecase.
 error: Bad exit status from /var/tmp/rpm-tmp.FVnBBF (%install)
```

Solution:

Delete all the pytest-related content in the `~/rpmbuild/BUILD`, `~/rpmbuild/BUILDROOT` folders, then rebuild.

## cython

```bash
 rpmbuild --rebuild /opt/srpms/Packages/Cython-3.0.9-8.el10.src.rpm --nodeps
```

## python-lxml

 Build with root

## python-pathspec

Build normally.

## python-trove-classifiers

Build normally.

## python-hatchling

Build normally.

## python-pygments

Build normally.

## gtk-doc

```bash
rpmbuild --rebuild /opt/srpms/Packages/gtk-doc-1.33.2-12.el10.src.rpm --nodeps
```

## Troubleshooting

During the build process, I shut down the virtual machine, and after rebooting, I could no longer log into the virtual machine; after entering the password it showed login incorrect.

Directly mount the qcow2 file, chroot into it, and check the logs

/tools/bin/journalctl

I saw these errors:
```text
May 04 13:53:45 RockyLoong (agetty)[410]: console-getty.service: Executing: /sbin/agetty -o "-- \\u" --noreset --noclear --keep-baud 115200,57600>
May 04 13:53:45 RockyLoong systemd[1]: Got handoff timestamp event for PID 410.
May 04 13:53:49 RockyLoong login[410]: PAM _pam_load_conf_file: unable to open config for postlogin
May 04 13:53:49 RockyLoong login[410]: PAM _pam_load_conf_file: unable to open config for postlogin
May 04 13:53:55 RockyLoong login[410]: FAILED LOGIN 1 FROM console FOR rocky, Authentication failure
```
After searching and analyzing with a large model, it was confirmed that the `/etc/pam.d/postlogin` file was missing.

Add this file back and try again.

Using chroot, enter the already mounted qcow2 image, then execute
```bash
cat <<EOF > /etc/pam.d/postlogin
#%PAM-1.0
session optional pam_umask.so
session optional pam_lastlog.so nowtmp
EOF
```

After adding this file back, the passwd program could run normally, but logging in still had problems.

I consulted Teacher Sun Haiyong, and the root cause of the problem was found: during the process of building packages, the login-related packages in the temporary system `/tools/` folder were gradually being replaced by newly built rpm packages, but if only half of them were replaced, login problems would appear.

The solution to this problem is not to use the virtual machine, but instead to mount the virtual machine's qcow2 file into the system, then use chroot to enter the virtual machine and start building. This way, the login process can be bypassed.

From now on, the subsequent package builds are carried out in a chroot environment.

## libtirpc

Since krb5 has already been built, all the dependencies of this package are satisfied. Build normally.

## libselinux

When calling multiprocess, I encountered a Permission denied problem:
```text
sl = self._semlock = _multiprocessing.SemLock(
                         ^^^^^^^^^^^^^^^^^^^^^^^^^
PermissionError: [Errno 13] Permission denied
```

Package as root to work around this problem.

## libcap-ng

Package as root.

## pam

Commented out the for loops in the spec file concerning adg, dmg and other documentation

Build script:
```bash
rpmbuild -bb rpmbuild/SPECS/pam.spec --nodeps --define '_unpackaged_files_terminate_build 0'
```

When packaging, there is a warning:
```text
Installed (but unpackaged) file(s) found:
/usr/lib/systemd/system/pam_namespace.service
```

## libpwquality

Switch to packaging as root, and it builds normally.

## audit

Note: the audit configure script seems to have processor-related parts, such as `--with-aarch64` `--with-riscv`

It needs to be confirmed whether this affects use under loongarch.

Warning messages when packaging:
```text
  Installed (but unpackaged) file(s) found:
   /usr/lib/systemd/system/audit-rules.service
   /usr/lib/systemd/system/auditd.service
   /usr/lib/tmpfiles.d/audit.conf
```

## util-linux

Modify the build parameters in the spec file,

- Change `--with-udev` to `--without-udev`

- Change `--with-systemd` to `--without-systemd`

- Comment out some files:
```text
 %{_unitdir}/fstrim.*
uuidd.*
uuidd-tmpfiles.conf
```

## ima-evm-utils

Build normally.

## elfutils

Modify the spec file as follows

`--enable-debuginfod-ima-verification=no`

Delete `--enable-debuginfod-ima-cert-path=%{_sysconfdir}/keys/ima`

## strace

Build normally. After installing strace, the strace tool can be used to trace the execution of programs.

Through strace tracing, it was found that the system was missing `/etc/nsswitch.conf`; according to Sun Haiyong's CLFS documentation, add this file back.

## tss2

Build normally.

## tpm2-tss

Build normally.

## gnutls

It depends on tss2, which needs to be built in advance.

Build script
```bash
rpmbuild -bb rpmbuild/SPECS/gnutls.spec --without dane --without tpm2 --without tpm12 --nodeps
```

## crypto-policies

Modify the ExclusiveArch in the spec file, add loongarch64

Build script:
```bash
rpmbuild -bb rpmbuild/SPECS/crypto-policies.spec --nodeps --nocheck --define '_unpackaged_files_terminate_build 0'
```
 When packaging, there is an error like this:

```text
    Installed (but unpackaged) file(s) found:
   /usr/lib/systemd/system/fips-crypto-policy-overlay.service
```

## fakechroot

Source package not found. Skipped.

## libpq

Change `--with-ldap` to `--without-ldap`

## mariadb-connector-c

Build normally.

## cyrus-sasl

 Set `bootstrap_cyrus_sasl` to 1.

## unixodbc

Build normally.

## openldap

Build normally.

## stunnel

Build normally.

## byacc

Build normally.

## checkpolicy

Build normally.

## sharutils

Build normally.

## libarchive

Build normally.

## Brotli

Build normally.

## Cmocka

Build normally.

## ducktape

Build normally.

## polkit

Modify the spec file, `-Dintrospection=false`

Some files were not generated, such as `polkit.service, polkit-tmpfiles.conf, *.gir, *.typelib`, so the spec file needs corresponding modifications.

Build script:
```bash
PKG_CONFIG_PATH+=/tools/lib64/pkgconfig: rpmbuild -bb rpmbuild/SPECS/polkit.spec --nodeps  --define '_unpackaged_files_terminate_build 0'
```
Warnings given at final packaging:
```text
Installed (but unpackaged) file(s) found:
/usr/lib/systemd/system/polkit.service
/usr/lib/sysusers.d/polkit.conf
/usr/lib/tmpfiles.d/polkit-tmpfiles.conf
```

## pcsc-lite

Modify the spec file, change `-Dlibsystemd=true` to `-Dlibsystemd=false`

Build script:
```bash
PKG_CONFIG_PATH+=/tools/lib64/pkgconfig: rpmbuild -bb rpmbuild/SPECS/pcsc-lite.spec --nodeps
```

## opensc

Dependencies are basically satisfied, build normally.
```bash
rpmbuild -bb rpmbuild/SPECS/opensc.spec --nodeps
```

## cppunit

Build normally

## softhsm

Build normally

## pkcs11-provider

Build normally.

## libssh

It needs opensc, softhsm to be completed, and the spec file also needs to be modified
```text
-DUNIT_TESTING=OFF \\
-DCLIENT_TESTING=OFF \\
-DSERVER_TESTING=OFF
```

## libidn2

Build normally.

## publicsuffix-list

Build normally.

## libpsl

Build normally.

## curl

Build normally.

## jq

Build script
```bash
QA_RPATHS=$(( 0x0002|0x0010 )) rpmbuild --rebuild /opt/srpms/Packages/jq-1.7.1-11.el10.src.rpm
```

## socat

Build normally.

## elfutils

Modify the spec file as follows

`--enable-debuginfod-ima-verification=no`

## tss2

Build normally.

## ima-evm-utils

Build normally.

## autoconf-archive

Build normally.

## python3-mallard-ducktype

Build normally.

## yelp-xsl

Build normally.

## yelp-tools

Build normally.

## mallard-rng

Build normally.

## dbus

Build using the following command:
```bash
rpmbuild -bb rpmbuild/SPECS/dbus.spec  --nodeps  --define '_unpackaged_files_terminate_build 0'
```

Warning messages are as follows:
```text
absolute symlink: /usr/share/gtk-doc/html/dbus -> /usr/share/doc/dbus
absolute symlink: /usr/libexec/dbus-1/installed-tests/dbus/data/valid-config-files-system/system.conf -> /usr/share/dbus-1/system.conf
absolute symlink: /usr/libexec/dbus-1/installed-tests/dbus/data/valid-config-files/session.conf -> /usr/share/dbus-1/session.conf
Installed (but unpackaged) file(s) found:
/tools/lib/sysusers.d/dbus.conf
```

## crontabs

Build normally.

## cronie

Build normally.

## logrotate

Build script:
```bash
rpmbuild --rebuild /opt/srpms/Packages/logrotate-3.22.0-4.el10.src.rpm --nodeps --nocheck
```

## rpm

Since rust is not installed, rpm-sequoia cannot be installed. Temporarily remove the dependency on it.

I encountered a lua problem; looking at `/usr/include/luaconf.h`, I found that it does not support loongarch.

First, lua needs to be rebuilt and installed.

Modify `rpmbuild/SOURCE/luaconf.h`, add:
```bash
#elif defined(__loongarch64)
#include "luaconf-loongarch64.h"
```
When building again, a new problem was encountered: `undefined reference to `pgpDigParamsSalt`

Modify the spec file, remove two patches
```text
#rpm-4.19.x-pqc-algo.patch
#rpm-4.19.x-pqc-fixes.patch
```
These two patches caused the above problem
New problem:
```text
tools/rpmdb --rcfile rpmrc --define '_db_backend ndb' --dbpath=/root/rpmbuild/BUILD/rpm-4.19.1.1/_build/ndb --initdb
error: can't create transaction lock on / (是一个目录)
```
Following the hints in Sun Haiyong's book, comment out the relevant statements.

The final build script
```bash
rpmbuild -bb rpmbuild/SPECS/rpm.spec --without sequoia --nodeps
```
One of rpm's plugins is not very usable; temporarily disable this plugin, and you can rename it to `/usr/lib64/rpm-plugins/dbus_announce.so`

## perl-generators

Because the package management system has changed, at this stage some packages need to be rebuilt.

Build normally.

## redhat-rpm-config

Note: this package was modified before, so do not use the original package; package the modified one!

## python-rpm-generators

Build normally.

## fonts-rpm-macros

Build normally.

## ruby

ruby needs redhat-annobin-cc1, which should be emptied, because plugin/gcc-annobin.so does not exist. At the same time, empty redhat-hardened-ld.

Note: you must use the modified `redhat-rpm-config`, otherwise the parameters used when building ruby will be wrong, and the build will fail!

Many files were installed under /usr/local/share instead of under /usr/share/; the root cause of the problem needs to be confirmed. Such a problem should not occur.

The following changes were made, changing
```bash
--with-sitedir='%{ruby_sitelibdir}' \
--with-sitearchdir='%{ruby_sitearchdir}' \
```
to
```bash
--with-sitedir=%{_datadir}/ruby/site_ruby \
--with-sitearchdir=%{_libdir}/ruby/site_ruby \
```
Changing `mv %{buildroot}%{ruby_libdir}/gems %{buildroot}%{gem_dir}`

to
```bash
if [ -d %{buildroot}%{ruby_libdir}/gems ]; then
  mv -v %{buildroot}%{ruby_libdir}/gems %{buildroot}%{gem_dir}
fi
```

## lua

Build normally.

## dwz

It depends on xxhash and gdb.

When building, I encountered an error like this: the `NATIVE_POINTER_SIZE` parameter was not defined. It should be related to the current locale.

When building, add `LC_ALL=C`, and the problem is solved.

## babeltrace

Build normally.

## xxhash

Build normally.

## gdb

Build normally.

## kmod

Build normally.

## libaio

Build normally.

## libqb

Build normally.

## libnl3

Build normally.

## kronosnet

Build normally.

## corosync

Build script:
```bash
rpmbuild --rebuild /opt/srpms/Packages/corosync-3.1.9-2.el10.src.rpm --without systemd --without snmp --nodeps
```

## device-mapper-persistent-data

Missing the `%cargo_prep` macro. Need to build this package after completing `rust-toolset`.

## lvm2

In addition to making the modifications according to the content in the book, the spec file also needs to be changed to fix the configure parameters

```bash
--with-systemd=no
--disable-app-machineid
```

Fix the systemd-related content; modify `/usr/lib/rpm/macros.d/macros.systemd`, changing all `/tools/` to
`/usr/`.
```bash
sed -i "s@/tools/@/usr/@g" macros.systemd
```
This way, the systemd-related content will be installed to `/usr/lib/systemd`, instead of `/tools/lib/systemd`

In fact, this operation was already mentioned on page P171 of the book. Be sure to read the book carefully.

Rebuild to generate rpm packages, starting with lvm2-, device-mapper. First install some of the rpm packages.
```bash
rpm -ivh device-mapper-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      device-mapper-devel-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      device-mapper-libs-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      --nodeps
```
Note: many other packages may have already hit this pitfall, and they all need to be rebuilt as well.

## json-c

Build normally.

## argon2

Source package not found.

## cryptstepup

It depends on device-mapper. Packaging script
```bash
rpmbuild -bb rpmbuild/SPECS/cryptsetup.spec --define "version_no_tilde 2.7.5" --nodeps
```

## libpcap

Build normally.

## libmnl

Packaging script
```bash
rpmbuild --rebuild /opt/srpms/Packages/libmnl-1.0.5-7.el10.src.rpm --define '_unpackaged_files_terminate_build 0'
```
Generated a lot of documentation `man3/`

## libnftnl

Build normally.

## libnfnetlink

Build normally.

## libnetfilter_conntrack

Build normally.

## iptables

Build normally.

## qrencode

Source package not found.

## libxkbfile

Build normally.

## xorg-x11-xkb-utils

Source package not found

## xkeyboard-config

Install normally

## libxkbcommon

It depends on things from wayland; continue after completing them.

## console-setup

Missing the `/usr/lib/rpm/perl.req` program; see which package needs to be rebuilt.

perl-generators needs to be rebuilt, but this package depends on itself, so it needs to be bridged with /tools/lib/rpm/perl.req.

## kbd

It depends on xkeyboard-config

## libbpf

Build normally.

## python-docutils

Build normally.

## bpftool

It depends on rst2man, so python-docutils needs to be installed

## libseccomp

The source code seems to have architecture-related content. 2.5.6 does not support the LoongArch architecture. Needs to be confirmed.

<https://github.com/seccomp/libseccomp/pull/356>

It is best to upgrade libseccomp to 2.6.0

## libcbor

Remove the documentation build

## python-markupsafe

Build normally.

## python-jinja2

Build normally.

## systemd

It depends on libbpf, so install libbpf first.

It depends on bpftool,

It depends on clang; see whether it can avoid depending on clang.

Remove the dependency on bpf

`-Dbpf-framework=disabled`

`-Dbpf-compiler=gcc`

Remove the dependency on libseccomp.

`-Dseccomp=disabled`

Remove the dependency on fido2

`-Dlibfido2=disabled`

Remove the dependency on xkbcommon

`-Dxkbcommon=disabled`

Build script

```bash
rpmbuild -bb rpmbuild/SPECS/systemd.spec --define "version_no_tilde 257 " --nodeps --nocheck
```

## dbus-broker

Build normally.

## dbus

Build normally.

## ipset

Build normally.

## intltools

Build normally.

## jansson

Remove the documentation-building part.

## tree-sitter

Build normally.

## desktop-file-utils

It depends on emacs. Adjust the order and install emacs first.

Resolve the circular dependency.

When building, write it like this:
```bash
rpmbuild -bb rpmbuild/SPECS/desktop-file-utils.spec -D "_emacs_sitestartdir /usr/share/emacs/site-lisp/site-start.d" -D " _emacs_sitelispdir /usr/share/emacs/site-lisp" --nodeps
```

## appstream

Build normally.

## emacs

Depends on json; build jansson first.

Depends on tree-sitter.

emacs also depends on desktop-file-utils. Circular dependency.

Also depends on appstream-utils. Add `--nocheck` when building.

## google-noto-fonts

```bash
rpmbuild --rebuild /opt/srpms/Packages/google-noto-fonts-20240401-5.el10.src.rpm --nocheck
```

## xaw3d

Already built.

## vala

Build normally.

## libsecret

Depends on gi-docgen; need to find a way to bypass it or fill it in.

Add a line to the spec file: `-Dgtk_doc=false`

Need to fill in the package according to the error below.

```text
Couldn't find include 'GObject-2.0.gir' (search path: '['/usr/share/gir-1.0', '/root/.local/share/gir-1.0', 'gir-1.0', '/usr/local/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0']')
```
After confirmation, the file GObject-2.0.gir should be provided by the gobject-devel package.

After filling in glib2, build libsecret again, done.

## python3-mako

Done.

## PyYaml

Done

## Python-markdown

Add `--nocheck`

## gobject-introspection

Rebuild. All dependencies are already satisfied.

## shared-mime-info

Build normally.

## glib2

Rebuild, filling in the dependencies as much as possible.

The rebuilt glib2 filled in many files; importantly, it filled in `/usr/share/gir-1.0/GObject-2.0.gir`.

## Packages that can be built easily

- perl-error
- perl-termreadkey
- xorg-x11-xauth
- ~~libusbx~~ (no source package)
- ~~hidapi~~ (no source package)
- libcbor
- libfido2

## openssh

Build error encountered:
```text
configure: error: seccomp_filter sandbox not supported on
loongarch64-redhat-linux-gnu
```
comment out

`--with-sandbox=seccomp_filter`

check fails, add `--nocheck`

## git

Build script
```bash
rpmbuild --rebuild /opt/srpms/Packages/git-2.47.3-1.el10.src.rpm --nodeps --nocheck
```
Just selectively install some of the git packages.

## umockdev

Build script
```bash
rpmbuild --rebuild /opt/srpms/Packages/umockdev-0.19.1-2.el10.src.rpm --nodeps --nocheck
```

## libgudev1

Build normally.

## libusb

The dependencies are somewhat complex: libusb->umockdev-devel->libgudev1-devel->umockdev-devel

Need to resolve the circular dependency.

Resolved; build and install done.

## efi-rpm-macros

Build normally; there don't seem to be any problems.

## mandoc

Build normally.

## libabigail

Skip the documentation-generation part; the build passes.

## efivar

Add loongarch64 to exclusivearch. The source package needs to be repackaged.

## pesign

Add loongarch64 to exclusivearch. The source package needs to be repackaged.

## linux-atm

Source package not found.

## psmisc

Modify the spec file, adding loongarch in an architecture-related place. Repackage the source package too.
```bash
rpmbuild -ba rpmbuild/SPECS/psmisc.spec
```

## iproute

Build normally

## iputils

Build normally

## perl-Test-Pod

Build normally

## perl-Pod-Parser

Build normally

## perl-Test-Output

```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-Test-Output-1.03.4-6.el10.src.rpm --nodeps
```

## perl-IPC-Run3

```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-IPC-Run3-0.049-4.el10.src.rpm --nodeps --nocheck
```

## libmaxminddb

Try to fill in the dependencies as much as possible

## rubygem-ronn-ng

```bash
rpmbuild --rebuild /opt/srpms/Packages/rubygem-ronn-ng-0.10.1-5.el10.src.rpm --nodeps --nocheck
```

## rubygem-kramdown

```bash
rpmbuild  --rebuild /opt/srpms/Packages/rubygem-kramdown-2.4.0-11.el10.src.rpm --nodeps --nocheck
```

## rubygem-mustache

```bash
rpmbuild --rebuild /opt/srpms/Packages/rubygem-mustache-1.1.1-11.el10.src.rpm --nodeps --nocheck
```

## rubygem-nokogiri

Build normally

## rubygem-racc

Installation failed; unclear why.

The error is as follows:
```text
ERROR:  Error installing racc-1.7.3.gem:
ERROR: Failed to build gem native extension.
    No such file or directory @ dir_s_mkdir - /usr/share/gems/gems/racc-1.7.3/ext/racc/cparse/.gem.20260512-110776-rija44
```

## ipcalc

Missing racc, so it could not be installed. ronn is only used to generate documentation. Can skip.

Solution: remove the rubygems-ronn-ng package, and comment out ipcalc.1 in the file list
```bash
rpmbuild -bb rpmbuild/SPECS/ipcalc.spec --nodeps --nocheck
```

## dhcp

Depends on ipcalc; still need to find a way to fill in ipcalc.

Filled in ipcalc

## dracut

Build normally.

## os-prober

Build normally, generate the rpm package. **Note**: This package was not installed!
`/root/rpmbuild/RPMS/loongarch64/os-prober-1.81-9.el10~bootstrap.loongarch.loongarch64.rpm`

## grub2

Skipped, not built. Plan to replace it with grub 2.14.

## tzdata

Modify the spec file; comment out all statements related to java.
```bash
rpmbuild -bb rpmbuild/SPECS/tzdata.spec --nodeps
```

## bc

Build normally.

## dwarves

Build normally.

## libical

```bash
rpmbuild --rebuild /opt/srpms/Packages/libical-3.0.18-3.el10.src.rpm --nodeps --nocheck
```
During installation, it depends on tzdata

## libell

bluez depends on libell. Done.

## bluez

Modify the SPEC file; remove enable-cups.

## net-tools

Depends on bluez-devel. Done

## tinyxml2

Source package not found.

## cppcheck

Source package not found.

## libkcapi

```bash
rpmbuild --rebuild /opt/srpms/Packages/libkcapi-1.5.0-3.el10.src.rpm --nodeps --without doc --nocheck
```

## linux-firmware

Done building, not yet installed

## gnu-efi

Add loongarch64 to exclusivearch.

A more appropriate solution would be to add loongarch64 to the %efi rpm macro.

After the line `%ifarch x86_64 aarch64`, also add loongarch64

Done.

## inih

Done.

## userspace-rcu

Needs to be patched. Refer to the previous notes for the patch content. After patching, build normally.

## xfsprogs

Depends on inih-devel and userspace-rcu-devel; need to fill in the dependencies.

Done after filling in.

## sudo

Done.

## kernel

Skip

At this point, rebuild some packages that depend on systemd

## perl-Module-Build

```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-Module-Build-0.42.34-7.el10.src.rpm --nodeps --nocheck
```

Fill in the dependencies of utils-linux

## po4a

```bash
rpmbuild --rebuild /opt/srpms/Packages/po4a-0.69-7.el10.src.rpm --nodeps
```

## perl-YAML-Tiny

Build normally.

## perl-Module-Install

```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-Module-Install-1.21-6.el10.src.rpm --nodeps --nocheck
```

## perl-Module-Install-AuthorRequires

Build normally

## perl-File-Remove

Build normally

## perl-Module-Install-ReadmeFromPod

Failed to build successfully.

## authselect

Fill in the dependencies of utils-linux.

Because the dependencies of `perl-Module-Install-ReadmeFromPod` could not be filled in, do not build manpages.

Some .mo files will be installed, and these files are not packaged. E.g.: `/usr/share/locale/ca/LC_MESSAGES/authselect.mo`

The final build script:
```bash
rpmbuild -bb authselect.spec --nocheck --define '_unpackaged_files_terminate_build 0'
```
Installation done.

## utils-linux

Rebuild, and fill in the dependencies (as much as possible).

## dbus

No need to modify the spec file.
```bash
rpmbuild -bb dbus.spec --nodeps
```
After modifying macros.systemd, the files will be placed in the correct path.

## pcsc-lite

Just rebuild.

## polkit

Just rebuild.

## crypto-policies

Modify ExclusiveArch in the SPEC file, add Loongarch64

Repackage, build, install.

## elfutils

Rebuild and repackage; no modifications needed.

## audit

Rebuild and repackage. No more errors.

## libeconf

Build normally.

## docbook5-schemas

Build normally.

## pam

Depends on libeconf, docbook5-schemas; fill in.

Rebuild and install.

## chkconfig

Rebuild and install.

## libusers

Rebuild and install.

## shadow-utils

Rebuild and install.

## procps-ng

Rebuild and install.

## libtirpc

Rebuild and install.

## sanlock

Build and install normally; fill in the dependencies of lvm2.

## libnvme

Do not build documentation; change `-Ddocs=all -Ddoc-build=true` to `-Ddocs=false -Ddoc-build-false`

Comment out all statements related to documentation installation.
```bash
rpmbuild -bb rpmbuild/SPECS/libnvme.spec --nodeps
```

## lvm2

Rebuild and install.

## corosync

```bash
rpmbuild --rebuild /opt/srpms/Packages/corosync-3.1.9-2.el10.src.rpm --without snmp
```

## libpq

Rebuild and install.

## krb5

Still does not generate documentation. Rebuild and install.

## elinks

Rebuild and install.

## p11-kit

```bash
rpmbuild --rebuild /opt/srpms/Packages/p11-kit-0.25.5-7.el10.src.rpm --nocheck
```

## fuse3

e2fsprogs depends on fuse3.

## e2fsprogs

```bash
rpmbuild --rebuild /opt/srpms/Packages/e2fsprogs-1.47.1-4.el10.src.rpm --nocheck
```

## Configure and restart the system

```bash
sudo mv /etc/machine-id /etc/machine-id.bak
sudo mv /etc/profile /etc/profile.bak
sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/noarch/setup-*.rpm --force
```
Restart the system, get ready to enter Chapter 10.

The restart went smoothly; begin building the content of Chapter 10.

## zchunk

Source package not found.

## libsolv

Build and install normally.

## python-coverage

Source package not found.

## vim

gtk3 has not been built yet, so do not build vim's GUI.
```bash
rpmbuild --rebuild /opt/srpms/Packages/vim-9.1.083-6.el10.src.rpm --without gui --nodeps
```

## python-nose

Source package not found

## gpgme

Encountered a problem during installation. Need to disable another rpm plugin.
```bash
sudo rpm -e \
rpm-plugin-ima-4.19.1.1-20.el10~bootstrap.loongarch.loongarch64 \
rpm-plugin-dbus-announce-4.19.1.1-20.el10~bootstrap.loongarch.loongarch64
```

Build script:
```bash
QA_RPATHS=$(( 0x0001)) rpmbuild --rebuild /opt/srpms/Packages/gpgme-1.23.2-6.el10.src.rpm --without qt5 --without qt6
```

## patchutils

Just build and install directly.

## check

```bash
rpmbuild --rebuild /opt/srpms/Packages/check-0.15.2-17.el10.src.rpm --nodeps
```

## Packages that can be built easily

- python-charset-normalizer
- python-idna
- python-urllib3
- python-requests
- pyxattr

## librepo

Build error encountered during check
```text
home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:51:F:Main:test_gpg_check_signature:0: Checking valid key and data from file failed with "Error during parsing OpenPGP packets"
/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:171:F:Main:test_gpg_check_armored_key_import_test_export:0: Assertion 'subkeys != NULL' failed: subkeys == 0
/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:171:F:Main:test_gpg_check_binary_key_import_test_export:0: Assertion 'subkeys != NULL' failed: subkeys == 0
/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:283:F:Main:test_gpg_check_import_padded:0: Assertion 'tmp_err == NULL' failed: tmp_err == 0x150f61de0
```
The cause is unclear for now. Add `--nocheck` first to bypass it.

```bash
rpmbuild --rebuild /opt/srpms/Packages/librepo-1.18.0-6.el10_0.src.rpm --without pythontest --nodeps --nocheck
```

## tix

Build and install normally.

## pycairo

Build and install normally.

## pygobject

```bash
rpmbuild --rebuild /opt/srpms/Packages/pygobject3-3.46.0-7.el10.src.rpm --nodeps --nocheck
```

## python-six

Build and install normally.

## libmodulemd

Modify the SPEC file: change `-Dwith_docs=true` to `-Dwith_docs=false`

## libdnf

Modify the SPEC file as required in the book.

One test fails. Skip.
```bash
rpmbuild -bb rpmbuild/SPECS/libdnf.spec --nodeps --nocheck
```
Build and fill in some packages

## debugedit

debugedit is only installed in tools. Because /etc/profile was updated,
it no longer searches /tools/bin/ for programs, so this program can no longer be found. debugedit depends on itself, so first create a symbolic link in /usr/bin pointing to /tools/bin/debugedit to fill in the dependency.

After building and installing, /usr/bin/debugedit is no longer a symbolic link.

## pps-tools

 Build and install normally.

## chrony

 Build and install normally.

## dhcpcd

 Build and install normally. After installation, you can run dhcpcd to obtain an IP address from dhcp, so that ssh can be used normally.

## libcomps

Modify the spec file; do not build pydocs

## lua-rpm-macros

Build and install normally

## forge-srpm-macros

Build and install normally

## redhat-rpm-config

Rebuild and install, so as to correctly provide the fedora/common.lua file.

Remember to make `/usr/lib/rpm/redhat/redhat-annobin-cc1` an empty file!

## augeas

Build error encountered

```text
error: lua script failed: [string "forgemeta"]:2: module 'fedora.common' not found:
	no field package.preload['fedora.common']
	no file '/usr/lib/rpm//lua/fedora/common.lua'
	no file '/usr/lib64/lua/5.4/fedora/common.so'
	no file '/usr/lib64/lua/5.4/loadall.so'
	no file './fedora/common.so'
	no file '/usr/lib64/lua/5.4/fedora.so'
	no file '/usr/lib64/lua/5.4/loadall.so'
	no file './fedora.so'
```

After reinstalling redhat-rpm-config, it can be built and installed normally.

## Packages that do not need to be built (no source package)

- libtar
- satyr
- libmodman

## gsettings-desktop-schemas

Build and install normally.

## libproxy

A gi-docgen program is needed; it is not easy to build for now, so skip the documentation generation.

Add the `-Ddocs=false` option, and comment out the `%{_docdir}/libproxy-1.0/` line.

## xmlrpc-c

No source package.

## mailx

No source package.

## libreport

No source package.

## dnf

Modify the spec file as prompted in the book.

## composefs

Modify the SPEC file; forcibly add -Dman=disabled

## ostree

```bash
rpmbuild --rebuild /opt/srpms/Packages/ostree-2025.6-1.el10.src.rpm --nodeps
```
**Note**: ostree-grub2 is not installed.

## python-Dateutil

Remove the documentation build command.

## python-distro

Build normally.

## python3-systemd

Do not build documentation.

## dbus-python

Build normally.

## drpm

Source package not found.

## createrepo_c

```bash
rpmbuild --rebuild /opt/srpms/Packages/createrepo_c-1.1.2-4.el10.src.rpm --nodeps
```

## dnf-plugins-core

Build normally.

## mc

A handy file management tool with a terminal interface. Build normally.

## Create a local repository

Follow the instructions in the book.

You can use the rsync -auv command instead of cp to copy the files.

Modify `~/.bashrc` and set:
```bash
export PS1='[\u@\h \W]\$ '
```

## nano

Build normally.

## Create a local yum repo file

## man-pages

Build with rpmbuild and install with dnf install.

## python-psutils

Complete the dependencies for llvm as much as possible

## python-ptyprocess

Build normally.

## zsh

```bash
rpmbuild --rebuild /opt/srpms/Packages/zsh-5.9-15.el10.src.rpm --nodeps --nocheck
```

## python-pexpect

## llvm

Encountered many errors during the build, not fully resolved.

Final solution: when building runtimes, only build compiler-rt and do not build openmp. Made many changes to the spec file, which are not detailed one by one here.

After a successful build, 39 files were generated.

## execstack

execstack does not support LoongArch and will need architecture adaptation later.

## zstd

Build normally.

## liburing

Build normally.

## plocate

Build normally.

## rust

Download the LoongArch version of rust.

Download rust
1.88.0, loongarch64 version, from the official rust website. After decompressing, place it in /tools.

## openjdk

Download jdk 25.0.2 from Loongnix.cn. After decompressing, place it in /tools/.

## tzdata

Rebuild

## javapackges-tools

## Packages that do not need to be built (no source package)

- lua-lunit
- lua-posix
- copy-jdk-config

## openjdk

Skip this big package for now; you need to be well prepared before tackling this giant.

## Packages that do not need to be built (no source package)

- future
- python-commonmark
- python-recommonmark
- rust-srpm-macros

## rust

Failed. Missing libclang_rt.profile.a. After commenting out the related statements, it still hangs at

```text
compiling cc v1.2.17
Building [=>                          ] 9/85: libc(build), proc-macro2(build), generic-array(build), typenum(build)
```

No idea why this happens. Skip for now.

## Circular build script

Already copied, not tested.

## Perl family packages

Skip for now

## Python family packages

Skip for now

## A batch of packages that can be built easily

- wayland
- wayland-protocols
- hwdata
- pciutils
- libpciaccess
- libdrm
- vulkan-header
- libxxf86vm
- libXrandr
- libXdamage
- libxshmfence
- ~~libomxil-bellagio~~(source package not found)
- libglvnd
- libvdpau
- libva
- opencl-filesystem
- opencl-headers
- ocl-icd
- vulkan-loader
- spirv-headers
- glslang
- python-ply
- python3-pycparser
- mesa
- xcb-util
- xcb-util-image
- xcb-util-renderutil
- xcb-util-cursor
- xcb-util-keysyms
- xcb-util-wm
- libfontenc
- xorg-x11-font-utils
- libXfont2
- libXres
- libXv
- libdmx (source package not found)
- libepoxy
- eglexternalplatform
- egl-wayland
- lapack
- openblas-srpm-macros

## openblas

Add loongarch64 to ExclusiveArch.
During the build, encountered this error while testing:
```text
TEST 105/109 potrf:smoketest_trivial [FAIL]
  ERR: test_potrs.c:535  U s(0,0) difference: 1.19209e-07
TEST 106/109 potrf:bug_695 [OK]
TEST 107/109 kernel_regress:skx_avx [OK]
TEST 108/109 fork:safety [OK]
TEST 109/109 fork:safety_after_fork_in_parent [OK]
RESULTS: 109 tests (108 ok, 1 failed, 0 skipped) ran in 293 ms
```
Solution to the problem:

<https://blog.csdn.net/gitblog_00861/article/details/151479368>

In fact, this solution did not solve my problem.

Skip testing for now, skip this problem.

All I know is that it's a compiler problem. Set this package aside for now.

## numpy

Build normally.

## snappy

Build normally.

## wget

Build script
```bash
rpmbuild --rebuild /opt/srpms/Packages/wget-1.24.5-5.el10.src.rpm --nodeps --nocheck
```

## crash

Depends on snappy and wget, and needs to be modified to support LoongArch.

## avahi

Build script
```bash
rpmbuild  -D "version_no_tilde 0.9-rc2" -bb ~/rpmbuild/SPECS/avahi.spec
```

## libdaemon

Build normally

## dyninstall

Needs LoongArch support added. Not installed.

## xmltoman

Build normally

## systemtap

Encountered an error during the build:
```text
staplog.c:50:2: error: invalid preprocessing directive #warn; did you mean #warning?
   50 | #warn "unknown architecture for crash/staplog support"
```
Need to modify the code to support loongarch64. Add a patch.

Build script:
```bash
rpmbuild -ba -D "with_dyninst 0"  -D "with_java 0" -D "with_virthost 0" rpmbuild/SPECS/systemtap.spec --nodeps
```

## Packages that can be built easily

- yelp-xsl
- ~~ atk~~(source package not found)
- fribidi
- libdatrie
- libthai
- babel (provides python-babel)
- cups
- libexif

## libcupsfilter

```bash
rpmbuild --rebuild /opt/srpms/Packages/libcupsfilters-2.0.0-11.el10.src.rpm  --nodeps --nocheck
```

## Packages that can be built easily

- libppd
- cups-filter
- python-imagesize
- python-sphinx-theme-alabaster

## python-sphinx

Build script:
```bash
TZ=UTC rpmbuild  --rebuild /opt/srpms/Packages/python-sphinx-7.2.6-10.el10.src.rpm --nodeps --nocheck
```
Note that TZ=UTC is required

## xdg-utils

  Build normally

## latexmk

  Build normally

## gi-docgen

Depends on sphinx. Build script:
```bash
TZ=UTC rpmbuild --rebuild /opt/srpms/Packages/gi-docgen-2023.3-10.el10.src.rpm --nodeps --nocheck
```

## harfbuz

It was built once before but the build was incomplete, so rebuild it again.

## Packages that can be built easily

- python-typogrify
- python-smartypants
- pango
- libXcursor
- ~~gl-manpages~~(source package not found)
- mesa-libGLU
- freeglut
- jasper
- gdk-pixbuf2

## librsvg2

Depends on gi-docgen, depends on rust-toolset. Need to complete rust. Skip for now
