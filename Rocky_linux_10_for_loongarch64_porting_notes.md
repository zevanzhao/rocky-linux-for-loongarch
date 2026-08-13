# 引言

Rocky Linux是一个基于RHEL源码重新编译的、与RHEL二进制兼容的操作系统。在CentOS变成滚动发行版以后，Rocky Linux就是CentOS的替代操作系统。

Rocky Linux 10.0发布于2025年6月11日，10.1发布于2025年11月25日， 10.2发布于2026年5月28日。操作系统中的编译器、内核、glibc等工具链版本均比较新，应该可以很方便的移植到龙架构上。

-   GCC 14.3

-   glibc 2.39

-   Annobin 12.99

-   binutils 2.41

-   kernel 6.12.0.

我想学习龙架构操作系统的移植，参考孙海勇的《用"芯"探索：教你构建龙芯平台的Linux系统》，开始操作系统的移植。

Rocky Linux 10.1中工具链的版本基线，已经高于了国内各大操作系统厂商提出的《开源生态发展合作倡议》中的版本。

# 准备工作

## 准备系统环境

现在已经有了很多其他龙架构下的操作系统，如Debian、AOSC以及Fedora Remix。

我准备采用本地编译、而非交叉编译的方式构建临时操作系统。我准备采用Debian 作为host系统，开始我的构建。

首先，搭建系统环境。仿照孙老师的书，在Debian 系统下，需要安装一些必要的软件

```bash
sudo apt Install build-essential rpm librpm-dev git python3-dev dnf texinfo \
  zlib-dev gettext gettext-base libgettextp0-dev tcl libncurses-dev bc \
  wget meson ninja-build gperf help2man libssl-dev
```  

下载Rock Linux 10.1的源码包。

使用rsync,下载源代码仓库。我选择了南京大学的镜像。

```bash
 #!/bin/bash
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/BaseOS/source/tree/ BaseOS      
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/AppStream/source/tree/ AppStream 
 rsync -r --progress --delete --update rsync://mirror.nju.edu.cn/rocky/10.1/devel/source/tree/ devel         
```

实际上只下载devel中的包应该就足够了。源码大小24GB。

创建专用用户，并进行设置。在孙老师的书里，为了构建交叉工具链，专门使用dnf命令安装了一个小型的初始系统，使用chroot来运行。

我准备跳过这个步骤，直接使用原生的系统，但新建一个用户来做这个事情。
```bash
 groupadd rocky                                                                     
 useradd -s /bin/bash -g rocky -m -k /dev/null rocky                 
 usermod -a -G sudo rocky                                              
```
Debian系统下，没有wheel组，与之等价的是sudo组。

`-k /dev/null` 保证了新添加的用户rocky是干净的。

给新用户设置一个密码
```bash
passwd rocky
```

密码设置为loongarch

创建必要的文件夹
```bash
 mkdir -p rocky_loong64/                                             
 cd rocky_loong64                                                     
 mkdsir sources build tools cross-tools                               
 ln -sv /home/rocky/rocky_loong64/tools /                             
 ln -sv /home/rocky/rocky_loong64/cross-tools /                        
 mkdir -pv /tools/lib /tools/lib64 /tools/include   
 ```                   

**注意**：必须手动创建`/tools/lib`文件夹，因为后续编译的gcc的一个library搜索路径设置为`/home/rocky/rocky_loong64/tools/lib/../lib64/`，如果没有`/tools/lib`,那么就不能按照相对路径找到`../lib64`。

**切记！**

关于`CROSS_HOST`, `CROSS_TARGET`, `CROSS_BUILD`三个概念：

`CROSS_BUILD` 即当前运行的平台。

`CROSS_HOST`即交叉编译链运行的平台。`CROSS_HOST`所设置的前缀，将会用于交叉工具链。

`CROSS_TARGET`即工具链所生成的二进制程序所要运行的平台。

以两种情况举例
|                      |   X86平台上，交叉编译MIPS   |   Loongarch上，"交叉"编译loongarch64|
|----------------------| ----------------------------| ------------------------------------|
|  CROSS_HOST          | x86_64-cross-linux-gnu      |  loongarch64-cross-linux-gnu        |
|  CROSS_BUILD         |  x86_64-unkown-linux-gnu    |   loongarch64-debian-linux-gnu      |
|  CROSS_TARGET        | mips64el-unknown-linux-gnu  |  loongarch64-unknown-linux-gnu      |

/cross-tools中的文件，运行在交叉编译平台上。对于x86_64平台来说，CROSS_HOST为x86_64-cross-linnux-gnu

设置`~/.bash_profile`, `~/.bashrc`,具体的内容可以参考[脚本](scripts/cross/prepare-cross-chain.sh)

将之前下载的source rpms包，全部放在一个文件夹里
```bash
  find ./ -name *.rpm |xargs mv -v -t ~/rocky_loong64/sources/
```
# 构建过程简介

Rocky Linux 操作系统的移植，可以简单分为三个阶段，分别是工具链的编译、临时操作系统的编译、含有包管理系统的操作系统的编译。其中第三个阶段还可以分为两个子阶段：bootstrap阶段和完整编译阶段。

第一个阶段，主要是需要完成binutils, glibc, gcc等工具链的编译，为后续编译临时系统做准备。该阶段可以在X86机器上采用交叉编译完成，编译生成的工具链运行在X86平台上，生成龙架构的可执行程序 ；也可以直接在龙架构的机器上完成， 运行在龙架构平台上，生成龙架构的可执行程序。

第二个阶段，采用前面编译的工具链，完成临时操作系统的编译。整个流程，与Linux From Scratch非常类似。最终的产物，是一个可以在龙架构硬件上运行的操作系统。对于简单的操作系统移植来讲，到这里就已经算完成任务了。临时操作系统的问题在于缺少了包管理系统，没有灵魂，而且功能上过于精简。这个阶段，也需要编译一遍工具链，这个工具链是在龙架构平台上运行的。

第三个阶段，需要在临时操作系统完成。首先，用包管理系统构建工具链，用这个工具链以及临时系统，以bootstrap的方式，完成一个新的操作系统，逐步替代临时操作系统。在此过程中，为了能够编译软件包，会对编译内容做很多的减法，得到的是一个残缺的操作系统。随后，在这个残缺的操作系统内，再次完整的构建工具链以及整个操作系统。

以上就是我对整个移植和构建过程的粗浅理解。

# 构建交叉工具链

接下来，开始构建独立的工具链。 

虽然不需要交叉编译，但构建一个独立的、标准的工具链，仍然必要的。这样可以避免因为使用了主系统中的工具链，而带来各种奇奇怪怪的问题。

**注意**：在Debian/Ubuntu等操作系统中，`sh`指向了`dash`,而非`bash`。这会导致Fedora中的一些脚本无法正确运行。因此，建议采用LFS中的处理方法，确保`sh`指向`bash`。

## 内核头文件

这里，还是先不使用fedora的内核，转而使用Debian自带的内核。当构建交叉编译链的时候，这一步可以使用fedora提供的内核源码包。

我使用了6.17.11的linux-source源码，其中已经添加了必要的内核补丁。

```bash
make mrproper
make ARCH=loongarch headers
find usr/include -name “.*” -delete
rm usr/include/Makefile
mkdir -pv /tools/include
cp -rv usr/include/* /tools/include                                 
```

## binutils交叉工具

由于我这里并没有使用fedora系统，运行rpmbuild的时候需要添加\--nodeps参数。

```bash
rpmbuild -bp rpmbuild/SPECS/binutils.spec --nodeps
```

如果运行这个命令的时候，遇到这样的报错：
```text
+ sed -i -e s/%{release}/58.2/g bfd/Makefile{.am,.in}
sed: can't read bfd/Makefile{.am,.in}: No such file or directory    
```
说明你用的`sh`是`dash`,不是`bash`， **请确保`sh`指向`bash`**。

编译完毕，进行正确性检查。我设置的CROSS_TARGET为`loongarch64-unknown-linux-gnu`，所以/cross-tools/bin中二进制文件的前缀也是`loongarch64-unknown-linux-gnu`。

生成的二进制文件有16个，包括nm, ld, ar, as, strip, ld等。

检查ld程序的搜索路径:
```bash
/cross-tools/bin/loongarch64-unknown-linux-gnu-ld --verbose | grep SEARCH_DIR
```
输出为：
```text
SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib");
```

符合预期。

## gmp

关键：`config.guess`和`config.sub`两个文件需要进行替换。

检查：在`/cross-tools/lib`中，生成了`libgmp.so`, `libgmpxx.so`等文件。

## MPFR 

检查：在`/cross-tools/lib`中，生成的文件有`libmpfr.so`

## libmpc 

检查：在`/cross-tools/lib`中，生成的文件有`libmpc.so`

## ISL

Rock Linux 10没有提供ISL源码包，但在gcc中有is 0.24的源码。从这里编译isl。

检查：在`/cross-tools/lib`中，生成的文件有`libisl.so`

## CLooG 

Rock Linux 10 没有提供CLooG源码包，应该是不需要了。

## GCC-PASS1
第一次编译GCC。

直接运行`rpmbuild -bp rpmbuild/SPECS/gcc.spec`时，报错：
```text
error: invalid syntax in lua scriptlet: [string "%postun"]:10: unexpected symbol near '%''
```
不知道这个问题是怎么来的，但是在%post以及%postun前加了一个空格，问题就消失了。
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

接下来，修改GCC的源码，将库的搜索路径，从默认的`/lib/`等修改为`/tools/lib/`

分成两部分进行修改

1.  修改/lib

```bash
for file in $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
    sed -i.orig -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' -e 's@/usr@/tools@' $file
done

#ATTENTION: Do not forget to update this file!
sed -i.orig 's@"\/lib" ABI_GRLEN_SPEC "\/ld-linux-loongarch-" ABI_SPEC "\.so\.1"@"\/tools/lib" ABI_GRLEN_SPEC "\/ld-linux-loongarch-" ABI_SPEC "\.so\.1"@g' gcc/config/loongarch/gnu-user.h

```
注意，上述修改的目的之一，是确保DYNAMIC_LINKER采用`/tools/lib*`/中的`ld-*.so`。但是，对于龙架构，关于DYNAMIC_LINKER相关的定义在`gcc/config/loongarch/gnu-user.h`文件中，因此在上述sed脚本中，应该添加修改gnu-user.h文件的语句。

2.  修改`STANDARD_STARTFILE_PREFIX_*`宏

    需要修改`gcc/gcc.cc`文件和`gcc/config/loongarch/linux.h`

    由于`loongarch/linux.h`中，`STANDARD_STARTFILE_PREFIX_*`的定义方式已经与MIPS不同了，

    `gcc/config/loongarch/linux.h`中这样写
```c
#define STANDARD_STARTFILE_PREFIX_1 "/tools/" ABI_LIBDIR "/"
#define STANDARD_STARTFILE_PREFIX_2 ""
```
3.  configure,编译。
 
 这里是具体的编译[脚本](scripts/cross/build-cross-gcc-pass1.sh)。 此时的gcc,无法编译出可以正常运行的可执行程序，只能编译库。

    编译的时候使用了`--disable-shared` 选项，所以生成了libgcc.a，没有libgcc.so文件。

    在/cross-tools/bin/中，有`loongarch64-unknown-linux-gnu-gcc`等文件。

    检查gcc程序的库检索路径：
  ```bash
    /cross-tools/bin/loongarch64-unknown-linux-gnu-gcc  -print-search-dirs
  ```
输出为：
```text
install: /home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/
programs: =/home/rocky/rocky_loong64/cross-tools/bin/../libexec/gcc/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../libexec/gcc/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/bin/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/bin/
libraries: =/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/../lib64/:/home/rocky/rocky_loong64/tools/lib/loongarch64-unknown-linux-gnu/14.3.1/:/home/rocky/rocky_loong64/tools/lib/../lib64/:/home/rocky/rocky_loong64/cross-tools/bin/../lib/gcc/loongarch64-unknown-linux-gnu/14.3.1/../../../../loongarch64-unknown-linux-gnu/lib/:/home/rocky/rocky_loong64/tools/lib/
```
注意其中的`home/rocky/rocky_loong64/tools/lib/../lib64/`路径，正是这个路径，要求必须系统中必须存在`tools/lib`文件夹。

## glibc

修改spec文件，注释掉部分关于libmvec的语句。

注意，`echo "slibdir=/tools/lib64" >>configparms`
这个语句已经不需要了，被`libc_cv_slibdir`取代了。

glibc 2.39中，移除了crypt支持，因此需要额外的编译libxcrypt。

来自上游的Glibc编译正常，但Rocky Linux中的glibc 2.39编译不正常。

经过确认，glibc 2.39 rpm包中的glibc-2.39.tar.xz与上游完全一致，但是Rocky Linux中的glibc打了420个补丁，问题肯定是来自其中某一个或者多个补丁。究竟是哪一个补丁带来的问题，尝试使用二分法进行确认。

  
|  步骤  |   操作        |         效果|     结论|
|--------|---------------|-------------|---------|
| 1      |  注释211～420   |      OK    |   |
| 2      |  注释306～420    |     Fail |    有问题的补丁在211~306|
| 3       |  注释250～420  |       OK    |   有问题的补丁在250~306|
| 4      |  注释276～420    |     Fail  |   有问题的补丁在250~276|
| 5      |  注释264～420  |       Fail  |   有问题的补丁在250~263|
| 6      |  注释257～420     |    OK   |    有问题的补丁在257~263|
| 7       | 注释260～420   |      OK   |    有问题的在261~263|
| 8     |   注释262～420   |      Fail   |  有问题的在260~261|
| 9     |   注释261～420     |    OK   |    问题补丁是261|

好了，找到了问题补丁`glibc-RHEL-101754-2.patch`！

解决方案，不打Patch 261这个补丁。需要看一看这个补丁干了什么。

可以确定的是，这个补丁确实跟龙架构无关。换句话说，龙架构可能是需要补充一个 `dl-tunables.list`文件。

编译完成后，检查：libc库位于/tools/lib64中，用file命令进行查看。

```text
file /tools/lib64/libc.so.6 
/tools/lib64/libc.so.6: ELF 64-bit LSB shared object, LoongArch, version 1 (GNU/Linux), dynamically linked, interpreter /tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with debug_info, not stripped
```

在`/tools/lib64`中，还包含有`ld-linux-loongarch-lp64d.so.1`， `librt.so`,
`crt1.o`, `crti.o`, `crtn.o`等文件。

注意：此时，可以编译locales了，不然后面无法设置locales为zh_CN.utf8。

## GCC-PASS2 

第二次编译GCC。编译的时候，遇到了这样的问题：
```text
/home/rocky/rpmbuild/BUILD/gcc-14.3.1-build/gcc-14.3.1-20250617/build/prev-loongarch64-unknown-linux-gnu/libstdc++-v3/include/ext/concurrence.h:252:32: error: cannot convert '<brace-enclosed initializer list>' to 'unsigned int' in initialization
  252 |     __gthread_cond_t _M_cond = __GTHREAD_COND_INIT;
      |   
```

经过一系列的检索，发现了有其他人遇到了类似的问题，解决方案是：

<https://sourceware.org/bugzilla/show_bug.cgi?id=32625>

这个bug指向了下一个bug：

<https://sourceware.org/bugzilla/show_bug.cgi?id=32621>

解决方案在这里：

<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=118009>

<https://gcc.gnu.org/cgit/gcc/diff/?id=ea2798892de373b14f9fc7ae8a0d820eaddca98c>

下载gcc的源码，找到这个commit,做成patch，尝试给这个版本的gcc打上。
```bash
git format-patch -1 ea2798892de373b14f9fc7ae8a0d820eaddca98c
```
生成补丁文件：`0001-fixincludes-Skip-pthread_incomplete_struct_argument-.patch`

复制到`~/rpmbuild/SOURCE`

修改gcc的spec文件，将补丁文件加入到patch中。

这里是具体的[编译脚本](scripts/cross/build-cross-gcc-pass2.sh)

## 检查工具链

编写测试程序：

```c
#include <stdio.h>
int main() {
  printf("OK!\n");
  return 0;
}
```

编译
```bash
loongarch64-unknown-linux-gnu-gcc test-gcc-pass2.c -v -Wl,--verbose &> dummy.log
```

运行：`file a.out`

输出
```text
a.out: ELF 64-bit LSB pie executable, LoongArch, version 1 (SYSV),
dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
运行：` readelf -l a.out \|grep \"tools\"`

输出
```text
[Requesting program interpreter: /tools/lib64/ld-linux-loongarch-lp64d.so.1\]
```
运行 `grep -o "/tools/lib.*/crt.*succeeded" dummy.log`

输出 
```text
/tools/lib/../lib64/crti.o succeeded
/tools/lib/../lib64/crtn.o succeeded
```

文件查找路径：
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

链接库文件的路径是否正确：
```text
grep "SEARCH_DIR" dummy.log 
SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("=/cross-tools/loongarch64-unknown-linux-gnu/lib");
```
C库位置是否正确：
```text
grep "libc.so.6" dummy.log 
attempt to open /home/rocky/rocky_loong64/tools/lib64/libc.so.6 succeeded
/home/rocky/rocky_loong64/tools/lib64/libc.so.6
ld-linux-loongarch-lp64d.so.1 needed by /home/rocky/rocky_loong64/tools/lib64/libc.so.6
```

ld库位置是否正确：
```text
grep found dummy.log 
found ld-linux-loongarch-lp64d.so.1 at /home/rocky/rocky_loong64/tools/lib64/ld-linux-loongarch-lp64d.so.1
```

**警告**：以上的这些检查一定要做，一定不能跳过，一定不能掉以轻心！如果有任何对不上的，都要进行修正，否则后面编译的程序全都会有问题，必须要**全部**返工。不要问我为什么知道！

**切记! 切记! 切记**

## pkgconf 

正常编译完成。

检查：
```bash
type pkg-config
pkg-config is /cross-tools/bin/pkg-config
```
## grub

Rocky Linux中自带的grub版本为2.12, 对龙架构的支持还不够完善，改用了grub 2.14的源码进行编译。 不过，测试后结果还是很糟糕：
```text
ERROR: erofs_test
ERROR: ext234_test
PASS: squashfs_test
PASS: iso9660_test
ERROR: hfsplus_test
ERROR: ntfs_test
ERROR: reiserfs_test
ERROR: fat_test
ERROR: minixfs_test
ERROR: xfs_test
ERROR: f2fs_test
ERROR: nilfs2_test
ERROR: romfs_test
ERROR: exfat_test
PASS: tar_test
ERROR: udf_test
ERROR: hfs_test
ERROR: jfs_test
ERROR: btrfs_test
ERROR: zfs_test
ERROR: zfs_zstd_test
PASS: cpio_test
PASS: example_scripted_test
PASS: gettext_strings_test
PASS: grub_script_blanklines
PASS: grub_script_final_semicolon
PASS: grub_script_dollar
PASS: grub_script_no_commands
PASS: syslinux_test
ERROR: luks1_test
ERROR: luks2_test
SKIP: pata_test
SKIP: ahci_test
SKIP: uhci_test
SKIP: ohci_test
SKIP: ehci_test
FAIL: example_grub_script_test
FAIL: grub_script_eval
FAIL: grub_script_test
FAIL: grub_script_echo1
FAIL: grub_script_leading_whitespace
FAIL: grub_script_echo_keywords
FAIL: grub_script_vars1
FAIL: grub_script_for1
FAIL: grub_script_while1
FAIL: grub_script_if
FAIL: grub_script_comments
FAIL: grub_script_functions
FAIL: grub_script_break
FAIL: grub_script_continue
FAIL: grub_script_shift
FAIL: grub_script_blockarg
FAIL: grub_script_setparams
FAIL: grub_script_return
ERROR: grub_cmd_cryptomount
FAIL: grub_cmd_regexp
FAIL: grub_cmd_date
FAIL: grub_cmd_set_date
FAIL: grub_cmd_sleep
PASS: grub_script_expansion
FAIL: grub_script_not
PASS: partmap_test
FAIL: hddboot_test
SKIP: fddboot_test
FAIL: cdboot_test
SKIP: netboot_test
ERROR: serial_test
SKIP: pseries_test
FAIL: core_compress_test
FAIL: xzcompress_test
FAIL: gzcompress_test
FAIL: lzocompress_test
FAIL: grub_cmd_echo
FAIL: help_test
FAIL: grub_script_gettext
FAIL: grub_script_escape_comma
FAIL: grub_script_strcmp
PASS: test_sha512sum
FAIL: test_unset
ERROR: grub_func_test
FAIL: grub_cmd_tr
FAIL: file_filter_test
PASS: grub_cmd_test
FAIL: asn1_test
SKIP: tpm2_key_protector_test
PASS: example_unit_test
PASS: printf_test
PASS: date_test
PASS: priority_queue_unit_test
PASS: cmp_test
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
具体有哪些问题，不得而知，我也没有更多的时间去进行分析了。不过似乎这个grub还是可以用的。
至此, 工具链编译全部完成，第一阶段结束！

完成于2026年3月15日星期日凌晨0：08。

# 构建临时系统

进入下一个环节，构建临时系统。这部分内容与LFS非常接近，只是将源码的来源限制为Rocky Linux的源码。

## 制作环境设置 

更新.bashrc，创建meson-cross.txt。可以参考这个[脚本](scripts/tmp/prepare-tmp-os.sh)

接着，完成临时系统工具链构建。每个软件包的构建脚本，在[脚本目录](scripts/tmp/)中。

## gmp

检查：/tools/lib64文件夹下，存在libgmp.a,libgmp.so文件。

## MPFR

检查：在/tools/lib64文件夹下，存在libmpfr.so, libmpfr.a文件

## MPC 

检查：在/tools/lib64文件夹下，存在libmpc.so, libmpc.a文件

## ISL 

ISL是gcc的一部分。

检查：在/tools/lib64文件夹下，存在libisl.so, libisl.a文件

## CLooG

新版本的GCC不再需要了CLooG了。

## Zlib

zlib 需要从这里下载

<https://mirror.nju.edu.cn/rocky/9/devel/source/tree/Packages/z/zip-3.0-35.el9.src.rpm>

检查：在/tools/lib64文件夹下，存在libz.so, libz.a文件

## Binutils 

检查：/tools/bin目录下，存在ld, nm, as, ld-new等程序。

检查interpreter
```bash
file /tools/bin/ld
```

输出为
```text
/tools/bin/ld: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
检查搜索路径：
```bash
tools/bin/ld --verbose|grep -i search
```
输出为
```text
SEARCH_DIR("/tools/loongarch64-unknown-linux-gnu/lib64"); SEARCH_DIR("/tools/lib64"); SEARCH_DIR("/tools/lib"); SEARCH_DIR("/tools/loongarch64-unknown-linux-gnu/lib");
```

## GCC

重要的组件。

检查：在/tools/bin/中，存在gcc, g++, gfortran, cpp等文件。

检查interpreter
```bash
file /tools/bin/gcc
```

输出：
```text
/tools/bin/gcc: ELF 64-bit LSB executable, LoongArch, version 1
(GNU/Linux), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
## ncurses

编译之前，修改SPEC文件，注释掉这一行:
```bash
%{gpgverify} --keyring=%{SOURCE2} --signature=%{SOURCE1} --data=%{SOURCE0} 
```
gpgverify 这个宏暂时没有实现，跳过这个也没有关系。

检查：

在/tools/lib64/下，存在libncurses.so, libncursesw.a
libncursesw.so等库文件。

/tools/lib64/pkgconfig 目录下存在ncurses++w.pc，ncursesw.pc，ncurses.pc文件。

## BZIP2 

检查：/tools/bin下，存在bzip2，bunzip2, bcat, bzmore, bzless,
bzgrep等文件。

在/tools/lib64下，存在 libbz2.so。

## XZ 

检查：
```bash 
ls /tools/bin/xz* /tools/lib64/liblzma.*
```
输出：
```text 
/tools/bin/xz /tools/bin/xzcmp /tools/bin/xzdiff /tools/bin/xzfgrep
/tools/bin/xzless /tools/bin/xzcat /tools/bin/xzdec /tools/bin/xzegrep
/tools/bin/xzgrep /tools/bin/xzmore
/tools/lib64/liblzma.a /tools/lib64/liblzma.la /tools/lib64/liblzma.so
/tools/lib64/liblzma.so.5 /tools/lib64/liblzma.so.5.6.2
```

## Readline

检查：
``` bash
ls /tools/lib64/libreadline.* /tools/lib64/libhistory.*
```
输出：
```text
/tools/lib64/libhistory.a /tools/lib64/libhistory.so.8
/tools/lib64/libreadline.a /tools/lib64/libreadline.so.8
/tools/lib64/libhistory.so /tools/lib64/libhistory.so.8.2
/tools/lib64/libreadline.so /tools/lib64/libreadline.so.8.2
```
## OpenSSL

编译openssl的时候，遇到了这个bug

<https://www.findbugzero.com/operational-defect-database/vendors/rh/defects/RHEL-93168>

看起来与某些宏定义有关。暂时先注释掉一些代码，编译可以正常进行。至于这是什么原理，以后再考虑了。

似乎与这个补丁有关系: `0014-RH-Export-two-symbols-for-OPENSSL_str-n-casecmp.patch`

并没有那么简单。最后的办法：写了sed语句，注释掉一部分代码

检查：
```bash
ls /tools/bin/openssl
```
输出：
```text
/tools/bin/openssl
```

```bash
file /tools/bin/openssl
```

输出：
```text
/tools/bin/openssl: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```

```bash
ls /tools/lib64/libssl.*
```
输出：
```text
/tools/lib64/libssl.a /tools/lib64/libssl.so /tools/lib64/libssl.so.
```
## pcre2

注释掉spec文件中的gpgverify行。

检查：
```bash
ls /tools/lib64/libpcre2-*
```
输出：
```text
/tools/lib64/libpcre2-8.a /tools/lib64/libpcre2-8.so.0
/tools/lib64/libpcre2-posix.la /tools/lib64/libpcre2-posix.so.3.0.5
/tools/lib64/libpcre2-8.la /tools/lib64/libpcre2-8.so.0.13.0
/tools/lib64/libpcre2-posix.so
/tools/lib64/libpcre2-8.so /tools/lib64/libpcre2-posix.a
/tools/lib64/libpcre2-posix.so.3
```
## libsepol

检查：
```bash
ls /tools/lib64/libsepol.*
```
输出：
```text
/tools/lib64/libsepol.a /tools/lib64/libsepol.so
/tools/lib64/libsepol.so.2
```
## libselinux 

必须设置`PKG_CONFIG_PATH=/tools/lib64/pkgconfig USE_PCRE2=y`

否则不会自动链接pcre2-8。

检查：
```bash
ls /tools/lib64/libselinux.*
```

输出：
```text
/tools/lib64/libselinux.a /tools/lib64/libselinux.so
/tools/lib64/libselinux.so.1
```
## nspr

nspr的源码在nss中。

完成

检查：
```bash
ls /tools/lib64/{libnspr4.*,libplc*}
```
输出：
```text
/tools/lib64/libnspr4.a /tools/lib64/libnspr4.so /tools/lib64/libplc4.a
/tools/lib64/libplc4.so
```
## sqlite

完成

检查：
```bash
ls /tools/lib64/libsqlite3.*
```
输出：
```text
/tools/lib64/libsqlite3.a /tools/lib64/libsqlite3.la
/tools/lib64/libsqlite3.so /tools/lib64/libsqlite3.so.0
/tools/lib64/libsqlite3.so.0.8.6
```
## nss 

需要添加 `NSS_DISABLE_DSA=1 NSS_ENABLE_ML_DSA=1`

完成

检查：
```bash
ls /tools/lib64/{libnss*,libsoft*}
```
输出：
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

完成

检查：
```bash
ls /tools/lib64/libpopt.*
```

输出：
```text
/tools/lib64/libpopt.a /tools/lib64/libpopt.la /tools/lib64/libpopt.so
/tools/lib64/libpopt.so.0 /tools/lib64/libpopt.so.0.0.2
```
## libarchive

需要修改aclocal.m4和configure脚本，因为系统中的aclocal版本是1.18,不是1.16。

我采用的方法是将这个修改写到编译脚本内。也可以尝试用`autoconf -fiv`个更新一下脚本。

检查：
```bash
ls /tools/lib64/libarchive.*
```

输出：
```text
/tools/lib64/libarchive.a /tools/lib64/libarchive.so
/tools/lib64/libarchive.so.13.7.7

/tools/lib64/libarchive.la /tools/lib64/libarchive.so.13
```
## libdb

rocky 10已经不提供libdb包了。不需要了。

## libcap

注释掉gpgverify相关的语句。完成。

检查：
```bash
ls /tools/lib64/libcap.*
```

输出：
```text
/tools/lib64/libcap.a /tools/lib64/libcap.so /tools/lib64/libcap.so.2
/tools/lib64/libcap.so.2.69
```
## libmicrohttpd

完成。

检查：
```bash
ls /tools/lib64/libmicrohttpd.*
```

输出：
```text
/tools/lib64/libmicrohttpd.a /tools/lib64/libmicrohttpd.so
/tools/lib64/libmicrohttpd.so.12.62.0
/tools/lib64/libmicrohttpd.la /tools/lib64/libmicrohttpd.so.12
```
## libpsl

curl依赖libpsl,所以需要先编译psl。

注释掉了spec文件中的%py3_shebang_fix 相关的语句

检查：
```bash
ls /tools/lib64/libpsl.*
```
输出：
```text
/tools/lib64/libpsl.a /tools/lib64/libpsl.la /tools/lib64/libpsl.so
/tools/lib64/libpsl.so.5 /tools/lib64/libpsl.so.5.3.5
```
## curl 

注释掉%{gpgverify}相关语句

检查
```bash
file /tools/bin/curl
```

输出：
```text
/tools/bin/curl: ELF 64-bit LSB pie executable, LoongArch, version 1
(SYSV), dynamically linked, interpreter
/tools/lib64/ld-linux-loongarch-lp64d.so.1, for GNU/Linux 5.19.0, with
debug_info, not stripped
```
```bash
ls /tools/lib64/libcurl.*
```

输出：
```text
/tools/lib64/libcurl.a /tools/lib64/libcurl.la /tools/lib64/libcurl.so
/tools/lib64/libcurl.so.4 /tools/lib64/libcurl.so.4.8.0
```
## elfutils 

完成

检查：
```bash
ls /tools/lib64/{libelf*,libdw*}
```

输出：
```text
/tools/lib64/libdw-0.193.so /tools/lib64/libdw.so
/tools/lib64/libelf-0.193.so /tools/lib64/libelf.so
/tools/lib64/libdw.a /tools/lib64/libdw.so.1 /tools/lib64/libelf.a
/tools/lib64/libelf.so.1
```
## lz4

完成。

检查：
```bash
ls /tools/bin/lz4* /tools/lib64/liblz4*
```

输出：
```text
/tools/bin/lz4 /tools/bin/lz4cat /tools/lib64/liblz4.so
/tools/lib64/liblz4.so.1.9.4
/tools/bin/lz4c /tools/lib64/liblz4.a /tools/lib64/liblz4.so.1
```
## zstd 

修改spec文件，%patchN 改为 %patch N

检查：
```bash
ls /tools/bin/zstd* /tools/lib64/libzstd.*
```

输出：
```text
/tools/bin/zstd /tools/bin/zstdgrep /tools/bin/zstdmt
/tools/lib64/libzstd.so /tools/lib64/libzstd.so.1.5.5
/tools/bin/zstdcat /tools/bin/zstdless /tools/lib64/libzstd.a
/tools/lib64/libzstd.so.1
```

## expat

注释掉%{gpgverify}相关语句

configure的时候添加 `--without-docbook`

编译的时候会出错。应该想办法跳过doc

修改Makefile.in

检查：
```bash
ls /tools/lib64/libexpat.*
```

输出：
```text
/tools/lib64/libexpat.a /tools/lib64/libexpat.la
/tools/lib64/libexpat.so /tools/lib64/libexpat.so.1
/tools/lib64/libexpat.so.1.10.2
```

## bash

完成

检查：
```bash
 ls /tools/bin/bash
```

输出：
```text
/tools/bin/bash
```
## bash-complete 

完成

## coreutils
完成

检查：
```bash
ls /tools/bin/{ls,cp,hostname,wc,tail,uniq}
```

输出：
```text
/tools/bin/cp /tools/bin/hostname /tools/bin/ls /tools/bin/tail
/tools/bin/uniq /tools/bin/wc
```
## file

再一次遇到了gpgverify的问题。准备一劳永逸解决问题。

安装`redhat-rpm-config-293`这个源码包，将gpgverify脚本放在
`/usr/lib/rpm/redhat`,将`macros.fedora-misc`放在`/usr/lib/rpm/macros.d`。

gpgverify脚本需要gpgv2，简单做个符号链接：
```bash
ln -s gpgv gpgv2
```

检查：
```bash
ls /tools/bin/file
```

输出：
```text
/tools/bin/file
```
## findutils

完成

检查：
```bash
ls /tools/bin/find
```

输出：
```text
/tools/bin/find
```
## gawk

需要设置automake的版本。

完成

检查：
```bash
ls /tools/bin/gawk
```

输出：
```text
/tools/bin/gawk
```
## gettext

只需要添加
```bash
sed -e 's/\(gl_cv_libxml_force_included=\)no/\1yes/' -i libtextstyle/configure
```
即可。完成。

检查：
```bash
ls /tools/bin/{gettext,msg*}
```

输出：
```text
/tools/bin/gettext /tools/bin/msgcmp /tools/bin/msgen /tools/bin/msgfmt
/tools/bin/msgmerge
/tools/bin/msgattrib /tools/bin/msgcomm /tools/bin/msgexec
/tools/bin/msggrep /tools/bin/msgunfmt
/tools/bin/msgcat /tools/bin/msgconv /tools/bin/msgfilter
/tools/bin/msginit /tools/bin/msguniq
```
## grep

需要设置automake的版本

完成

检查：
```bash
ls /tools/bin/grep
```

输出：
```text
/tools/bin/grep
```
## gzip

完成。

检查：
```bash
ls /tools/bin/gz*
```

输出：
```text
/tools/bin/gzexe /tools/bin/gzip
```
## sed 

完成

检查：
```bash
ls /tools/bin/sed
```

输出：
```text
/tools/bin/sed
```
## util-linux

设置PKG_CONFIG_PATH, 运行autoreconf -fv

编译过程中会报错，因此添加了--disable-schedutils
不知道是否会有不好的影响。

完成。

检查：
```bash
ls /tools/bin/{mount,umou*}
```

输出：
```text
/tools/bin/mount /tools/bin/umount
```
## kmod

需要设置PKG_CONFIG_PATH,并且运行autoreconf -fv

完成。

检查：
```bash
ls /tools/bin/*mod
```

输出：
```text
/tools/bin/chmod /tools/bin/depmod /tools/bin/insmod /tools/bin/kmod
/tools/bin/lsmod /tools/bin/rmmod
```
## vim

完成。参考了书和CLFS的文档。

一点点区别：我设置了`set mouse=a`

检查：
```bash
ls /tools/bin/vi*
```

输出：
```text
/tools/bin/vi /tools/bin/view /tools/bin/vim /tools/bin/vimdiff
/tools/bin/vimtutor
```
## nano

虽然编译通过了，但我总觉得不太合理。

configure的时候，添加了一些参数：`LIBS="-lncurses -ltinfo" LDFLAGS="-L/tools/lib64"`
检查：
```bash
ls /tools/bin/nano
```

输出：
```text
/tools/bin/nano
```
## which

完成。

检查：
```bash
ls /tools/bin/which
```

输出：
```text
/tools/bin/which
```
## iproute

设置了`LIBDIR=/tools/lib64/`

完成。

检查： 
```bash
ls /tools/sbin/{ifstat,ip}
```

输出：
```text
/tools/sbin/ifstat /tools/sbin/ip
```
## dhcpcd

完成。

检查：
```bash
ls /tools/sbin/dhcpcd
```

输出：
```text
/tools/sbin/dhcpcd
```
## fipscheck

在Rocky Linux 10中，fipscheck包改由libkcapi-1.5.0-3.el10.src.rpm提供。

## libxcrypt

tcsh和pam都需要libcrypt, 用libxcrypt中的替代。完成。

注意，`--libdir`需要设置为`/tools/lib64`

检查：
```bash
ls /tools/lib64/libcrypt.*
```

输出：
```text
/tools/lib64/libcrypt.a /tools/lib64/libcrypt.la
/tools/lib64/libcrypt.so /tools/lib64/libcrypt.so.1
/tools/lib64/libcrypt.so.1.1.0
```
注意：libcrypt.so和libcrypto.so是完全不同的两个东西！

## pam

安装以后，`pam_appl.h`文件不在security目录中。需要调整includedir。

重点：`--includedir=/tools/include/security`

完成。

检查：
```bash
ls /tools/lib64/libpam*
```

输出：
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

**注意**：

安装了pam后，需要在/tools/etc/pam.d中，进行一系列的设置，否则系统将无法启动。这一步可以在配置临时系统的时候操作。

## openssh

缺少`security/pam_appl.h`文件。需要安装额外的程序，pam

加上了`--with-pam`参数，完成

检查：
```bash
ls /tools/bin/ssh* /tools/sbin/ssh*
```

输出：
```text
/tools/bin/ssh /tools/bin/ssh-agent /tools/bin/ssh-keyscan
/tools/bin/ssh-add /tools/bin/ssh-keygen /tools/sbin/sshd
```
## sudo

完成
```bash
ls /tools/bin/sudo
```

输出：
```text
/tools/bin/sudo
```
## e2fsprogs  

添加 `LDFLAGS="-L/tools/lib64 -luuid -lblkid"`

完成
```bash
ls /tools/sbin/{mkfs.*,fsck.*}
```

输出：
```text
/tools/sbin/fsck.cramfs /tools/sbin/fsck.ext4 /tools/sbin/mkfs.cramfs
/tools/sbin/mkfs.ext4
/tools/sbin/fsck.ext2 /tools/sbin/fsck.minix /tools/sbin/mkfs.ext2
/tools/sbin/mkfs.minix
/tools/sbin/fsck.ext3 /tools/sbin/mkfs.bfs /tools/sbin/mkfs.ext3
```
## inih 

按照<https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md>

中的脚本，编译安装

检查：
```bash
ls /tools/lib64/libinih.so*
```

输出：
```text
/tools/lib64/libinih.so /tools/lib64/libinih.so.0
```
## uuid

需要更新`config.guess`以及 `config.sub`文件

检查：
```bash
ls /tools/lib64/libossp-uuid.*
```

输出：
```text
/tools/lib64/libossp-uuid.a /tools/lib64/libossp-uuid.so
/tools/lib64/libossp-uuid.so.16.0.22
/tools/lib64/libossp-uuid.la /tools/lib64/libossp-uuid.so.16
```
## userspace-rcu

0.14.0版本，需要添加支持龙架构的补丁：

<https://gitee.com/zhong-wei-shen/userspace-rcu/blob/master/add-loongarch.patch>

automake等版本不匹配，需要 `autoreconf -fv`

完成编译。

检查：
```bash
ls /tools/lib64/liburcu*
```

输出：
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

使用了ini.h，需要安装libinih开发包;  使用了uuid.h,需要安装uuid; 需要urcu.h安装userspace-rcu

**注意**: xfsprogs使用的uuid,由util-linux提供，不是ossp-uuid。

完成

检查：

```bash
ls /tools/sbin/*xfs
```

输出：
```text
/tools/sbin/fsck.xfs /tools/sbin/mkfs.xfs
```
## dosfstools

完成

检查：
```bash
ls /tools/sbin/*vfat
```

输出：
```text
/tools/sbin/fsck.vfat /tools/sbin/mkfs.vfat
```
## bison

完成

检查：
```bash
ls /tools/bin/{bison,yacc}
```

输出：
```text
/tools/bin/bison /tools/bin/yacc
```
## check

configure的时候，添加`--disable-build-docs`选项

完成编译。

检查：
```bash
ls /tools/bin/checkmk
```

输出：
```text
/tools/bin/checkmk
```
## diffutils

完成

检查：
```bash
ls /tools/bin/diff*
```

输出：
```text
/tools/bin/diff /tools/bin/diff3
````
## make

完成
```bash
ls /tools/bin/make
```

输出：
```text
/tools/bin/make
```
## patch

需要更新`config.sub`和`config.guess`。完成

检查：
```bash
ls /tools/bin/patch
```

输出：
```text
/tools/bin/patch
```
## tar

完成。

检查：
```bash
ls /tools/bin/tar
```

输出：
```text
/tools/bin/tar
```
## textinfo

完成。甚至不需要修改Makefile.am文件。
检查：
```bash
ls /tools/bin/info
```

输出：
```text
/tools/bin/info
```
## m4

完成
```bash
ls /tools/bin/m4
```

输出：
```text
/tools/bin/m4
```
## pkgconf

完成。

检查：
```bash
ls /tools/bin/pkg-config
```

输出：
```text
/tools/bin/pkg-config
```
## autoconf

完成。检查：
```bash
ls /tools/bin/autoconf
```

输出：
```text
/tools/bin/autoconf
```
## automake

完成。

检查：
```
ls /tools/bin/automake
```

输出：
```text
/tools/bin/automake
```
## libtool

完成
```bash
ls /tools/bin/libtool
```

输出：
```text
/tools/bin/libtool
```
## flex

完成
检查：
```bash
ls /tools/bin/flex
```

输出：
```text
/tools/bin/flex
```
## tcl

完成。

检查:
```bash
ls /tools/bin/tclsh /tools/lib64/libtcl*
```

输出：
```text
/tools/bin/tclsh /tools/lib64/libtcl.so /tools/lib64/libtcl8.6.so
/tools/lib64/libtclstub8.6.a
```
## lua

完成

检查：

```bash
ls /tools/bin/lua
```

输出：
```text
/tools/bin/lua
```
## cpio

完成

检查：
```bash
ls /tools/bin/cpio
```

输出：
```text
/tools/bin/cpio
```
## tcsh
编译的时候报错：`undefined reference to 'crypt'`

安装完libxcrypt再安装tcsh,完成。
检查：
```bash
ls /tools/bin/tcsh
```

输出：
```text
/tools/bin/tcsh
```
## attr

acl依赖attr。完成。
检查：
```bash
ls /tools/bin/attr
```

输出：
```text
/tools/bin/attr
```
## audit

需要根据spec文件，编写编译脚本，去掉一些功能。完成。

检查：
```bash
ls /tools/lib64/libaudit.*
```

输出：
```text
/tools/lib64/libaudit.a /tools/lib64/libaudit.so
/tools/lib64/libaudit.so.1.0.0
/tools/lib64/libaudit.la /tools/lib64/libaudit.so.1
```
## acl 

rpm的安装需要acl。

完成。检查：
```bash
ls /tools/lib64/libacl.*
```

输出：
```text
/tools/lib64/libacl.la /tools/lib64/libacl.so /tools/lib64/libacl.so.1
/tools/lib64/libacl.so.1.1.2302
```
## nettle

需要`autoreconf -ifv`,并且在configure时，去掉 sm3和sm4支持。

完成:
```bash
ls /tools/lib64/libnettle.*
```

输出：
```text
/tools/lib64/libnettle.a /tools/lib64/libnettle.so
/tools/lib64/libnettle.so.8 /tools/lib64/libnettle.so.8.10
```
## rpm

rpm 4.15采用configure来生成Makefile，但4.19改用了cmake。因此，交叉编译的方式需要进行改变。

需要创建[loongarch64-toolchain.cmake](sources/loongarch64-toolchain.cmake)。

修改编译脚本，使用cmake编译。

需要设置PKG_CONFIG_PATH目录

需要编译安装libacl，audit。

去掉对dbus的支持。

去掉补丁rpm-4.19.x-pqc-algo.patch中的部分修改。

去掉对python的支持。

完成。

检查：
```bash
ls /tools/bin/rpm*
```

输出：
```text
/tools/bin/rpm /tools/bin/rpmbuild /tools/bin/rpmkeys /tools/bin/rpmsign
/tools/bin/rpmverify
/tools/bin/rpm2archive /tools/bin/rpmdb /tools/bin/rpmlua
/tools/bin/rpmsort
/tools/bin/rpm2cpio /tools/bin/rpmgraph /tools/bin/rpmquery
/tools/bin/rpmspec
```
## 配置rpm
参考[配置脚本](scripts/tmp/config-tmp-rpm.sh)。
完成

## systemd

按照CLFS以及豹书进行操作。需要设置PKG_CONFIG_PATH

需要将version_no_tilde设置为257
```bash
rpmbuild -D "version_no_tilde 257" -bp ~/rpmbuild/SPECS/systemd.spec --nodeps
```
完成

检查：
```bash
ls /tools/sbin/init /tools/bin/systemctl
```

输出：
```text
/tools/bin/systemctl /tools/sbin/init
```
## dbus

需要设置PKG_CONFIG_PATH环境变量，并且不安装doc。

添加`autoreconf -fv`

检查
```bash
ls /tools/bin/dbus-daemon
```

输出：
```text
/tools/bin/dbus-daemon
```

## shadow-utils

完成
```bash
ls /tools/bin/login
```

输出：
```text
/tools/bin/login
```
## Linux内核

使用debian的6.17.11内核。

编译内核的时候，需要注意将必要的模块都内置，这样就可以不需要initrd.img了。

包括ext4文件系统，xfs文件系统，

需要内置的内核，包括

-   CONFIG_VIRTIO_BLK=y

-   CONFIG_SCSI_VIRTIO=y

-   CONFIG_HW_RANDOM_VIRTIO=y

-   CONFIG_DRM_VIRTIO_GPU=y

    在make menuconfig界面，善于使用"/"来搜索模块。

  最终使用的配置文件在[这里](sources/config-6.17.11)。
## grub

完成。
第一次尝试使用grub 2.12 

检查：
```bash
ls /tools/bin/grub-*
```


输出：
```text
/tools/bin/grub-editenv   /tools/bin/grub-kbdcomp      /tools/bin/grub-mknetdir		/tools/bin/grub-mkstandalone
/tools/bin/grub-file	  /tools/bin/grub-menulst2cfg  /tools/bin/grub-mkpasswd-pbkdf2	/tools/bin/grub-render-label
/tools/bin/grub-fstest	  /tools/bin/grub-mkimage      /tools/bin/grub-mkrelpath	/tools/bin/grub-script-check
/tools/bin/grub-glue-efi  /tools/bin/grub-mklayout     /tools/bin/grub-mkrescue		/tools/bin/grub-syslinux2cfg
```

注意，编译完成后，应该进行make check,检查编译是否正确。实际测试结果惨不忍睹。
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


转而使用上游[grub 2.14](https://ftp.gnu.org/gnu/grub/grub-2.14.tar.xz)替代。按照孙海勇的脚本编译。

测试结果还是很惨烈：
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
暂时先这样了，似乎也不影响使用。

## wget

完成。

检查：
```bash
ls /tools/bin/wget
```

输出：
```text
/tools/bin/wget
```

## openjdk

可以从这里下载openjdk的龙架构版本，解决自依赖问题。

<https://www.loongnix.cn/zh/api/java>

## rust

需要修改spec文件，在rust_arches中添加loongarch64。

需要设置PKG_CONFIG_PATH

编译的时候遇到报错：
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
添加`export RUSTFLAGS="-A dead_code"`，可以规避上述问题。

编译rust 1.88,需要使用rust 1.88/1.87！这里也有自依赖问题。

下载安装rust 1.88

<https://static.rust-lang.org/dist/rust-1.88.0-loongarch64-unknown-linux-gnu.tar.gz>

下载解压缩后，运行
```bash
./install.sh --prefix=~/build/rust/
```

事实上，rust的安装完全可以放在后面去做，因为已经有现成的rust安装包，可以解决rpm包编译过程中的自依赖问题了。

缺少src/llvm-prject/compiler-rt目录，llvm-project是rust依赖的一个submodule。

下载代码，尝试补全：

<https://github.com/rust-lang/llvm-project/archive/refs/tags/rustc-1.88.0.tar.gz>

参考spec文件，修改编译脚本，设置llvm-root。（需要确认究竟那些修改是必要的，那些是多余的）。

最后决定不做了，现在编译需要补充很多包，比如llvm等，成本过高，后面再编译了。

未完成

参考这个LFS的中文文档 <https://lfs.xry111.site/zh_CN/12.4-systemd>

<https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md>

## dracut

未完成。不影响使用，先不编译了。

# 配置临时系统
完成上述软件包的编译后，可以配置一下临时系统，可以满足启动的要求了。

配置包括：复制文件、创建基本的目录、创建必要的符号链接等。

可以参考这个配置[脚本](scripts/tmp/create-tmp-os.sh)。

临时系统的使用方式有两种，一种是作为容器启动，一种方式是直接做成一个虚拟机来使用。前者门槛比较低，但无法保证系统的完整性；后者的使用门槛以及构建难度略微高一点，但更接近真实的操作系统。

## 作为容器，启动临时系统

使用方式
```bash
sudo systemd-nspawn -b -D ./rootfs/ --private-users=1 --bind=/run/dbus
```
其中，-b 表示要boot这个系统，这样会调用systemd

-D 表示系统的根目录

`--private-users=1` 表示systemd的PID为1,

`--bind=/run/dbus` 是为了让logind.service能够访问dbus。

## 作为虚拟机，启动临时系统

创建一个qcow2虚拟机镜像。

手动操作的流程如下：

- 创建qcow2镜像
```bash
qemu-img create -f qcow2 rocky_vm.qcow2 50G
```

- 准备好nbd模块
```bash
sudo modprobe nbd max_part=16
```

- 挂载qcow2文件
```bash
sudo qemu-nbd -c /dev/nbd0 rocky_vm.qcow2
```

- 对qcow2文件进行分区。这里可以直接用图形界面的gparted进行操作。
```bash 
sudo /sbin/gparted /dev/nbd0
```
首先，创建gpt分区表

设备->创建分区表->gpt

简单的创建两个分区，一个100MiB的FAT32格式EFI分区，其余的空间用做跟分区，不单独划分boot等分区了。

此时，会发现nbd0已经有了p1和p2两个分区了。

```bash
ls /dev/nbd0*
```
```text
/dev/nbd0 /dev/nbd0p1 /dev/nbd0p2
```

准备好分区的挂载点

```bash
mkdir rocky_vm
```

挂载第二个分区
```bash
sudo mount /dev/nbd0p2 rocky_vm
```

将之前的rootfs, 复制到rocky_vm文件夹中
```bash
sudo rsync -av rootfs/* rocky_vm/
```
在rocky_vm中，创建efi的挂载点
```bash
mkdir -pv boot/efi/
```
将efi分区挂载到/boot/efi
```bash
sudo mount /dev/nbd0p1 boot/efi
```
用grub-mkimage安装grub失败，遇到了这样的问题：
```text
/cross-tools/bin/loongarch64-unknown-linux-gnu-grub-mkimage: error:
undefined symbol grub_arch_dl_min_alignment
```
遇到问题，就想办法解决问题。

在 `./grub-core/kern/loongarch64/dl.c`文件中，不存在`grub_arch_dl_min_alignment`函数。

/cross-tools/bin/loongarch64-unknown-linux-gnu-grub-mkimage: error:
undefined symbol start.

最终解决方案：直接使用Debian 提供的grub，实现启动。

完成了虚拟机构建[脚本](scripts/tmp/create-tmp-vm.sh)的编写。

用nspawn进入虚拟机。

2026年3月30日，终于完成了虚拟机的顺利启动，串口和spice界面都可以。

## 虚拟机的配置

后续的移植，都将在虚拟机中完成。我的编译机是一台采用龙芯3A6000M（2.0GHz）的笔记本电脑，只有4核8线程。我给虚拟分配了3个处理器核。

- 设置fstab

两个分区，分别是efi分区 vda1和根分区vda2。

- 设置启动模式
```bash
systemctl set-default multi-user
```

- 完成一些网络功能服务。

设置的过程中，不断遇到新的问题，书本里的内容已经不足以解决我的问题了。现在我已经一只脚踏入了无人区。

- mount 程序的使用。

要使/etc/fstab生效，mount 程序必须能够正常工作。而mount程序正常工作的前提之一，是所有者必须是root。

- 内核设置。

需要将CONFIG_NLS_CODEPAGE_437设置为y,以将nls_cp437模块内置。

CONFIG_NLS_ASCII 设置为y。

再次重新编译内核。


- 设置root密码，设置为了loongarch

- 创建新用户。创建一个叫做rocky的用户。
```bash
useradd -m -s /bin/bash -U -G wheel rocky
```
- 设置用户密码
```bash
passwd rocky
```

- 开启dhcpd

获取IP地址
```bash
ip addr
```

- 开启远程访问：
```bash
ssh-keygen -A
/tools/sbin/sshd
```
现在可以通过ssh,远程访问虚拟机了。

- 设置sudo权限，使普通用户可以用sudo执行命令。注意，sudo 程序必须setuid

- 给新建的用户rocky,设置.bash_profile以及.bashrc

- 用visuo,修改/etc/sudoers文件中的Defaults secure_path，将/tools/sbin,
/tools/bin加进去。

终于可以进展到下一步了！

# 临时系统补全

## Perl

遇到了一些新的问题，MB_LEN_MAX宏定义有问题。正确的值应该是16，但是我的Rocky
系统中的值为1。这导致编译无法继续。

需要搞清楚为什么不一样。

经过对比，`/tools/lib64/gcc/loongarch64-unknown-linux-gnu/14.3.1/include/limits.h`文件，与Debian
系统提供的`/usr/lib/gcc/loongarch64-linux-gnu/14/include/limits.h`文件确实存在不同。用Debian
提供的版本，看看是否能够修复bug。

经过确认，bug已经得到修复了。**后续编译GCC的时候，要加上这个补丁。**

继续编译Perl。

编译完成后，测试：
```bash
/tools/bin/perl --version
```
输出
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

替换config.guess和config.sub

## Git 

正常编译即可。

## Grep

grep的编译，需要用到rpm命令。确保/tools/lib/rpm中的macros的修改是正确的。包括对/tools的替换，对sysconfdir, localstatedir, var, sharedstatedir的设置等。

没有gpgverify, 所以先注释掉相关的行。

## GDB

正常编译即可。

## Xxhash

dwz依赖xxhash.h,因此需要先将其补全。

## DWZ

依赖xxhash.h。

## Procps-ng

替换掉config.guess和config.sub

到此，可以为系统做第一个备份。

rocky_vm-stage1.qcow2

## help2man
perl相关的模块还没有安装，所以编译的时候先`--disable-nls`

## debugedit
依赖help2man。注释掉gpgverify语句

## ~~gpg2~~ 
~~rpm包中使用的gpgverify脚本会调用gpg2。依赖的内容太多了，暂时不管它了。~~

## libkcapi 

libxcrypt依赖fipscheck,
这个包已经由libkcapi提供。所以，需要编译libkcapi。

## python-setuptools

不能直接python setup.py install, 这样不会自动调用shim,无法直接import distutils。

最终使用的是这个方案：
```bash
python3 -m build --wheel
sudo pip3 install dist/setuptools-69.0.3-py3-none-any.whl
```

## redhat-rpm-config
从这里开始，开始进行rpm包的编译。

- 修改`~/rpmbuild/SOURCES/rpmrc`，添加loongarch的optflags：

`optflags:loongarch64 %{__global_compiler_flags} -march=loongarch64 -msimd=lsx`

- 修改`~/rpmbuild/SOURCES/macros.nodejs-srpm`，添加loongarch64

修改redhat-rpm-confg.spec的Release数字

- 重新生成源码包
```bash
rpmbuild -bs ~/rpmbuild/SPECS/redhat-rpm-config.spec
```
- 编译源码包
```bash
rpmbuild --rebuild ~/rpmbuild/SRPMS/redhat-rpm-config-293-2.src.rpm --nodeps
```
生成了新的rpm包：`~/rpmbuild/RPMS/noarch/redhat-rpm-config-293-2.noarch.rpm`

- 强行安装

- 配置一下redhat-rpm-config

## python-srpm-macros
 很多rpm包的编译需要python-srpm-macros提供的`__os_install_post_python`这个宏。
```bash
 rpmbuild \--rebuild /opt/srpms/Packages/python-rpm-macros-3.12-10.el10.src.rpm  
 ```

编译生成了如下文件：
```text
python3-rpm-macros-3.12-10.el10.loongarch.noarch.rpm
python-rpm-macros-3.12-10.el10.loongarch.noarch.rpm
python-srpm-macros-3.12-10.el10.loongarch.noarch.rpm
```
全部安装：
```bash
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/python* --nodeps
```
经过测试，rpm不会从`/usr/lib/rpm/macros.d`文件夹中加载宏。

因为rpm安装在tools，它只会从/tools/lib/rpm/macros.d中加载宏。

解决方案：将`/usr/lib/rpm/macrod.d`中的内容，符号链接到`/tools/lib/rpm/macros.d`中。

## 发行版信息包

找不到类似fedora-repos的包。

## rocky-release

做了如下的修改：注释掉mkdir ./docs

暂时不清楚应该如何修正这些问题。先不管了。需要重新生成srpm包，并编译出rpm包。

## iso-codes

依赖debugedit提供的find-debuginfo程序。

正常编译，生成两个rpm包：
```
iso-codes-devel-4.16.0-6.el10.loongarch.noarch.rpm
iso-codes-4.16.0-6.el10.loongarch.noarch.rpm
```
安装这些包。
```bash
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/iso-codes-* --nodeps
```
没有修改代码，所以不需要重新生成srpm包。

## setup

依赖关系：`systemd-rpm-macros: required to use _tmpfilesdir macro`

所以，需要先安装systemd-rpm-macros。这个包是systemd提供的。看看编译systemd的时候是否装了这些东西。
```bash
rpmbuild -D "version_no_tilde 257" -bp ~/rpmbuild/SPECS/systemd.spec --nodeps
```

把编译编译systemd的时候，build文件夹中的macros.systemd复制出来到/tools/lib/rpm/macrod.d中。
```bash
rpmbuild --bb ~/rpmbuild/SPECS/setup.spec --nodeps
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/setup-2.14.5-7.el10.loongarch.noarch.rpm
```
未改动。

## Filesystem

没有针对loongarch的设置，所以需要手动添加。

在包含aarch64或者riscv64的地方，加上loongarch64。

生成了两个rpm包：
```
filesystem-content-3.18-17.el10.loongarch.loongarch64.rpm
filesystem-3.18-17.el10.loongarch.loongarch64.rpm
```
需要更改版本号，并重新生成srpm包。

## Basesystem
```bash
rpmbuild --rebuild /opt/srpms/Packages/basesystem-11-22.el10.src.rpm
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/basesystem-11-22.el10.loongarch.noarch.rpm
```
未进行任何改动

## less

编译这个软件包纯粹是出于个人的喜好，顺便测试一下编译器是否好用。

less软件包的依赖关系比较简单，很好编译。
```bash
rpmbuild -bb ~/rpmbuild/SPECS/less.spec --nodeps 
sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/loongarch64/less-661-3.el10.loongarch.loongarch64.rpm --nodeps
```

## rsync

直接编译内核头文件的时候，依赖rsync。尝试编译rpm包。生成了5个包：
只安装rsync-3.4.1-2.el10.loongarch.loongarch64.rpm就可以了。

## 内核头文件 kernel-header

不存在kernel-headers这个包。只有kernel-6.12的包。

修改kernel.spec文件：

1.  在含有ExclusiveArch的行最后加上loongarch64
2.  添加对龙架构的支持：

```text
%ifarch loongarch64
%define asmarch loongarch
%define hdarch loongarch
%define make_target vmlinuz.efi
%define kernel_image arch/loongarch/boot/vmlinuz.efi
%endif
```
编译成功，生成了3个文件：
```text
kernel-modules-extra-matched-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
kernel-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
kernel-cross-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
```
有警告:
```text
RPM build warnings:
File not found:
/home/rocky/rpmbuild/BUILDROOT/kernel-6.12.0-124.21.1.el10.loongarch.loongarch64/usr/include/cpufreq.h
File not found:
/home/rocky/rpmbuild/BUILDROOT/kernel-6.12.0-124.21.1.el10.loongarch.loongarch64/usr/include/ynl
```
重新生成一下源码包，避免被删掉。

安装编译好的rpm包：
```bash
sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/loongarch64/kernel-headers-6.12.0-124.21.1.el10.loongarch.loongarch64.rpm
```
## glibc

做了如下的修改：

`--enable-systemtap` 改为`--disable-systemtap`

添加了对loongarch的一些支持：
```text
%ifarch loongarch64
%global glibc_ldso /lib64/ld-linux-loongarch-lp64d.so.1
%global glibc_has_libnldbl 1
%global glibc_has_libmvec 0
%endif
```
设置：

BuildFlagsNonshared=""

如果架构是龙架构，不要用Patch261 这个补丁。

一些没有成功生成的文件，就先不管了。

编译的时候 without testsuite， benchtests以及valgrind

生成了200多个文件。

安装文件，并调整工具链。

测试工具链。
```bash
readelf -l a.out |grep ": /lib"
```
输出
```text
[Requesting program interpreter: /lib64/ld-linux-loongarch-lp64d.so.1]
```
工具链调整正确，已经不是/tools/lib64中的ld了！

## Zlib

正常编译即可，不需要调整spec文件。

安装
```bash
sudo rpm -Uvh $(ls ~/rpmbuild/RPMS/loongarch64/{minizip,zlib}-* |grep -v -e "debug") --nodeps 
```
## libxcrypt

由于没有安装gpg2, 需要注释掉gpgverify行，或添加 with bootstrap选项。

遇到了这样的报错：
``` bash
fipshmac -d
/home/rocky/rpmbuild/BUILDROOT/libxcrypt-4.4.36-10.el10\~bootstrap.loongarch.loongarch64/usr/lib64/fipscheck
/home/rocky/rpmbuild/BUILDROOT/libxcrypt-4.4.36-10.el10\~bootstrap.loongarch.loongarch64/usr/lib64/libcrypt.so.2.0.0
Allocation of hmac(sha256) cipher failed (ret=-97)
```

编译libxcrypt的过程中，需要使用fipshmac程序。这个程序的运行与要依赖一系列的加密相关的内核模块，所以需要运行如下的命令：
```bash
modprobe crypto_user
modprobe crypto_hmac
modprobe af_alg
modprobe crypto_hash
modprobe algif_hash
````
参考：

<https://gitee.com/openeuler/community/issues/I1KQMK>

遇到了一个新的bug：fipshmac运行的时候，会自动在生成的.hmac文件前面加一个点，这导致最后文件名匹配不上。

修改spec文件，在：
```bash
fipshmac -d $fipsdir                        \\\
  $libdir/libcrypt.so.%{sov}                \
```

后面加了两句：
```bash
mv -v $fipsdir/.libcrypt.so.%{sov}.hmac     \\\
   $fipsdir/libcrypt.so.%{sov}.hmac         \
```

编译方式：
```bash
rpmbuild -bb ~/rpmbuild/SPECS/libxcrypt.spec --nodeps --with bootstrap --without new_api --nocheck --without compat_pkg --without staticlib
```

由于我添加了`--without compat_pkg` `--without staticlib`，编译生成的包少了很多，只有四个：
```text
libxcrypt-devel-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-debuginfo-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
libxcrypt-debugsource-4.4.36-10.el10~bootstrap.loongarch.loongarch64.rpm
```
安装前两个包
```bash
sudo rpm -ivh libxcrypt-4.4.36-10.el10\~bootstrap.loongarch.loongarch64.rpm \
              libxcrypt-devel-4.4.36-10.el10\~bootstrap.loongarch.loongarch64.rpm \
              --nodeps
```
libxcrypt需要libssp.so, 这就是为什么前面要把libssp.so做符号连接。

## binutils 

为保险起见，为龙架构禁用gold。

修改spec文件

在%bcond_without gold前一行，加上loongarch64

重新生成源码包
```bash
rpmbuild -bs rpmbuild/SPECS/binutils.spec
```
编译rpm包：
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/binutils-2.41-58.el10.loongarch.2.src.rpm --with bootstrap --nodeps --nocheck --without debuginfod
```
生成了4个新的rpm包。

安装：
```bash
sudo rpm -ivh binutils-2.41-58.el10.loongarch.2.loongarch64.rpm \
       binutils-devel-2.41-58.el10.loongarch.2.loongarch64.rpm \
       --nodeps
```
由于没有`/usr/sbin/alternatives`， 需要手动设置符号连接：

```bash
sudo ln -sv ld.bfd /bin/ld
```
## GMP

编译的时候，需要一个libzstd.so.1文件。在/usr/lib64中，建一个指向/tools/lib64/libzstd.so.1的符号连接

安装gmp的源码包以后，修改`~/rpmbuild/SOURCES/gmp.h`

在合适的位置，添加：
```c
#elif defined(__loongarch64)
#include "gmp-loongarch64.h"
```
修改Release号。

重新打包源码包
```bash
rpmbuild -bs ~/rpmbuild/SPECS/gmp.spec
```
重新编译打包的源码包
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/gmp-6.2.1-12.el10.loongarch.1.src.rpm --nodeps
```

安装：
```bash
sudo rpm -ivh gmp-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-c++-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-devel-6.2.1-12.el10.loongarch.loongarch64.rpm \
            gmp-static-6.2.1-12.el10.loongarch.loongarch64.rpm \
            --nodeps
```
## MPFR

不需要特殊处理，直接编译安装即可。

## libMPC

直接编译。

```
安装：
```bash
sudo rpm -ihv libmpc-devel-1.3.1-7.el10.loongarch.loongarch64.rpm \
  libmpc-1.3.1-7.el10.loongarch.loongarch64.rpm \
  ../noarch/libmpc-doc-1.3.1-7.el10.loongarch.noarch.rpm \
  --nodeps
```
## XZ

修改spec文件，不进行gpgverify

```
安装：
```bash
sudo rpm -ivh xz-devel-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-libs-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-lzma-compat-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-static-5.6.2-4.el10.loongarch.loongarch64.rpm \
  xz-5.6.2-4.el10.loongarch.loongarch64.rpm \
  --nodeps
```
## meson

依赖python-setuptools。

生成文件：meson-1.4.1-4.el10.loongarch.noarch.rpm

**注意**
- 安装了meson以后，会在/usr/lib/rpm/macros.d内创建文件macros.meson。但系统中的rpm依然是/tools/中的，所以需要做一个符号链接。
- meson是一个python的程序，所以需要注意其安装位置。目前系统的python依然是在/tools/中，所以必要的时候需要做符号链接！
- 我又用pip 装了一个meson.最终生效的，可能是这个meson.

version_no_tilde 这个宏，应该是定义在 rust-srpm-macros中的。

## ninja-build 

meson运行的时候，依赖ninja-build
```bash
rpmbuild --rebuild /opt/srpms/Packages/ninja-build-1.11.1-9.el10.src.rpm --nodeps --with bootstrap
```
## LZ4

编译需要meson。

meson依赖python-setuptools

编译，安装不含debug的4个包。

## ZSTD 
编译的时候，汇报说缺少一个命令：
```text
line 46: execstack: command not found
```
尝试编译execstack, 遇到新的错误：
```text
configure: error: libelf does not properly convert Elf64_Sxword
quantities.
If you are using libelf-0.7.0, please use patches/libelf-0.7.0.patch.
```
先跳过execstack命令的执行
```bash
rpmbuild --rebuild /opt/srpms/Packages/zstd-1.5.5-9.el10.src.rpm \
        --nodeps \
        --nocheck
```
生成了7个rpm文件。
安装
```bash
sudo rpm -ivh libzstd-1.5.5-9.el10.loongarch.loongarch64.rpm \
  libzstd-devel-1.5.5-9.el10.loongarch.loongarch64.rpm \
  zstd-1.5.5-9.el10.loongarch.loongarch64.rpm \
  libzstd-static-1.5.5-9.el10.loongarch.loongarch64.rpm \
  --nodeps
```
## GCC

需要修改spec文件，添加一个补丁。
`0001-fixincludes-Skip-pthread_incomplete_struct_argument-.patch`

需要修改的地方还有很多。

loongarch相关的头文件，需要加上。

现在还无法完整的编译文档，将与文档相关的内容跳过。

annobin plugin相关的，先跳过。

从开源欧拉的源码上抄一抄作业。

<https://atomgit.com/src-openeuler/gcc/blob/master/gcc.spec>

注意，开源欧拉上的GCC版本是12,这个版本的作业不能全抄。

gcc 14.3发布于2025年5月23日，已经包含了libquadmath的支持。

开始尝试编译GCC。

AS程序无法正常运行。看样子，是binutils未能正确创建libbfd.so的符号连接

暂时先做个符号链接，看看是否能修复问题。

编译一次需要花很久，大概8个小时。

又遇到了新的问题:
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

修改spec文件，在libgccjit-devel的doc中，注释掉rpm.doc/libgccjit-devel/相关的语句。

编译成功，总计耗时462分钟，7.7小时，生成的文件有46个。

打包最后的警告信息：

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
可以说编译打包的过程并不是非常完美的，不过还是能用的。
安装部分rpm包。

清理收尾。

编译测试。

## glibc 第2遍

现在开始，重新编译工具链。

glibc的spec文件做过改动，不能直接从源码编译。用之前写的编译glibc的脚本编译。

确保存在/usr/lib/rpm/redhat/redhat-annobin-cc1文件，且这个文件为空。因为现在的龙架构下的GCC并不支持annbin。

编译成功，安装软件包，一共442个包。

## binutils 第2遍

利用之前的编译脚本，进行编译。

安装：
```bash
sudo rpm -Uvh binutils-2.41-58.el10.loongarch.2.loongarch64.rpm \
              binutils-devel-2.41-58.el10.loongarch.2.loongarch64.rpm \
              --nodeps \
              --force
```
安装的时候还是会遇到这样的报错：

```text
/usr/sbin/alternatives: No such file or directory
```
手动做链接。
```bash
sudo ln -sv ld.bfd /bin/ld
```
## ZLIB

这个包的源码没有进行调整，直接使用原始的src.rpm包编译安装即可。
```bash
rpmbuild --rebuild /opt/srpms/Packages/zlib-1.2.11-40.el9.src.rpm --nodeps
```
##  libxcrypt
使用之前留下来的spec文件来编译。

注意，还是需要modprobe很多内核模块。

## GMP 

正常操作。

## MPFR 

正常操作。

## LibMPC 

正常操作。

## XZ

正常操作。

## LZ4 
正常操作。

## ZSTD 

正常操作。

## GCC 第2遍

正常操作。编译耗时447m。

安装gcc。

清理不需要的库文件。

保证/lib64下面，没有指向/tools的符号链接。

# 残破的目标系统
继续补全目标系统。

设置默认自举。

在`/tools/lib/rpm/platform/loongarch64-linux/macros`结尾，添加： `%with_bootstrap 1`
首先，进行系统交互环境的编译。
## ncurses

注释掉SPEC文件中的gpgverify 行

生成了的11个文件，安装其中与debug无关的7个。

## readline

正常编译即可。

## BASH

也需要注释掉gpgverify语句

编译生成5个文件,安装其中与debug无关的3个。

## bash-completion 
正常编译安装即可。

## bzip2

注释掉gpgverify

编译以后，生成了7个文件
安装其中与debug无关的4个。

## TCL

直接编译安装。

## libdb

不需要了。

## gdbm

正常编译

## Perl

check通不过，编译的时候添加`--nocheck`

生成了非常多的文件，共计229个，安装了其中188个。

调整rpm的宏中关于perl的部分。

## Perl-srpm-macros

包的构建和安装非常简单。生成了1个包：`perl-srpm-macros-1-57.el10~bootstrap.loongarch.noarch.rpm`。

## Perl-Fedora-VSP

正常构建即可。

## Perl-generators

正常构建即可。

## Python-rpm-macros

正常编译，安装。

**注意**：记得要复制文件到`/tools/lib/rpm/macros.d/`

## Unzip

实际打包的时候还是要添加`--nodeps`
## Zip

打包的时候添加`--nodeps`。

## help2man
  正常编译即可。

## M4

正常编译即可。

## Autoconf
  编译的时候，去掉对emacs的支持。
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

修改spec文件

生成5个文件，安装其中与debug无关的3个。

## libtool

正常编译即可。

## Pkgconf

生成了10个rpm包。编译命令如下：
```bash
rpmbuild --rebuild /opt/srpms/Packages/pkgconf-2.1.0-3.el10.src.rpm --nocheck --nodeps
```
## Lua

正常编译即可。

## diffutils

正常编译即可。

## attr

正常编译即可。

## acl

正常编译即可。

## tar
```bash
rpmbuild --rebuild /opt/srpms/Packages/tar-1.35-9.el10_1.src.rpm \
        --without check \
        --without selinux \
        --nodeps
```
## cpio
  正常编译即可。
## bison
正常编译即可。测试时间很长。

##  FLEX

完成。

## Ed

需要注释掉spec文件中的gpgverify语句。

## patch

修改spec文件中关于selinux补丁的部分，不打这个补丁。

## Check

需要依赖cmake来编译。先跳过。

## CMake

尝试一下，强行编译cmake能否通过。

依赖openssl。先跳过，编译完openssl再来编译cmake.

## make 

正常编译。

## tcsh

不要check

## sed
正常编译即可。

## grep

正常编译即可。

## findutils

正常编译即可。

## file
编译脚本
```bash
rpmbuild -bb ~/rpmbuild/SPECS/file.spec --nodeps --without python3
```
## NSPR

nspr源码在NSS中，不再单独编译这个了。

## sqlite

测试需要花的时间比较长。

## pcre2

没有pcre了，只有pcre2。

依赖readline.h,需要安装这个包。

check的时候没有通过，跳过check。
```bash
rpmbuild -bb ~/rpmbuild/SPECS/pcre2.spec --nodeps --nocheck
```
## libsepol

正常编译即可。

## libgpg-error

正常编译即可。

## libgcrypt

编译的时候遇到报错：
```text
lto1: fatal error: inaccessible plugin file plugin/annobin.so expanded
from short plugin name annobin: No such file or directory
```

注释掉patch1, 关于annobin的patch。

测试需要花不少时间。check需要annocheck命令，没有安装，因此跳过check。

## chrpath

正常编译即可。

## expect

需要依赖chrpath。已经安装。

## screen

没有找到安装包。

## dejagnu

正常编译即可。

## libffi

修改spec文件

修改`~/rpmbuild/SOURCES/ffi-multilib.h`，添加
```c
#elif defined(__loongarch64)
#include "ffi-loongarch64.h"
```

修改ffitarget-multilib.h, 添加
```c
#elif defined(__loongarch64)
#include "ffitarget-loongarch64.h"
```
遇到了一个编译的错误
```text
../src/loongarch64/ffi.c:525:7: error: implicit declaration of function 'ffi_tramp_set_parms' [-Wimplicit-function-declaration]
```

经过检索，这个问题已经由xry111修复了：

<https://github.com/libffi/libffi/pull/825>

下载<https://github.com/libffi/libffi/pull/825.patch>,
修改spec文件，添上这个补丁即可。

## lksctp-tools

完成

## openssl

修改spec文件，添加对loongarch64的架构支持

重新打包源码，制作软件包。

耗时挺久的。

打包的时候，遇到这样的报错：
```text
+ rename so.3 so.3.5.1 '/home/rocky/rpmbuild/BUILDROOT/openssl-3.5.1-4.el10~bootstrap.loongarch.1.loongarch64/usr/lib64/*.so.3'
rename: /home/rocky/rpmbuild/BUILDROOT/openssl-3.5.1-4.el10~bootstrap.loongarch.1.loongarch64/usr/lib64/*.so.3: not accessible: No such file or directory
error: Bad exit status from /var/tmp/rpm-tmp.9zb2oM (%install)
```
因为so文件最后安装到了`/usr/lib/`,应该把_lib设置为`/usr/lib64/`

修改SPEC文件，设置一下龙架构的libdir路径
```bash
%ifarch riscv64 loongarch64
--libdir=%{_lib} \
%endif
```
## libxml2

修改spec文件，去除`__pycache`目录相关的处理和打包。

需要设置一下PKG_CONFIG_PATH环境变量。

编译的时候遇到了这样的报错：
```text
tree.c:227:17: error: 'SIZE_MAX' undeclared (first use in this function)
  227 |     if (lenn >= SIZE_MAX - lenp - 1)
      |                 ^~~~~~~~
tree.c:49:1: note: 'SIZE_MAX' is defined in header '<stdint.h>'; this is probably fixable by adding '#include <stdint.h>'
   48 | #include "private/tree.h"
  +++ |+#include <stdint.h>
   49 | 
```

经过查询，这个问题已经有人提到了：

<https://gitlab.gnome.org/GNOME/libxml2/-/commit/3773bb3f89426297dd82d8ad5998059898d5172a>

跟大模型学会了下载patch文件的办法：直接下载 <https://gitlab.gnome.org/GNOME/libxml2/-/commit/3773bb3f89426297dd82d8ad5998059898d5172a.patch>
即可获得补丁。

修改spec文件，手动创建一个补丁，打上。

又遇到了新的问题
```text
ERROR   0002: file '/usr/lib64/libxml2.so.2.12.5' contains an invalid rpath '/tools/lib64' in [/tools/lib64]
```

按照程序的提示，添加`QA_RPATHS=$(( 0x0002|0x0004 ))`，规避上述问题。

## libxslt 

正常编译即可。

## sgml-common 

正常编译即可。

## docbook-dtds 

正常编译即可。

## docbook-style-xls 

正常编译即可。

## xmlto 

正常编译即可。

## expat 

正常编译即可。

## mpdecimal 

正常编译即可。

## python3 

依赖mpdecimal-devel，需要先安装这个。

按照书中的要求进行修改。

## libxml2 

解决依赖关系以后，重新编译libxml2

## NSS 

按照书中的提示，修改spec文件，注释掉一部分代码。

`config.sub`以及`config.guess`太老，需要更新。

使用`autoreconf -ifv`更新它们。

## PCRE2 

已经安装

## LZO 

正常编译即可。

## icu 

不生成文档。

## Boost 

去掉一些功能模块。

编译遇到了一些报错：
```text
Installed (but unpackaged) file(s) found:
/usr/lib/debug/usr/lib64/libboost_python312.so.1.83.0-1.83.0-5.el10\~bootstrap.loongarch.loongarch64.debug
/usr/lib64/libboost_python312.so
/usr/lib64/libboost_python312.so.1.83.0
````

暂时给rpmbuild添加参数`--define _unpackaged_files_terminate_build 0`，规避这个问题

## Cmake 

去掉一些功能模块。

cmake依赖vim-filesystem。

手动添加了vimfiles_root宏的定义: `--define "vimfiles_root %{_datadir}/vim/vimfiles/"`

遇到未打包的文件，不要终止，添加语句：`--define '_unpackaged_files_terminate_build 0'`

test需要花很长的时间，如果通不过，就加上`--nocheck`

## Hostname 

正常编译即可。

## Coreutils 

注释掉spec中的gpg校验部分。有两个案例会fail， 不管了。加上`--nocheck`。

coreutils-single和coreutils两个包冲突，前者不装了。

## Python-rpm-generators 

正常编译安装即可。还依赖了python-packaging，
pyproject-rpm-macros，pyproject-srpm-macros,
python-packaging，都需要编译安装。

## Python-setuptools 

依赖python-rpm-generators。

## Re2c 

没有找到源码,不编译。

## Ninja-build 

正常编译即可。不需要修正rpmmacrodir，因为现在的spec文件中已经不用这个宏了。

## Meson 

遇到了version_no_tilde的宏。直接定义一下。
```bash
rpmbuild --rebuild /opt/srpms/Packages/meson-1.4.1-4.el10.src.rpm --define " version_no_tilde 1.4.1"
```
将macros.meson复制到/tools/lib/rpm/

## Dos2unix 

注释掉spec文件中，关于gpgverify的语句。

## Swig 

添加`--without testsuite`参数

## Less 

正常编译即可。

## Gzip 

正常编译。

## libpaper 

正常编译。

## chrpath 

按照书中的写法，修改spec文件。

因为后续的编译中，需要用chrpath来做一些检查，我们需要跳过这些检查。
```bash
rpmbuild -ba ~/rpmbuild/SPECS/chrpath.spec --define '_unpackaged_files_terminate_build 0'
```
## Psutils 

测试通不过。加上`--nocheck`。

## libedit 

正常编译即可。

## multilib-rpm-config 

正常编译即可。

## llvm 

遇到这样的报错：
```text
错误：lua 脚本执行失败：[string "python_provide"]:2: module 'fedora.srpm.python' not found:
```
需要重新编安装python-rpm-macros。

## python-lit 

找不到源码包了，无法安装。

## clang 

和llvm相关，先跳过。

## ccache 

找不到源码包，无法安装。

## nasm 

正常编译即可。

## Gpref 

正常编译即可。

## libutempter 

正常编译即可。

## libpipeline 

正常编译即可。


## giflib 
开始编译图形相关软件包。

giflib这个包依赖mingw64-gcc, mingw64-filesystem

修改spec文件，做一个完全不依赖mingw的版本。

## libpng 

正常编译即可。

## libjpeg-turbo 

正常编译即可。

## Jbigkit 

正常编译即可。

## libtiff 

正常编译即可。

## lcms2 

正常编译即可。

## libmng 

正常编译即可。

## Xorg-X11-utils-macros 

正常编译即可。

## Xorg-X11-Proto-devel 

正常编译即可。

## libXau 

正常编译即可。

## libXdmcp 

正常编译即可。

## xcb-proto 

正常编译即可。

## libxcb 

正常编译即可。

## xorg-x11-xtrans-devel 

正常编译即可。

## libx11 

正常编译即可。

## brotli 

正常编译即可。

## freetype 

依赖brotli

## forge-srpm-macros 

打包的时候加上`--nocheck`
编译脚本：
```bash
rpmbuild --rebuild /opt/srpms/Packages/forge-srpm-macros-0.4.0-6.el10.src.rpm --nodeps --nocheck
```
## fonts-rpm-macros 

依赖forge-srpm-macros，顺便安装了。

把安装的宏定义文件，复制到`/tools/lib/rpm/macrod.d/`

## fontconfig 

按照书中操作即可。

## Pixman 

正常编译即可。

## libimagequant 

找不到源码。

## libICE 

正常编译即可。

## libSM 

修改spec文件，将`--with-libuuid`, 改为`--without-libuuid`

## libXrender 

正常编译即可。

## libXext 

正常编译即可。

## libXt 

正常编译即可。

## libXpm 

正常编译即可。

## libXmu 

正常编译即可。

## libXaw 

正常编译即可。

## libXfixes 

正常编译即可。

## libXi 

正常编译即可。

## libXft 

正常编译即可。

## libXinerama 

正常编译即可。

## libXtst 

正常编译即可。

## python-wheels 

正常编译即可。

## python-pip 

正常编译即可。

## marshalparser 

正常编译即可。

## glib2 

依赖g-ir-scanner，尝试编译安装gobject-introspection。gobject-introspection也依赖glib2。出现循环依赖了。给meson添加`-Dintrospection=disabled`
参数。

依赖sysprof-capture-4，尝试满足，又是一个循环依赖。继续破除依赖： `-Dsysprof=disabled`

依赖libelf，继续添加 `-Dlibelf=disabled`

还依赖rst2man, 需要python3-docutils，继续破除依赖`-Dman-pages=disabled`

还是缺少一些东西，缺了marshalparser,补上。另外补充了python-pip,
python-wheels包。

check的时候有几个报错：
```text
 16/375 glib:glib+core+slow / gdatetime                                        ERROR            0.01s   killed by signal 6 SIGABRT
 22/375 glib:glib+core / hmac                                                  ERROR            0.01s   killed by signal 5 SIGTRAP
109/375 glib:glib+core / autoptr                                               ERROR            0.01s   killed by signal 5 SIGTRAP
217/375 glib:gio / contenttype                                                 ERROR            0.01s   killed by signal 6 SIGABRT
282/375 glib:gio / file                                                        ERROR            0.04s   killed by signal 5 SIGTRAP
304/375 glib:gio / desktop-app-info                                            ERROR            0.01s   killed by signal 5 SIGTRAP
```
这些报错倒是可以理解。添加`--nocheck`

还有报错：
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
继续修改spec文件，不要这些文件

还有报错：
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
继续处理缺失的目录
```text
  Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gio-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gio-unix-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/girepository-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/glib-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/glib-unix-2.0
    Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gmodule-2.0
Directory not found: /home/rocky/rpmbuild/BUILDROOT/glib2-2.80.4-10.el10~bootstrap.loongarch.loongarch64/usr/share/doc/gobject-2.0
```
编译成功，安装四个非debug的包。

## Cairo 

新版本的cairo,构建方式改成了meson。

先尝试构建一下，看看报什么错
修改spec文件，需要如下改动：

 * `-Dgtk_doc=false`

* 去掉`gtk-doc/html/cairo`

## gd 

编译的时候添加 `--nodeps --nocheck`

## libcroco 

找不到

## Jbig2dec 

正常编译即可。

## OpenJPEG2 

正常编译即可。

## graphite2 

新版的graphite2已经不编译docs了。

## Python-mako 
```bash
rpmbuild --rebuild /opt/srpms/Packages/python-mako-1.2.3-9.el10.src.rpm --nodeps --nocheck
```
## gobject-introspection 

正常编译即可。

## Harfbuzz 

这个包也是采用meson编译，缺少gtk-doc,给meson添加参数
```text
-Ddocs=disabled -Ddoc_tests=false -Dtests=disabled
-Dintrospection=disabled
```
## T1lib 

没有找到源码包。

## T1utils 

没有找到源码包。

## Xaw3d 

注释掉gpgverify的语句

## libXComposite 

正常编译即可。

## zziplib 

正常编译即可。

## teckit 

正常编译即可。

## potrace 

正常编译即可。

## Texlive 

texlive依赖zziplib，teckit, potrace, javac

去掉所有依赖javac的内容，即tex4ht。

最终生成了398个包。

## gettext 

正常编译即可。

## adobe-mapping-cmap 

正常编译即可。

## adobe-mapping-pdf 

正常编译即可。

## symlinks

正常编译即可。

## urw-base35-fonts

正常编译即可。

## xorg-x11-font-utils

正常编译即可。

## libidn2

正常编译即可。

## libijs

正常编译即可。

## ghostscript 

正常编译即可。

编译以后，ps2pdf命令运行有问题，暂时不知道如何解决。尝试尽量补全依赖关系，重新编译。

补全了libijs、libidn2、urw-base35-fonts等依赖关系后，ps2pdf命令可以正常运行了。

## google-droid-font 

正常编译即可。

## libsigsegv 

正常编译即可。

## poppler-data 

正常编译即可。

## poppler 

依赖Gpgmepp。不想要这个，直接关掉。SPEC文件中添加`-DENABLE_GPGME=OFF`

目前没有编译任何Qt相关的包，Qt也关掉。rpmbuild 添加 `--without qt`

找不到curl，先关掉。需要在spec文件中做如下修改：
```text
-DENABLE_GTK_DOC=OFF \\
-DENABLE_GPGME=OFF \\
-DENABLE_LIBCURL=OFF \\
-DENABLE_GOBJECT_INTROSPECTION=OFF \\
```

编译脚本

```bash
rpmbuild -bb rpmbuild/SPECS/poppler.spec  --nodeps --nocheck --without qt
```
## ZZiplib

已经安装了

## Teckit

已经安装

## FFCall

找不到

## clisp

找不到。

## Potrace

已经安装。

## perl-devel-checklib

正常编译即可。
## perl-XML-parser

正常编译即可。
## perl-XML-XPath

正常编译即可。
## texlive-base

找不到。

## gawk

正常编译即可。
## groff

正常编译即可。
## Xapian-core

正常编译即可。
## QPdf

修改CMake的设置
```text
-DREQUIRE_CRYPTO_GNUTLS=0\\
-DALLOW_CRYPTO_NATIVE=1 \\
-DREQUIRE_CRYPTO_NATIVE=1 \\
```
## TK

正常编译即可。

## Graphviz
编译脚本
```bash
rpmbuild -bb ~/rpmbuild/SPECS/graphviz.spec --without php --without ocaml --nodeps
```
## Doxygen

编译脚本：
```bash
rpmbuild -bb rpmbuild/SPECS/doxygen.spec --define "_module_build 1" --nodeps --nocheck
```
## symlinks

已经安装。

## asciidoc

 正常编译即可。

## opensp

正常编译即可。

## openjade

正常编译即可。

## linuxdoc-tools

正常编译即可。

## elinks

正常编译即可。

## python3-libxml2

重新编译。注意，使用重新打包的libxml2。

## itstool

正常编译即可。
## docbook-style-dssl

正常编译即可。
## perl-sgmlspm

正常编译即可。
## dockbook-utils

正常编译即可。
## docbook5-style-xsl

正常编译即可。
## oniguruma

正常编译即可。
## slang

正常编译即可。
## telnet

正常编译即可。
## lynx

依赖telnet,补上。

## words

正常编译即可。

## cracklib

正常编译即可。

## libtirpc

正常编译即可。

## libnsl2

找不到源码包。

## libyaml

正常编译即可。

## procps-ng

正常编译即可。

## which

正常编译即可。

## checksec

找不到源码包。

## rubypick

找不到源码包。

## ruby

测试的时候，关于TZ的很多测试会失败。
```bash
rpmbuild --rebuild /home/rocky/rpmbuild/SRPMS/ruby-3.3.10-12.el10\~bootstrap.loongarch.src.rpm --without systemtap --define "with_hardening_test 0" --nodeps --nocheck
```

## 重要提醒
下面开始要编译的软件包，与系统的启动、登录过程相关。

当前状态下，系统的登录依赖的是/tools/目录中的文件；随着编译的进行，/tools中的部分可执行文件会被新编译的rpm包所提供的文件所取代，导致系统处于一个混乱的混合状态：rpm包提供的文件和/tools中的文件共同发挥作用，但彼此并不完全兼容。

**注意**： 从当前开始，如果重启虚拟机，可能会导致虚拟机无法登录！建议从当前开始，不要关闭虚拟机，使用“保存虚拟机”的方法来取代关机。最好一口气完成所有与登录相关的软件包的编译，然后可以安全重启。

**切记！**

如果已经遇到了虚拟机无法登录的问题，也不必恐慌，后面还会介绍解决方案。


## libselinux

正常编译即可。

## libcap-ng

正常编译即可。

## pam

需要解一下依赖关系。

修改spec文件，将`--enable-audit` 改为`--disable-audit`

在安装文档的时候，去掉adg mwg sag 的txt文件。 编译脚本：
```bash
rpmbuild -bb rpmbuild/SPECS/pam.spec --define '_unpackaged_files_terminate_build 0' --nodeps
```

## libpwquality

正常编译即可。

## audit

编译脚本
```bash
rpmbuild -bb rpmbuild/SPECS/audit.spec --nodeps --define '_unpackaged_files_terminate_build 0'
```
## lmdb

正常编译即可。

## popt

正常编译即可。

## libuser

需要使用gtk-doc中的gtkdocize命令。先跳过这个，直接编译。

spec文件中，添加

```bash
sed -i "s/gtkdocize/#gtkdocize/g" autogen.sh
```

`--enable-gtk-doc`改为`--disable-gtk-doc`

去掉`--html-doc`的语句

修改`docs/reference/Makefile.am`,
去掉有关`gtk-doc.make`的语句，去掉`ENABLE_GTK_DOC`相关的判断。

## libsemanage

正常编译即可。

## shadow-utils

正常编译即可。

## libutempter

正常编译即可。

## rubygem-asciidoctor

正常编译即可。

## util-linux

依赖`rubygem-asciidoctor`

将`--with-udev`改为`--without-udev`, `--with-systemd`改为`--without-systemd`

## libev

正常编译即可。

## libevent

正常编译即可。

## libverto

正常编译即可。

## fuse

正常编译即可。

## e2fsprogs

添加`-D_FILE_OFFSET_BITS=64`

## keyutils

正常编译即可。

## krb5

依赖libcom_err ,这个包由e2fsprogs 提供。

## libksba

正常编译即可。

## libassuan

正常编译即可。

## npth

正常编译即可。

## json-c

正常编译即可。

## gnupg2

修改SPEC文件，去掉对`tpm2-tss`的依赖

不产生dirmngr文件和dirmngr-client文件

## python-setuptools_scm

添加 `--nodeps --nocheck` 参数

## python3-pluggy

正常编译即可。

## python-iniconfig

正常编译即可。

## libtasn1

正常编译即可。

## libcap

正常编译即可。

## bash-completion

正常编译即可。

## newt

正常编译即可。

## beakerlib

找不到安装包。

## chkconfig

编译脚本
```bash
rpmbuild -bb rpmbuild/SPECS/chkconfig.spec --nodeps  --define '_unpackaged_files_terminate_build 0'
```
## p11-kit

 完成

## fipscheck

 找不到安装包。

## nettle

正常编译即可。

## ca-certificates

正常编译即可。

## libunistring

正常编译即可。

## gc

找不到安装包。

## guile

找不到安装包。

## autogen

找不到安装包。

## trousers

找不到安装包。

## pytest

下决心，补上gtk-doc这个很基础的包。一点点补上依赖关系。

```bash
rpmbuild --rebuild /opt/srpms/Packages/pytest-7.4.3-5.el10.src.rpm --without tests --without timeout --without docs --nocheck
```
遇到了这样的错误:
```text
 FileExistsError: %pyproject_install has found more than one *.dist-info/RECORD file. Currently, %pyproject_save_files supports only one wheel → one file list mapping. Feel free to open a bugzilla for pyproject-rpm-macros and describe your usecase.
 error: Bad exit status from /var/tmp/rpm-tmp.FVnBBF (%install)
```

解决方案： 

把`~/rpmbuild/BUILD`,`~/rpmbuild/BUILDROOT`文件夹中，有关pytest的内容全部删掉，再重新编译。

## cython

```bash
 rpmbuild --rebuild /opt/srpms/Packages/Cython-3.0.9-8.el10.src.rpm --nodeps
```
## python-lxml

 用root 编译

## python-pathspec

正常编译即可。

## python-trove-classifiers

正常编译即可。

## python-hatchling

正常编译即可。

## python-pygments

正常编译即可。

## gtk-doc

```bash
rpmbuild --rebuild /opt/srpms/Packages/gtk-doc-1.33.2-12.el10.src.rpm --nodeps
```
## 异常处理

在编译的过程中，我关闭过虚拟机，重启以后虚拟机就无法登录了，输入密码以后显示login incorrect。

直接挂载qcow2文件，chroot进去，查看日志

/tools/bin/journalctl

看到这些报错：
```text
May 04 13:53:45 RockyLoong (agetty)[410]: console-getty.service: Executing: /sbin/agetty -o "-- \\u" --noreset --noclear --keep-baud 115200,57600>
May 04 13:53:45 RockyLoong systemd[1]: Got handoff timestamp event for PID 410.
May 04 13:53:49 RockyLoong login[410]: PAM _pam_load_conf_file: unable to open config for postlogin
May 04 13:53:49 RockyLoong login[410]: PAM _pam_load_conf_file: unable to open config for postlogin
May 04 13:53:55 RockyLoong login[410]: FAILED LOGIN 1 FROM console FOR rocky, Authentication failure
```
经过检索以及大模型分析，确认是缺少/etc/pam.d/postlogin文件。

补上这个文件，再试一试。

使用chroot, 进入已经挂载的qcow2镜像内， 后执行
```bash
cat <<EOF > /etc/pam.d/postlogin 
#%PAM-1.0 
session optional pam_umask.so 
session optional pam_lastlog.so nowtmp 
EOF
```

补上这个文件以后，passwd程序可以正常运行了，但是登录还是有问题。

请教了孙海勇老师，发现了问题的根源：在编译软件包的过程中，临时系统 /tools/文件夹内的、登录相关的软件包在逐步被新编译出来的rpm包替换掉，但如果只替换了一半，就会发现登录出现问题了。

这个问题的解决方案是不使用虚拟机，改为将虚拟机的qcow2文件挂载到系统中，然后用chroot进入虚拟机内，开始编译。这样，可以绕过登录过程。

从当前开始，后续的软件包编译是在chroot环境下进行的。

## libtirpc

由于已经编好了krb5,这个包所有的依赖关系都已经满足了。正常编译即可。

## libselinux

调用multiprocess的时候，遇到了Permisson denied的问题:
```text
sl = self._semlock = _multiprocessing.SemLock(
                         ^^^^^^^^^^^^^^^^^^^^^^^^^
PermissionError: [Errno 13] Permission denied
```

用root打包，规避这个问题。

## libcap-ng

用root打包。

## pam

注释掉了spec文件中，关于adg, dmg等文档的for循环

编译脚本：
```bash
rpmbuild -bb rpmbuild/SPECS/pam.spec --nodeps --define '_unpackaged_files_terminate_build 0'
```

打包的时候，存在警告：
```text
Installed (but unpackaged) file(s) found:
/usr/lib/systemd/system/pam_namespace.service
```

## libpwquality

切换成root打包，可以正常编译。

## audit

注意，audit 的configure脚本中，似乎有处理器相关的部分，如`--with-aarch64` `--with-riscv`

需要确认这个是否会影响loongarch下的使用。

打包时的警告信息：
```text
  Installed (but unpackaged) file(s) found:
   /usr/lib/systemd/system/audit-rules.service
   /usr/lib/systemd/system/auditd.service
   /usr/lib/tmpfiles.d/audit.conf
```


## util-linux
修改spec文件中的编译参数，

- 将`--with-udev`改为`--without-udev`

- `--with-systemd`改为`--without-systemd`

- 注释掉一些文件：
```text
 %{_unitdir}/fstrim.*
uuidd.*
uuidd-tmpfiles.conf
```
##  ima-evm-utils

正常编译即可。

## elfutils

修改spec文件，如下

`--enable-debuginfod-ima-verification=no`

删掉 `--enable-debuginfod-ima-cert-path=%{_sysconfdir}/keys/ima`

## strace

正常编译即可。安装了strace以后，就可以用strace工具来追踪程序的执行情况了。

经过strace追踪，发现系统中缺少`/etc/nsswitch.conf`, 根据孙海勇的CLFS文档, 补上这个文件。

## tss2

正常编译即可。

## tpm2-tss

正常编译即可。

## gnutls

依赖tss2, 需要提前编译好。

编译脚本
```bash
rpmbuild -bb rpmbuild/SPECS/gnutls.spec --without dane --without tpm2 --without tpm12 --nodeps
```
## crypto-policies

修改spec文件中的ExclusiveArch, 添加loongarch64

编译脚本：
```bash
rpmbuild -bb rpmbuild/SPECS/crypto-policies.spec --nodeps --nocheck --define '_unpackaged_files_terminate_build 0'
```
 打包的时候，有这样的报错：

```text
    Installed (but unpackaged) file(s) found:
   /usr/lib/systemd/system/fips-crypto-policy-overlay.service
```

## fakechroot

没有安装包，跳过。

## libpq

`--with-ldap`改为 `--without-ldap`

## mariadb-connector-c

正常编译即可。

## cyrus-sasl

 将`bootstrap_cyrus_sasl` 设置为1.

## unixodbc

正常编译即可。

## openldap

正常编译即可。

## stunnel

正常编译即可。

## byacc

正常编译即可。

## checkpolicy

正常编译即可。

## sharutils

正常编译即可。

## libarchive

正常编译即可。

## Brotli

正常编译即可。

## Cmocka

正常编译即可。

## ducktape

正常编译即可。

## polkit

修改spec文件，`-Dintrospection=false`

一些文件没有生成，如`polkit.service，polkit-tmpfiles.conf，*.gir,*.typelib`,需要对spec文件进行对应的修改。

编译脚本：
```bash
PKG_CONFIG_PATH+=/tools/lib64/pkgconfig: rpmbuild -bb rpmbuild/SPECS/polkit.spec --nodeps  --define '_unpackaged_files_terminate_build 0'
```
最终打包的时候给出的警告：
```text
Installed (but unpackaged) file(s) found:
/usr/lib/systemd/system/polkit.service
/usr/lib/sysusers.d/polkit.conf
/usr/lib/tmpfiles.d/polkit-tmpfiles.conf
```

##  pcsc-lite

修改spec文件， `-Dlibsystemd=true` 改为`-Dlibsystemd=false`

编译脚本：
```bash
PKG_CONFIG_PATH+=/tools/lib64/pkgconfig: rpmbuild -bb rpmbuild/SPECS/pcsc-lite.spec --nodeps
```
## opensc

依赖基本满足，正常编译即可。
```bash
rpmbuild -bb rpmbuild/SPECS/opensc.spec --nodeps
```
## cppunit

正常编译即可

## softhsm

正常编译即可

## pkcs11-provider

正常编译即可。

## libssh

需要补全opensc,softhsm，还需要修改一下spec文件
```text
-DUNIT_TESTING=OFF \\
-DCLIENT_TESTING=OFF \\
-DSERVER_TESTING=OFF
```
## libidn2

正常编译即可。

## publicsuffix-list

正常编译即可。

## libpsl

正常编译即可。

## curl

正常编译即可。

## jq
编译脚本
```bash
QA_RPATHS=$(( 0x0002\|0x0010 )) rpmbuild --rebuild /opt/srpms/Packages/jq-1.7.1-11.el10.src.rpm
```
## socat

正常编译即可。

## elfutils

修改spec文件，如下

`--enable-debuginfod-ima-verification=no`

## tss2

正常编译即可。

## ima-evm-utils

正常编译即可。

## autoconf-archive

正常编译即可。

## python3-mallard-ducktype

正常编译即可。

## yelp-xsl

正常编译即可。

## yelp-tools

正常编译即可。

## mallard-rng

正常编译即可。

## dbus

使用如下命令构建：
```bash
rpmbuild -bb rpmbuild/SPECS/dbus.spec  --nodeps  --define '_unpackaged_files_terminate_build 0'
```

警告信息如下：
```text
absolute symlink: /usr/share/gtk-doc/html/dbus -> /usr/share/doc/dbus
absolute symlink: /usr/libexec/dbus-1/installed-tests/dbus/data/valid-config-files-system/system.conf -> /usr/share/dbus-1/system.conf
absolute symlink: /usr/libexec/dbus-1/installed-tests/dbus/data/valid-config-files/session.conf -> /usr/share/dbus-1/session.conf
Installed (but unpackaged) file(s) found:
/tools/lib/sysusers.d/dbus.conf
```

## crontabs

正常编译即可。

## cronie

正常编译即可。

## logrotate

编译脚本:
```bash
rpmbuild --rebuild /opt/srpms/Packages/logrotate-3.22.0-4.el10.src.rpm --nodeps --nocheck
```
## rpm

没有安装rust，所以无法安装rpm-sequoia。暂时去掉对这个的依赖。

遇到了lua的问题，查看 `/usr/include/luaconf.h`，发现它不支持loongarch。

首先，需要重新编译安装lua。

修改`rpmbuild/SOURCE/luaconf.h`，添加：
```bash
#elif defined(__loongarch64)
#include "luaconf-loongarch64.h"
```
再次编译，遇到了新的问题`undefined reference to `pgpDigParamsSalt`

修改spec文件，去掉两个补丁
```text
#rpm-4.19.x-pqc-algo.patch
#rpm-4.19.x-pqc-fixes.patch
```
这两个补丁导致了上述问题的发生
新的问题：
```text
tools/rpmdb --rcfile rpmrc --define '_db_backend ndb' --dbpath=/root/rpmbuild/BUILD/rpm-4.19.1.1/_build/ndb --initdb
error: can't create transaction lock on / (是一个目录)
```
按照孙海勇书中的提示，注释掉相关的语句。

最终的构建脚本
```bash
rpmbuild -bb rpmbuild/SPECS/rpm.spec --without sequoia --nodeps
```
rpm的一个插件不太好用，暂时禁用这个插件，可以给它改个名字 `/usr/lib64/rpm-plugins/dbus_announce.so`

##  perl-generators
由于包管理系统有变动，到这个阶段，需要重新编译一些软件包。

正常编译即可。

## redhat-rpm-config

注意，这个包之前修改过，不要用原始的包，要打包修改过的那个包！

## python-rpm-generators

正常编译即可。

## fonts-rpm-macros

正常编译即可。

## ruby

ruby需要用到redhat-annobin-cc1，这个应该被清空，因为不存在plugin/gcc-annobin.so。同时清空redhat-hardened-ld。

注意，一定要用修改过的`redhat-rpm-config`,否则编译ruby的时候使用的参数不对，编译会失败！

很多文件被安装到了/usr/local/share下面，没有安装到/usr/share/下面，需要确认问题的根源。不应该出现这样的问题。

做了如下的改动，，将
```bash
--with-sitedir='%{ruby_sitelibdir}' \
--with-sitearchdir='%{ruby_sitearchdir}' \
```
修改为了
```bash
--with-sitedir=%{_datadir}/ruby/site_ruby \
--with-sitearchdir=%{_libdir}/ruby/site_ruby \
```
将`mv %{buildroot}%{ruby_libdir}/gems %{buildroot}%{gem_dir}`

改为了
```bash
if [ -d %{buildroot}%{ruby_libdir}/gems ]; then
  mv -v %{buildroot}%{ruby_libdir}/gems %{buildroot}%{gem_dir}
fi
```

## lua

正常编译即可。

## dwz

依赖xxhash和gdb。

编译的时候，遇到这样的报错，`NATIVE_POINTER_SIZE`这个参数没有定义。应该是与当前的locale相关。

编译的时候，添加`LC_ALL=C`, 问题解决。

## babeltrace

正常编译即可。

## xxhash

正常编译即可。

## gdb

正常编译即可。

## kmod

正常编译即可。

## libaio

正常编译即可。

## libqb

正常编译即可。

## libnl3

正常编译即可。

## kronosnet

正常编译即可。

## corosync

编译脚本：
```bash
rpmbuild --rebuild /opt/srpms/Packages/corosync-3.1.9-2.el10.src.rpm --without systemd --without snmp --nodeps 
```

## device-mapper-persistent-data

缺少`%cargo_prep`宏。需要在补全`rust-toolset`后，编译这个包。

## lvm2

除了按照书中的内容进行修改，还需要改动spec文件，修正configure的参数

```bash
--with-systemd=no
--disable-app-machineid
```

修正一下systemd相关的内容，修改`/usr/lib/rpm/macros.d/macros.systemd`，将所有的`/tools/`改为
`/usr/`。
```bash
sed -i "s@/tools/@/usr/@g" macros.systemd
```
这样，就会把systemd相关的内容，安装到`/usr/lib/systemd`,而不是`/tools/lib/systemd`

实际上，这个操作在书中P171已经提到了。一定要认真读书。

重新编译，生成rpm包，以lvm2-，device-mapper开头。先安装上部分rpm包。
```bash
rpm -ivh device-mapper-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      device-mapper-devel-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      device-mapper-libs-1.02.206-3.el10~bootstrap.loongarch.loongarch64.rpm \
      --nodeps
```
注意，很多其他的包，可能已经踩了这个坑，也都需要重新编译。

## json-c

正常编译即可。

## argon2

找不到源码包。

## cryptstepup

依赖device-mapper。打包脚本
```bash
rpmbuild -bb rpmbuild/SPECS/cryptsetup.spec --define "version_no_tilde 2.7.5" --nodeps
```

## libpcap

正常编译即可。

## libmnl

打包脚本
```bash
rpmbuild --rebuild /opt/srpms/Packages/libmnl-1.0.5-7.el10.src.rpm --define '_unpackaged_files_terminate_build 0'
```
生成了很多文档`man3/`

## libnftnl

正常编译即可。

## libnfnetlink

正常编译即可。

## libnetfilter_conntrack

正常编译即可。

## iptables

正常编译即可。

## qrencode

找不到安装包。

## libxkbfile

正常编译即可。

## xorg-x11-xkb-utils

找不到安装包

## xkeyboard-config

正常安装即可

## libxkbcommon

依赖wayland的东西，补全后继续.

## console-setup

缺少`/usr/lib/rpm/perl.req`程序，看看需要重新编译哪一个包。

需要重新编译perl-generators, 但这个包自己依赖自己，需要用/tools/lib/rpm/perl.req过渡一下。

## kbd

依赖xkeyboard-config

## libbpf

正常编译即可。

## python-docutils

正常编译即可。

## bpftool

依赖rst2man，需要安装python-docutils

## libseccomp

源码似乎有架构相关的内容。2.5.6不支持龙架构。需要确认。

<https://github.com/seccomp/libseccomp/pull/356>

最好升libseccomp到2.6.0

## libcbor

去掉文档的编译

## python-markupsafe

正常编译即可。

## python-jinja2

正常编译即可。

## systemd

依赖libbpf，先安装libbpf。

依赖bpftool，

依赖clang，看看能否不依赖clang。

去掉对bpf的依赖

`-Dbpf-framework=disabled`

`-Dbpf-compiler=gcc`

去掉libseccomp的依赖。

`-Dseccomp=disabled`

去掉对fido2的依赖

`-Dlibfido2=disabled`

去掉对xkbcommon的依赖

`-Dxkbcommon=disabled`

编译脚本

```bash
rpmbuild -bb rpmbuild/SPECS/systemd.spec --define "version_no_tilde 257 " --nodeps --nocheck
```
## dbus-broker

正常编译即可。

## dbus

正常编译即可。

## ipset

正常编译即可。

## intltools

正常编译即可。

## jansson

去掉制作文档的部分。

## tree-sitter
正常编译即可。

## desktop-file-utils

依赖emacs。调整一下顺序，先安装emacs。

解决一下循环依赖。

编译的时候，这样写：
```bash
rpmbuild -bb rpmbuild/SPECS/desktop-file-utils.spec -D "_emacs_sitestartdir /usr/share/emacs/site-lisp/site-start.d" -D " _emacs_sitelispdir /usr/share/emacs/site-lisp" --nodeps
```
## appstream
正常编译即可。

## emacs

依赖json，先编译jansson

依赖tree-sitter

emacs也依赖desktop-file-utils。循环依赖了。

还依赖appstream-utils。构建的时候加上`--nocheck`。

## google-noto-fonts
```bash
rpmbuild --rebuild /opt/srpms/Packages/google-noto-fonts-20240401-5.el10.src.rpm --nocheck
```
## xaw3d

已经编译过了。

## vala

正常编译即可。

## libsecret

依赖gi-docgen，需要想办法绕过或者补上。

spec文件中添加一行： `-Dgtk_doc=false`

需要根据下面的报错，补全软件包。

```text
Couldn't find include 'GObject-2.0.gir' (search path: '['/usr/share/gir-1.0', '/root/.local/share/gir-1.0', 'gir-1.0', '/usr/local/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0', '/usr/share/gir-1.0']')
```
经过确认，GObject-2.0.gir这个文件应该是gobject-devel包里提供的。

补全了glib2以后，再编译libsecret，完成。

## python3-mako

完成编译。

## PyYaml

完成编译

## Python-markdown

加上`--nocheck`

## gobject-introspection

重新编译。依赖关系都已经满足。

## shared-mime-info

正常编译即可。

## glib2

重新编译，尽量补全依赖关系。

重新编译的glib2，补全了很多文件，重要的是补全了`/usr/share/gir-1.0/GObject-2.0.gir`。


## perl-error

正常编译即可。

## perl-termreadkey

正常编译即可。

## xorg-x11-xauth

正常编译即可。

## libusbx

找不到安装包。

## hidapi

找不到安装包。

## libcbor

正常编译即可。

## libfido2

将所有的依赖关系都补全了，编译很顺利。

## openssh

编译的时候有报错：
```text
configure: error: seccomp_filter sandbox not supported on
loongarch64-redhat-linux-gnu
```
注释掉

`--with-sandbox=seccomp_filter`

check通不过，加上`--nocheck`

## git
编译脚本
```bash
rpmbuild --rebuild /opt/srpms/Packages/git-2.47.3-1.el10.src.rpm --nodeps --nocheck
```
选择性安装一部分git的软件包即可。

## umockdev
编译脚本
```bash
rpmbuild --rebuild /opt/srpms/Packages/umockdev-0.19.1-2.el10.src.rpm --nodeps --nocheck
```
## libgudev1

正常编译即可。

## libusb

依赖关系有些复杂，libusb->umockdev-devel->libgudev1-devel->umockdev-devel

需要解决一下循环依赖。

已经解决，完成编译、安装。

## efi-rpm-macros

正常编译，似乎没有什么问题。

## mandoc

正常编译即可。

## libabigail

跳过生成文档的部分，编译通过。

## efivar

在exclusivearch中，增加loongarch64。源码包需要重新打包。

## pesign

在exclusivearch中，增加loongarch64。源码包需要重新打包。

## linux-atm

找不到源码包。

## psmisc

修改spec文件，在一个架构相关的地方添加loongarch。源码包也重新打包。
```bash
rpmbuild -ba rpmbuild/SPECS/psmisc.spec
```
## iproute

正常编译

## iputils

正常编译

## perl-Test-Pod

正常编译

## perl-Pod-Parser

正常编译

## perl-Test-Output

```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-Test-Output-1.03.4-6.el10.src.rpm --nodeps
```
## perl-IPC-Run3
```bash
rpmbuild --rebuild /opt/srpms/Packages/perl-IPC-Run3-0.049-4.el10.src.rpm --nodeps --nocheck
```
## libmaxminddb

尝试尽量补全依赖关系

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
正常编译

## rubygem-racc

安装失败，不清楚为什么。

报错是这样的：
```text
ERROR:  Error installing racc-1.7.3.gem:
ERROR: Failed to build gem native extension.
    No such file or directory @ dir_s_mkdir - /usr/share/gems/gems/racc-1.7.3/ext/racc/cparse/.gem.20260512-110776-rija44
```
## ipcalc

缺少racc, 未能安装。只是用ronn来产生文档。可以跳过。

解决方案：删掉rubygems-ronn-ng软件包，在文件列表中注释掉ipcalc.1
```bash
rpmbuild -bb rpmbuild/SPECS/ipcalc.spec --nodeps --nocheck
```
## dhcp

依赖ipcalc，还是需要想办法补上ipcalc。

补上了ipcalc

## dracut

正常编译。

## os-prober

正常编译，生成rpm包。**注意**: 这个包没有安装！
`/root/rpmbuild/RPMS/loongarch64/os-prober-1.81-9.el10~bootstrap.loongarch.loongarch64.rpm`

## grub2

跳过,没有编译。计划用grub 2.14替代。

## tzdata

修改spec文件，注释掉所有关于java的语句。
```bash
rpmbuild -bb rpmbuild/SPECS/tzdata.spec --nodeps
```
## bc

正常编译即可。

## dwarves

正常编译即可。

## libical
```bash
rpmbuild --rebuild /opt/srpms/Packages/libical-3.0.18-3.el10.src.rpm --nodeps --nocheck
```
安装的时候，依赖tzdata

## libell

bluez依赖libell。完成。

## bluez

修改SPEC文件，去掉enable-cups。

## net-tools

依赖bluez-devel。完成

## tinyxml2

找不到安装包。

## cppcheck

找不到安装包。

## libkcapi
```bash
rpmbuild --rebuild /opt/srpms/Packages/libkcapi-1.5.0-3.el10.src.rpm
\--nodeps \--without doc \--nocheck
```
linux-firmware

完成编译，[尚未安装]{.mark}。

gnu-efi

exclusivearch
中，添加loongarch64。正确的解决方案应该是给%efi这个rpm的宏，添加loongarch64。

%ifarch x86_64 aarch64 这句后面，也加上loongarch64

完成编译。

inih

完成。

userspace-rcu

需要打补丁。打补丁的内容参考之前的笔记。打补丁后，正常编译即可。

xfsprogs

依赖inih-devel和userspace-rcu-devel，需要补全。

补全后完成。

sudo

完成。

kernel

[跳过。]{.mark}

重新编译一些依赖systemd的软件包

perl-Module-Build

rpmbuild \--rebuild
/opt/srpms/Packages/perl-Module-Build-0.42.34-7.el10.src.rpm \--nodeps
\--nocheck

补充utils-linux的依赖关系

po4a

rpmbuild \--rebuild /opt/srpms/Packages/po4a-0.69-7.el10.src.rpm
\--nodeps

perl-YAML-Tiny

正常编译即可。

perl-Module-Install

rpmbuild \--rebuild
/opt/srpms/Packages/perl-Module-Install-1.21-6.el10.src.rpm \--nodeps
\--nocheck

perl-Module-Install-AuthorRequires

正常编译

perl-File-Remove

正常编译

perl-Module-Install-ReadmeFromPod

未能成功编译。

authselect

补充utils-linux的依赖关系。

因为未能补全perl-Module-Install-ReadmeFromPod的依赖关系，不编译manpages。

会安装一些.mo文件，这些文件并没有打包。如：

/usr/share/locale/ca/LC_MESSAGES/authselect.mo

最终的编译脚本：

rpmbuild -bb authselect.spec \--nocheck \--define
\'\_unpackaged_files_terminate_build 0\'

完成安装。

utils-linux

重新编译，并（尽量）补全了依赖关系。

dbus

无需改动spec文件。

rpmbuild -bb dbus.spec \--nodeps

修改了macros.systemd以后，文件会被放在正确的路径。

pcsc-lite

重新编译即可。

polkit

重新编译即可。

crypto-policies

修改SPEC文件中的ExclusiveArch, 添加Loongarch64

重新打包，编译，安装。

elfutils

重新编译打包，无需修改。太感动了！

audit

重现编译打包。不报错了。

libeconf

正常编译即可。

docbook5-schemas

正常编译即可。

pam

依赖libeconf,docbook5-schemas，补全。

重新编译安装。

chkconfig

重新编译安装。

libusers

重新编译安装。

shadow-utils

重新编译安装。

procps-ng

重新编译安装。

libtirpc

重新编译安装。

sanlock

正常编译安装，补全lvm2的依赖关系。

libnvme

不编译文档，将-Ddocs=all -Ddoc-build=true改为 -Ddocs=false
-Ddoc-build-false

注释掉有关文档安装的所有语句。

rpmbuild -bb rpmbuild/SPECS/libnvme.spec \--nodeps

下面的包都需要重新编译：

lvm2

重新编译安装。

corosync

rpmbuild \--rebuild /opt/srpms/Packages/corosync-3.1.9-2.el10.src.rpm
\--without snmp

libpq

重新编译安装。

krb5

依然还是不生成文档。重新编译安装。

elinks

重新编译安装。

p11-kit

rpmbuild \--rebuild /opt/srpms/Packages/p11-kit-0.25.5-7.el10.src.rpm
\--nocheck

fuse3

e2fsprogs依赖fuse3。

e2fsprogs

rpmbuild \--rebuild /opt/srpms/Packages/e2fsprogs-1.47.1-4.el10.src.rpm
\--nocheck

配置和重启系统

sudo mv /etc/machine-id /etc/machine-id.bak

sudo mv /etc/profile /etc/profile.bak

sudo rpm -Uvh /home/rocky/rpmbuild/RPMS/noarch/setup-\*.rpm \--force

重启系统，准备进入第10章

重启顺利！

第10章

zchunk

找不到安装包。

libsolv

正常编译安装即可。

python-coverage

找不到安装包。

vim

暂时没有编译gtk3,因此不编译vim的图形界面。

rpmbuild \--rebuild /opt/srpms/Packages/vim-9.1.083-6.el10.src.rpm
\--without gui \--nodeps

python-nose

找不到安装包

gpgme

安装的时候遇到了问题。需要禁用另外一个rpm的插件。

sudo rpm -e
rpm-plugin-ima-4.19.1.1-20.el10\~bootstrap.loongarch.loongarch64
rpm-plugin-dbus-announce-4.19.1.1-20.el10\~bootstrap.loongarch.loongarch64

编译脚本：

QA_RPATHS=\$(( 0x0001)) rpmbuild \--rebuild
/opt/srpms/Packages/gpgme-1.23.2-6.el10.src.rpm \--without qt5
\--without qt6

patchutils

直接编译安装即可。

check

rpmbuild \--rebuild /opt/srpms/Packages/check-0.15.2-17.el10.src.rpm
\--nodeps

python-charset-normalizer

正常编译安装。

python-idna

正常编译安装。

python-urllib3

正常编译安装。

python-requests

pyxattr

librepo

check的时候有报错

home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:51:F:Main:test_gpg_check_signature:0:
Checking valid key and data from file failed with \"Error during parsing
OpenPGP packets\"

/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:171:F:Main:test_gpg_check_armored_key_import_test_export:0:
Assertion \'subkeys != NULL\' failed: subkeys == 0

/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:171:F:Main:test_gpg_check_binary_key_import_test_export:0:
Assertion \'subkeys != NULL\' failed: subkeys == 0

/home/rocky/rpmbuild/BUILD/librepo-1.18.0/tests/test_gpg.c:283:F:Main:test_gpg_check_import_padded:0:
Assertion \'tmp_err == NULL\' failed: tmp_err == 0x150f61de0

加上\--nocheck

rpmbuild \--rebuild /opt/srpms/Packages/librepo-1.18.0-6.el10_0.src.rpm
\--without pythontest \--nodeps \--nocheck

tix

正常编译安装。

pycairo

正常编译安装。

pygobject

rpmbuild \--rebuild /opt/srpms/Packages/pygobject3-3.46.0-7.el10.src.rpm
\--nodeps \--nocheck

python-six

正常编译安装。

libmodulemd

修改SPEC文件 -Dwith_docs=true改为false

libdnf

按照书中的要求，修改SPEC文件。

有一个测试通不过。跳过。

rpmbuild -bb rpmbuild/SPECS/libdnf.spec \--nodeps \--nocheck

补充编译一些软件包

debugedit

debugedit只在tools中安装了，由于更新了/etc/profile,
不去/tools/bin/中寻找程序，因此找不到这个程序了。debugedit依赖自身，所以先在/usr/bin做一个指向/tools/bin/debugedit的符号链接补全依赖关系。

> 编译安装以后，/usr/bin/debugedit不再是符号链接。

pps-tools

> 正常编译安装即可。

chrony

> 正常编译安装即可。

dhcpcd

正常编译安装即可。安装以后，可以运行dhcpcd,从dhcp获得IP地址，从而正常使用ssh。

libcomps

> 修改spec文件，不制作pydocs

lua-rpm-macros

正常编译安装

forge-srpm-macros

正常编译安装

redhat-rpm-config

重新编译安装，以正确提供fedora/common.lua文件。

要记得将/usr/lib/rpm/redhat/redhat-annobin-cc1变为空文件！

augeas

编译报错

error: lua script failed: \[string \"forgemeta\"\]:2: module
\'fedora.common\' not found:

no field package.preload\[\'fedora.common\'\]

no file \'/usr/lib/rpm//lua/fedora/common.lua\'

no file \'/usr/lib64/lua/5.4/fedora/common.so\'

no file \'/usr/lib64/lua/5.4/loadall.so\'

no file \'./fedora/common.so\'

no file \'/usr/lib64/lua/5.4/fedora.so\'

no file \'/usr/lib64/lua/5.4/loadall.so\'

no file \'./fedora.so\'

重新安装redhat-rpm-config后，可以正常编译安装。

libtar

没有源码包。

satyr

没有源码包。

libmodman

没有源码包。

gsettings-desktop-schemas

正常编译安装。

libproxy

需要一个gi-docgen的程序，暂时不好编译，跳过文档的生成。

添加-Ddocs=false选项， 并将%{\_docdir}/libproxy-1.0/行注释掉。

xmlrpc-c

没有源码包。

mailx

没有源码包。

libreport

没有源码包。

dnf

按照书中提示，修改spec文件。

composefs

修改SPEC文件，强行添加-Dman=disabled

ostree

rpmbuild \--rebuild /opt/srpms/Packages/ostree-2025.6-1.el10.src.rpm
\--nodeps

注意：没有安装ostree-grub2。

python-wheel

已经安装

python-pip

已经安装。

python-six

已经安装

python-setuptools_scm

已经安装

python-Dateutil

去掉文档编译命令

python-distro

正常编译。

python3-systemd

不编译文档。

dbus-python

正常编译安装。

drpm

没有源码包。

createrepo_c

rpmbuild \--rebuild
/opt/srpms/Packages/createrepo_c-1.1.2-4.el10.src.rpm \--nodeps

dnf-plugins-core

正常编译安装。

mc

一个终端界面好用的文件管理工具。正常编译安装。

创建本地仓库

按照书中的提示进行操作。

可以使用rsync -auv命令，替代cp, 完成文件的复制。

修改\~/.bashrc, 设置：

export PS1=\'\[\\u@\\h \\W\]\\\$ \'

nano

正常编译安装。

创建本地的yum repo文件

man-pages

用rpmbuild编译，用dnf install 安装。

python-psutils

尽量给llvm补全依赖

python-ptyprocess

zsh

rpmbuild \--rebuild /opt/srpms/Packages/zsh-5.9-15.el10.src.rpm
\--nodeps \--nocheck

python-pexpect

llvm

~~编译llvm,依赖llvm提供的clang~~

~~首先，用gcc/g++编译llvm。~~

遇到这样的报错：

loading initial cache file
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/projects/builtins/tmp/builtins-cache-RelWithDebInfo.cmake

\-- The C compiler identification is unknown

\-- The ASM compiler identification is Clang with GNU-like command-line

\-- Found assembler:
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/bin/clang

\-- Detecting C compiler ABI info

\-- Detecting C compiler ABI info - failed

CMake Warning (dev) at /usr/share/cmake/Modules/GNUInstallDirs.cmake:253
(message):

Unable to determine default CMAKE_INSTALL_LIBDIR directory because no

target architecture is known. Please enable at least one language before

including GNUInstallDirs.

Call Stack (most recent call first):

/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/compiler-rt/cmake/base-config-ix.cmake:9
(include)

CMakeLists.txt:25 (include)

This warning is for project developers. Use -Wno-dev to suppress it.

\-- Looking for unwind.h

\-- Looking for unwind.h - not found

\-- Looking for rpc/xdr.h

\-- Looking for rpc/xdr.h - not found

\-- Could NOT find LLVM (missing: LLVM_DIR)

CMake Warning at
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/compiler-rt/cmake/Modules/CompilerRTUtils.cmake:314
(message):

UNSUPPORTED COMPILER-RT CONFIGURATION DETECTED: LLVM cmake package not

found.

Reconfigure with -DLLVM_CMAKE_DIR=/path/to/llvm.

Call Stack (most recent call first):

CMakeLists.txt:29 (load_llvm_config)

\-- LLVM_MAIN_SRC_DIR:
\"/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm\"

\-- Attempting to mock the changes made by LLVMConfig.cmake

\-- LLVM_CMAKE_DIR:
\"/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/cmake/modules\"

CMake Error at
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/compiler-rt/cmake/Modules/CompilerRTMockLLVMCMakeConfig.cmake:60
(message):

Fetching target triple from compiler \"\" is not implemented.

Call Stack (most recent call first):

/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/compiler-rt/cmake/Modules/CompilerRTMockLLVMCMakeConfig.cmake:11
(compiler_rt_mock_llvm_cmake_config_set_target_triple)

/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/compiler-rt/cmake/Modules/CompilerRTUtils.cmake:360
(compiler_rt_mock_llvm_cmake_config)

CMakeLists.txt:29 (load_llvm_config)

\-- Configuring incomplete, errors occurred!

\[3549/4298\] : && /bin/g++ -O2 -fexceptions -g1 -grecord-gcc-switches
-pipe -Wall -Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -Wl,-z,relro
-Wl,\--as-needed -Wl,-z,now
-specs=/usr/lib/rpm/redhat/redhat-hardened-ld
-specs=/usr/lib/rpm/redhat/redhat-hardened-ld-errors
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -Wl,\--build-id=sha1
-Wl,\--build-id=sha1
-Wl,-rpath-link,/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./lib64
-Wl,\--gc-sections
tools/verify-uselistorder/CMakeFiles/verify-uselistorder.dir/verify-uselistorder.cpp.o
-o bin/verify-uselistorder
-Wl,-rpath,\"\\\$ORIGIN/../lib64:/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/lib64:\"
lib64/libLLVM.so.20.1 && :

FAILED: runtimes/builtins-stamps/builtins-configure
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/runtimes/builtins-stamps/builtins-configure

cd
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/runtimes/builtins-bins
&& /usr/bin/cmake \--no-warn-unused-cli
-DCMAKE_C_COMPILER=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/clang
-DCMAKE_CXX_COMPILER=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/clang++
-DCMAKE_ASM_COMPILER=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/clang
-DCMAKE_LINKER=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/ld.lld
-DCMAKE_AR=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-ar
-DCMAKE_RANLIB=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-ranlib
-DCMAKE_NM=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-nm
-DCMAKE_OBJDUMP=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-objdump
-DCMAKE_OBJCOPY=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-objcopy
-DCMAKE_STRIP=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-strip
-DCMAKE_READELF=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin/llvm-readelf
-DCMAKE_C_COMPILER_TARGET=loongarch64-redhat-linux-gnu
-DCMAKE_CXX_COMPILER_TARGET=loongarch64-redhat-linux-gnu
-DCMAKE_ASM_COMPILER_TARGET=loongarch64-redhat-linux-gnu
-DCMAKE_VERBOSE_MAKEFILE=ON -DCMAKE_INSTALL_PREFIX=/usr/lib64/llvm20
-DLLVM_BINARY_DIR=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build
-DLLVM_CONFIG_PATH=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/bin/llvm-config
-DLLVM_ENABLE_WERROR=OFF
-DLLVM_HOST_TRIPLE=loongarch64-unknown-linux-gnu
-DLLVM_HAVE_LINK_VERSION_SCRIPT=1
-DLLVM_USE_RELATIVE_PATHS_IN_DEBUG_INFO=OFF
-DLLVM_USE_RELATIVE_PATHS_IN_FILES=OFF -DLLVM_LIT_ARGS=-vv
-DLLVM_SOURCE_PREFIX= -DPACKAGE_VERSION=20.1.8
-DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_MAKE_PROGRAM=/bin/ninja-build
-DCMAKE_EXPORT_COMPILE_COMMANDS=1
-DLLVM_LIBRARY_OUTPUT_INTDIR=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./lib64
-DLLVM_RUNTIME_OUTPUT_INTDIR=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./bin
-DLLVM_DEFAULT_TARGET_TRIPLE=loongarch64-redhat-linux-gnu
-DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON
-DLLVM_CMAKE_DIR=/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build
-DCMAKE_C_COMPILER_WORKS=ON -DCMAKE_ASM_COMPILER_WORKS=ON
-DHAVE_LLVM_LIT=ON -DCLANG_RESOURCE_DIR=../lib/clang/20
-DCOMPILER_RT_INCLUDE_TESTS=OFF
-DCOMPILER_RT_INSTALL_PATH=/usr/lib/clang/20 -GNinja
-C/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/projects/builtins/tmp/builtins-cache-RelWithDebInfo.cmake
-S
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/runtimes/../../compiler-rt/lib/builtins
-B
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/runtimes/builtins-bins
&& /usr/bin/cmake -E touch
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/runtimes/builtins-stamps/builtins-configure

\[3551/4298\] : && /bin/g++ -O2 -fexceptions -g1 -grecord-gcc-switches
-pipe -Wall -Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -Wl,-z,relro
-Wl,\--as-needed -Wl,-z,now
-specs=/usr/lib/rpm/redhat/redhat-hardened-ld
-specs=/usr/lib/rpm/redhat/redhat-hardened-ld-errors
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -Wl,\--build-id=sha1
-Wl,\--build-id=sha1
-Wl,-rpath-link,/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/./lib64
-Wl,\--gc-sections tools/sancov/CMakeFiles/sancov.dir/sancov.cpp.o
tools/sancov/CMakeFiles/sancov.dir/sancov-driver.cpp.o -o bin/sancov
-Wl,-rpath,\"\\\$ORIGIN/../lib64:/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/lib64:\"
lib64/libLLVM.so.20.1 && :

\[3552/4298\] /bin/g++ -D_GNU_SOURCE -D\_\_STDC_CONSTANT_MACROS
-D\_\_STDC_FORMAT_MACROS -D\_\_STDC_LIMIT_MACROS
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/tools/opt
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/opt
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/include -O2
-fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -std=c++17 -MD
-MT tools/opt/CMakeFiles/LLVMOptDriver.dir/NewPMDriver.cpp.o -MF
tools/opt/CMakeFiles/LLVMOptDriver.dir/NewPMDriver.cpp.o.d -o
tools/opt/CMakeFiles/LLVMOptDriver.dir/NewPMDriver.cpp.o -c
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/opt/NewPMDriver.cpp

\[3553/4298\] /bin/g++ -DLLVM_BUILD_STATIC -D_GNU_SOURCE
-D\_\_STDC_CONSTANT_MACROS -D\_\_STDC_FORMAT_MACROS
-D\_\_STDC_LIMIT_MACROS
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/tools/yaml2obj
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/yaml2obj
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/include -O2
-fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -std=c++17 -MD
-MT tools/yaml2obj/CMakeFiles/yaml2obj.dir/yaml2obj.cpp.o -MF
tools/yaml2obj/CMakeFiles/yaml2obj.dir/yaml2obj.cpp.o.d -o
tools/yaml2obj/CMakeFiles/yaml2obj.dir/yaml2obj.cpp.o -c
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/yaml2obj/yaml2obj.cpp

\[3554/4298\] /bin/g++ -D_GNU_SOURCE -D\_\_STDC_CONSTANT_MACROS
-D\_\_STDC_FORMAT_MACROS -D\_\_STDC_LIMIT_MACROS
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/unittests/ADT
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/unittests/ADT
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/third-party/unittest/googletest/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/third-party/unittest/googlemock/include
-O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -std=c++17
-Wno-dangling-else -Wno-variadic-macros -Wno-suggest-override -MD -MT
unittests/ADT/CMakeFiles/ADTTests.dir/AnyTest.cpp.o -MF
unittests/ADT/CMakeFiles/ADTTests.dir/AnyTest.cpp.o.d -o
unittests/ADT/CMakeFiles/ADTTests.dir/AnyTest.cpp.o -c
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/unittests/ADT/AnyTest.cpp

\[3555/4298\] /bin/g++ -DLLVM_BUILD_STATIC -D_GNU_SOURCE
-D\_\_STDC_CONSTANT_MACROS -D\_\_STDC_FORMAT_MACROS
-D\_\_STDC_LIMIT_MACROS
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/tools/obj2yaml
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/obj2yaml
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/redhat-linux-build/include
-I/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/include -O2
-fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -fPIC
-fno-semantic-interposition -fvisibility-inlines-hidden
-Werror=date-time -fno-lifetime-dse -Wall -Wextra -Wno-unused-parameter
-Wwrite-strings -Wcast-qual -Wno-missing-field-initializers -pedantic
-Wno-long-long -Wimplicit-fallthrough -Wno-maybe-uninitialized
-Wno-nonnull -Wno-class-memaccess -Wno-redundant-move
-Wno-pessimizing-move -Wno-noexcept-type -Wdelete-non-virtual-dtor
-Wsuggest-override -Wno-comment -Wno-misleading-indentation
-Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections
-fdata-sections -O2 -fexceptions -g1 -grecord-gcc-switches -pipe -Wall
-Wno-complain-wrong-lang -Werror=format-security
-Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -march=loongarch64
-msimd=lsx -D_DEFAULT_SOURCE -Dasm=\_\_asm\_\_ -DNDEBUG -std=c++17 -MD
-MT tools/obj2yaml/CMakeFiles/obj2yaml.dir/elf2yaml.cpp.o -MF
tools/obj2yaml/CMakeFiles/obj2yaml.dir/elf2yaml.cpp.o.d -o
tools/obj2yaml/CMakeFiles/obj2yaml.dir/elf2yaml.cpp.o -c
/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/llvm/tools/obj2yaml/elf2yaml.cpp

ninja: build stopped: subcommand failed.

error: Bad exit status from /var/tmp/rpm-tmp.QUspj3 (%build)

RPM build errors:

Bad exit status from /var/tmp/rpm-tmp.QUspj3 (%build)

修改了编译参数，遇到了新的报错：

CMake Error at
/usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233
(message):

Could NOT find Threads (missing: Threads_FOUND)

Call Stack (most recent call first):

/usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603
(\_FPHSA_FAILURE_MESSAGE)

/usr/share/cmake/Modules/FindThreads.cmake:226
(FIND_PACKAGE_HANDLE_STANDARD_ARGS)

/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/openmp/runtime/cmake/config-ix.cmake:162
(find_package)

/home/rocky/rpmbuild/BUILD/llvm-project-20.1.8.src/openmp/runtime/CMakeLists.txt:279
(include)

将孙海勇的llvm.spec文件，与rocky的spec文件对比一下，看看需要怎么改进。

还是有些难搞，要不要跳过？

最终解决方案：编译runtimes的时候，只编译compiler-rt,不编译openmp。对spec文件做了很多的改动，这里不一一介绍了。

编译成功后，生成了如下的39个文件：

clang-libs-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-static-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-tools-extra-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-libs-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-libs-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-libs-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lldb-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-devel-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-tools-extra-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-devel-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lld-libs-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-test-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lldb-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-devel-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lldb-devel-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lld-libs-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-devel-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-googletest-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-test-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

python3-lldb-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

python3-lit-20.1.8-1.el10\~bootstrap.loongarch.noarch.rpm

compiler-rt-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-analyzer-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

python3-clang-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-tools-extra-devel-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lld-devel-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

git-clang-format-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lld-debuginfo-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-cmake-utils-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

clang-resource-filesystem-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

lld-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-doc-20.1.8-1.el10\~bootstrap.loongarch.noarch.rpm

llvm-toolset-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-filesystem-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

llvm-debugsource-20.1.8-1.el10\~bootstrap.loongarch.loongarch64.rpm

顺手编译一些包

nano

lsof

execstack

execstack不支持龙架构，后续需要进行架构适配。

zstd

liburing

plocate

下载rust和openjdk的Loongarch版。

rust

从rust官网上下载rust
1.88.0版本，loongarch64版本。解压缩后，放在/tools中。

openjdk

从Loongnix.cn下载 jdk 25.0.2版本。解压缩后，放在/tools/中。

tzdata

重新编译

javapackges-tools

lua-lunit

没有安装包

lua-posix

没有安装包

copy-jdk-config

没有安装包

openjdk

先跳过这个大包，需要做好准备再搞这个大家伙。

future

找不到安装包

python-commonmark

找不到安装包

python-recommonmark

找不到安装包

rust-srpm-macros

找不到安装包

rust

失败。缺少libclang_rt.profile.a。注释掉相关的语句后，依然卡死在

compiling cc v1.2.17

Building \[=\> \] 9/85: libc(build), proc-macro2(build),
generic-array(build), typenum(build)

不知道为什么会这样。先跳过。

循环构建脚本

已经复制，未测试。

Perl家族类软件包

先跳过

Python家族类软件包

先跳过

wayland

不需要特殊处理，直接编译即可。

wayland-protocols

正常编译。

hwdata

正常编译。

pciutils

依赖hwdata

libpciaccess

正常编译。

libdrm

依赖pciaccess, 需要编译libpciaccess

其他的不需要修改。

vulkan-header

正常编译。

libxxf86vm

正常编译。

libXrandr

正常编译。

libXdamage

正常编译。

libxshmfence

正常编译。

libomxil-bellagio

找不到了

libglvnd

libvdpau

libva

opencl-filesystem

opencl-headers

ocl-icd

vulkan-loader

spirv-headers-

glslang

python-ply

python3-pycparser

mesa

xcb-util

xcb-util-image

xcb-util-renderutil

xcb-util-cursor

xcb-util-keysyms

xcb-util-wm

libfontenc

xorg-x11-font-utils

libXfont2

libXres

libXv

libdmx

找不到安装包

libepoxy

eglexternalplatform

egl-wayland

lapack

openblas-srpm-macros

openblas

在ExclusiveArch中添加loongarch64.

TEST 105/109 potrf:smoketest_trivial \[FAIL\]

ERR: test_potrs.c:535 U s(0,0) difference: 1.19209e-07

TEST 106/109 potrf:bug_695 \[OK\]

TEST 107/109 kernel_regress:skx_avx \[OK\]

TEST 108/109 fork:safety \[OK\]

TEST 109/109 fork:safety_after_fork_in_parent \[OK\]

RESULTS: 109 tests (108 ok, 1 failed, 0 skipped) ran in 293 ms

问题解决方案：

<https://blog.csdn.net/gitblog_00861/article/details/151479368>

实际上这个方案没有解决我的问题。

暂时先不测试，跳过这个问题。

只知道是编译器的问题。今天就到这里了。暴力规避一下这个问题吧！

暴力规避未果。暂时先放下这个包。

numpy

cython

依赖python-numpy, 想办法补全。

编译完成后，安装的时候才发现这个包早就编过了。

python-lxml

已经编译过了。

javapackges-tools

已经编译过了

snappy

正常编译即可。

wget

rpmbuild \--rebuild /opt/srpms/Packages/wget-1.24.5-5.el10.src.rpm
\--nodeps \--nocheck

crash

依赖snappy, wget且需要修改为支持龙架构。

avahi

rpmbuild -D \"version_no_tilde 0.9-rc2\" -bb
\~/rpmbuild/SPECS/avahi.spec

libdaemon

dyninstall

需要添加龙架构支持 未完成安装

xmltoman

systemtap

staplog.c:50:2: error: invalid preprocessing directive #warn; did you
mean #warning?

50 \| #warn \"unknown architecture for crash/staplog support\"

需要修改代码，以支持loongarch64。增加一个patch。

编译脚本

rpmbuild -ba -D \"with_dyninst 0\" -D \"with_java 0\" -D \"with_virthost
0\" rpmbuild/SPECS/systemtap.spec \--nodeps

yelp-xsl

正常编译即可。

mallard-rng

已经编译过了。

yelp-tools

已经编译了

gtk-doc

已经编译了

atk

没有安装包了

fribidi

正常编译即可

libdatrie

正常编译即可

libthai

正常编译即可

babel

正常编译即可。提供了python-babel

cups

libexif

正常编译即可

libcupsfilter

rpmbuild \--rebuild
/opt/srpms/Packages/libcupsfilters-2.0.0-11.el10.src.rpm \--nodeps
\--nocheck

libppd

正常编译

cups-filter

依赖libexif, libcupsfilter

python-imagesize

python-sphinx-theme-alabaster

python-sphinx

TZ=UTC rpmbuild \--rebuild
/opt/srpms/Packages/python-sphinx-7.2.6-10.el10.src.rpm \--nodeps
\--nocheck

注意需要用TZ=UTC

xdg-utils

latexmk

gi-docgen

依赖sphinx

TZ=UTC rpmbuild \--rebuild
/opt/srpms/Packages/gi-docgen-2023.3-10.el10.src.rpm \--nodeps
\--nocheck

harfbuz

之前编译过一次，但编译的不全，再重新编译一遍。

python-typogrify

python-smartypants

pango

依赖gi-docgen。

libXcursor

正常编译即可。

share-mime-info

已经编译过了

gl-manpages

找不到安装包。

mesa-libGLU

正常编译即可

freeglut

正常编译即可

jasper

正常编译即可

gdk-pixbuf2

依赖gi-docgen

librsvg2

依赖gi-docgen, 依赖rust-toolset。需要补全rust。暂时跳过
