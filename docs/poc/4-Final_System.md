## `IV` Final System

> #### * Beginning of as root!
### `1` - Preparing Virtual Kernel File Systems
> Various file systems exported by the kernel are used to communicate to and from the kernel itself. These file systems are virtual in that no disk space is used for them. The content of the file systems resides in memory.
```bash
# When the kernel boots the system, it requires the presence of a few device nodes, in particular the console and null devices.
# The device nodes must be created on the hard disk so that they are available before the kernel populates "/dev", and additionally when Linux is started with init=/bin/bash.
# Create the directories and initial device nodes.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv ${HEIWA}/{dev,proc,sys,run,tmp} && \
    mknod -m 600 ${HEIWA}/dev/console c 5 1   && \
    mknod -m 666 ${HEIWA}/dev/null c 1 3
fi

# Mounting and Populating VKFS
# The recommended method of populating the "/dev" directory with devices is to mount a virtual filesystem (such as tmpfs) on the "/dev" directory, and allow the devices to be created dynamically on that virtual filesystem as they are detected or accessed.
# Device creation is generally done during the boot process by Udev.
# Since this new system does not yet have Udev and has not yet been booted, it is necessary to mount and populate "/dev" manually.
# This is accomplished by bind mounting the host system's "/dev" directory.
# A bind mount is a special type of mount that allows you to create a mirror of a directory or mount point to some other location.
# In some host systems, "/dev/shm" is a symbolic link to "/run/shm". The "/run" tmpfs was mounted above so in this case only a directory needs to be created.
if [[ -n "$HEIWA" ]]; then
    mount -Rv /dev        ${HEIWA}/dev     && \
    mount -Rv /dev/pts    ${HEIWA}/dev/pts && \
    mount -vt proc  proc  ${HEIWA}/proc    && \
    mount -vt sysfs sysfs ${HEIWA}/sys     && \
    mount -vt tmpfs tmpfs ${HEIWA}/run     && \
    mount -vt tmpfs tmpfs ${HEIWA}/tmp     && \
    if [[ -h "${HEIWA}/dev/shm" ]]; then
        mkdir -pv ${HEIWA}/$(readlink ${HEIWA}/dev/shm)
    fi
fi
```

### `2` - Entering the Chroot Environment
> Now that all the packages which are required to build the rest of the needed tools are on the system.
```bash
# Term variable is set to `xterm` for better compability, instead of "$TERM" that will broken if using `rxvt-unicode`.
if [[ -n "$HEIWA" ]]; then
    chroot "$HEIWA" /clang1-tools/usr/bin/env -i                                 \
    HOME="/root" TERM="xterm" PS1='(heiwa chroot) \u: \w \$ '                    \
    PATH="/usr/sbin:/usr/bin:/sbin:/bin::/clang1-tools/usr/bin:/clang1-tools/bin" \
    /clang1-tools/bin/bash --login +h
fi

# The `-i` option given to the env command will clear all variables of the chroot environment.
# Note! that the bash prompt will say "I have no name!". This is normal because the "/etc/passwd" file has not been created yet.
```
> #### * End of as root!

> #### * Beginning of as root in a chroot env!
### `3` - Creating Directories
> Its time to create the full structure file system.
```bash
mkdir -pv /{{,s}bin,boot,etc,home,lib/firmware,media,mnt,opt,root,var/tmp}

mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man,misc,terminfo,zoneinfo}
mkdir -pv /usr/{,local/}share/man/man{1..8}

mkdir -pv /usr/libexec

mkdir -pv /var/{opt,cache,lib/{color,misc,locate},local,log,mail,spool}

ln -sv /run /var/run
ln -sv /run/lock /var/lock

chmod -v 0700 /root
chmod -v 1777 /{var/,}tmp
```

### `4` - Creating Essential Files and Symlinks
```bash
# Some programs use hard-wired paths to programs which do not exist yet.
# In order to satisfy these programs, create a number of symbolic links which will be replaced by real files throughout the course of this chapter after the software has been installed.
ln -sv /clang1-tools/bin/{bash,cat,echo,ln,pwd,rm,stty} /bin
ln -sv /clang1-tools/bin/perl                           /usr/bin
ln -sv /clang1-tools/usr/bin/{env,file,install,dd}      /usr/bin
ln -sv /clang1-tools/lib/libc++{,abi}.so{,.1}           /usr/lib
ln -sv /clang1-tools/lib/libunwind.{a,so{,.1}}          /usr/lib
ln -sv bash                                             /bin/sh

# Historically, Linux maintains a list of the mounted file systems in the file "/etc/mtab".
# Modern kernels maintain this list internally and exposes it to the user via the "/proc" filesystem.
# To satisfy utilities that expect the presence of "/etc/mtab", create the following symbolic link.
ln -sv /proc/self/mounts /etc/mtab

# In order for user root to be able to login and for the name `root` to be recognized, there must be relevant entries in the "/etc/passwd" and "/etc/group" files.
# Create the "/etc/passwd" file by running the following command.
# User `uucp` is required by OpenRC for later installation.
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
uucp:x:10:17:uucp:/var/spool/uucp:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF

# The actual password for root will be set later.
# Create the "/etc/group" file by running the following command.
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
uucp:x:17:uucp
messagebus:x:18:
input:x:24:
mail:x:34:
kvm:x:61:
wheel:x:97:
nogroup:x:99:
users:x:999:
EOF

# To remove the "I have no name!" prompt, start a new shell.
# Since the "/etc/passwd" and "/etc/group" files have been created, user name and group name resolution will now work.
exec /clang1-tools/bin/bash --login +h

# The login, agetty, and init programs (and others) use a number of log files to record information such as who was logged into the system and when.
# However, these programs will not write to the log files if they do not already exist.
# Initialize the log files and give them proper permissions.
touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp

# Apply toolchain persistent environment variables, set compiler to the Stage-1 Clang default triplet (pc).
cat > ~/.bash_profile << "EOF"
# Stage-1 Clang/LLVM environment.
CC="x86_64-pc-linux-musl-clang"
CXX="x86_64-pc-linux-musl-clang++"
AR="llvm-ar"
AS="llvm-as"
RANLIB="llvm-ranlib"
LD="ld.lld"
NM="llvm-nm"
OBJCOPY="llvm-objcopy"
OBJDUMP="llvm-objdump"
READELF="llvm-readelf"
SIZE="llvm-size"
STRIP="llvm-strip"
COMMON_FLAGS="-march=native -Oz -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
LLVM_SRC="/sources/llvm"
export CC CXX AR AS RANLIB LD NM OBJCOPY OBJDUMP READELF SIZE STRIP COMMON_FLAGS CFLAGS CXXFLAGS LLVM_SRC
# Make's multiple jobs based on CPU core/threads.
alias make="make -j$(nproc) -l$(nproc)"
EOF
source ~/.bash_profile
```

<br>

> #### Compilation Instruction!
> ```bash
> (heiwa chroot) root: /sources/pkgs # tar xf target-package.tar.xz
> (heiwa chroot) root: /sources/pkgs # tar xzf target-package.tar.gz
> (heiwa chroot) root: /sources/pkgs # pushd target-package
> 
> < compilation process >
> 
> (heiwa chroot) root: /sources/pkgs/target-package # popd
> (heiwa chroot) root: /sources/pkgs # rm -rf target-package
> ```

### `5` - Linux API Headers
> #### Xanmod-CacULE, `5.12.x` or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
```bash

# Apply patch to fix "swab.h" under musl libc.
patch -Np1 -i \
../../extra/linux-headers/patches/include-uapi-linux-swab-Fix-potentially-missing-__always_inline.patch

# Make sure there are no stale files embedded in the package.
time { make LLVM=1 LLVM_IAS=1 HOSTCC={$CC} mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time {
    make LLVM=1 LLVM_IAS=1 HOSTCC=${CC} headers_check && \
    make LLVM=1 LLVM_IAS=1 HOSTCC=${CC} headers
}

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
cp -rfv usr/include /usr/.
```

### `6` - Iana-Etc
> #### `20210611` or newer
> The Iana-Etc package provides data for network services and protocols.

> **Required!** Before `Perl`.
```bash
# Only need to copy these files into correct place.
cp -fv services protocols /etc/
```

### `7` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Apply patches (from Void Linux and Alpine Linux).
for P in {epoll_cp,isascii,mo_lookup,handle-aux-at_base}.patch; do
    patch -Np1 -i ../../extra/musl/patches/${P}
done; unset P

# Configure source.
LDFLAGS="-Wl,-soname,libc.musl-x86_64.so.1" \
./configure \
    --prefix=/usr         \
    --sysconfdir=/etc     \
    --localstatedir=/var  \
    --disable-gcc-wrapper \
    --enable-optimize=speed

# Build.
time { make; }

# Install.
time { make install; }

# Create a `ldd` symlink to use to print shared object dependencies.
ln -sv ../usr/lib/libc.so /bin/ldd

# Configure PATH for dynamic linker.
cat > /etc/ld-musl-x86_64.path << "EOF"
/lib
/usr/lib
/usr/local/lib
EOF

# Install fully-featured musl ldconfig.
# [ https://code.foxkit.us/smaeul/packages/-/commit/c631e4fc5ab64a9ad668ed5f753348ce8eae5219?view=parallel ]
install -vm755 -t /sbin/ ../../extra/musl/files/ldconfig

# Install `musl-legacy-compat` (from Void Linux).
for B in {cdefs,queue,tree}.h; do
    install -vm644 -t /usr/include/sys/ \
    ../../extra/musl/files/musl-legacy-compat/${B}
done; unset B; install -vm644 -t /usr/include/ \
../../extra/musl/files/musl-legacy-compat/error.h
```

### `8` - Adjusting Toolchain
> **Required!**
```bash
# Configure Stage-1 Clang with new triplet to produce binaries with "lib/ld-musl-x86_64.so.1" and libraries from "/usr/*".
ln -sv clang   /clang1-tools/bin/x86_64-heiwa-linux-musl-clang
ln -sv clang++ /clang1-tools/bin/x86_64-heiwa-linux-musl-clang++
cat > /clang1-tools/bin/x86_64-heiwa-linux-musl.cfg << "EOF"
--sysroot=/usr -Wl,-dynamic-linker /lib/ld-musl-x86_64.so.1
EOF

# Set toolchain to the new triplet from Stage-1 Clang/LLVM.
sed -i 's|CC=.*|CC="x86_64-heiwa-linux-musl-clang"|'     ~/.bash_profile
sed -i 's|CXX=.*|CXX="x86_64-heiwa-linux-musl-clang++"|' ~/.bash_profile
source ~/.bash_profile

# Quick test for the new triplet of Stage-1 Clang/LLVM.
echo "int main(){}" > dummy.c
${CC} dummy.c -v -Wl,--verbose &> dummy.log
${READELF} -l a.out | grep ": /lib"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /lib/ld-musl-x86_64.so.1]

grep "ld.lld:.*crt[1in]" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /usr/lib/Scrt1.o
# |ld.lld: /usr/lib/crti.o
# |ld.lld: /usr/lib/crtn.o

# Check the headers path.
grep -B1 "^ /usr/include" dummy.log

# | The output should be:
# |-----------------------
# |#include <...> search starts here:
# | /usr/include

# Check the dynamic linker libraries path.
grep -o -- -L/usr/lib dummy.log && grep -o -- -L/lib dummy.log

# | The output should be:
# |-----------------------
# |-L/usr/lib

# Clean up.
rm -fv dummy.{c,log} a.out
```

### `9` - TimeZone Database
> #### `2021a` and `0.5` for posixtz
> The TZDb package contains code and data that represent the history of local time for many representative locations around the globe. It is updated periodically to reflect changes made by political bodies to time zone boundaries, UTC offsets, and daylight-saving rules.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!** Since using musl libc.
```bash
# Create a directory and decompress needed tarball.
mkdir -v tzdata && pushd tzdata   && \
    tar xzf ../tzdata2021a.tar.gz && \
    tar xzf ../tzcode2021a.tar.gz && \
    tar xf  ../posixtz-0.5.tar.xz

# Apply patches (from Alpine Linux).
patch -Np1 -i ../../extra/tzdata/patches/0001-posixtz-ensure-the-file-offset-we-pass-to-lseek-is-o.patch
patch -Np1 -i ../../extra/tzdata/patches/0002-fix-implicit-declaration-warnings-by-including-strin.patch

export timezones="africa antarctica asia australasia europe northamerica
southamerica etcetera backward factory"

# Build. Ignore "pkg-config: No such file or directory" while building `posixtz`.
time {
    make CC=${CC} CFLAGS="$CFLAGS -DHAVE_STDINT_H=1" \
    TZDIR="/usr/share/zoneinfo" && \
    make -C posixtz-0.5 CC=${CC} posixtz
}

# Install.
install -vm755 -t /usr/bin/ tzselect posixtz-0.5/posixtz zic zdump
install -vm644 -t /usr/share/man/man8/ zic.8 zdump.8

# Install zoneinfo. Ignore all warnings.
install -vm444 -t /usr/share/zoneinfo/ iso3166.tab zone1970.tab zone.tab
zic -y ./yearistype -d /usr/share/zoneinfo ${timezones}
zic -y ./yearistype -d /usr/share/zoneinfo/right -L leapseconds ${timezones}
zic -y ./yearistype -d /usr/share/zoneinfo -p America/New_York

# Configure timezone.
# Use `tzselect` to determine <xxx>.
tzselect

cp -v /usr/share/zoneinfo/<xxx> /etc/localtime

# Back to "/sources/pkgs" directory.
popd && rm -rf tzdata; unset timezones
```

### `10` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** Before `Kmod`, `Perl`, and `Util-linux`.
```bash
# Apply patch (from OpenMandriva) to fix `Z_SOLO` while building perl.
patch -Np1 -i ../../extra/zlib-ng/patches/0001-Fix-Z_SOLO-mode.patch

# Configure source.
cmake -B build \
     -DCMAKE_BUILD_TYPE=Release     \
     -DCMAKE_INSTALL_PREFIX="/usr"  \
     -DWITH_NATIVE_INSTRUCTIONS=YES \
     -DWITH_SANITIZERS=ON           \
     -DZLIB_COMPAT=ON

# Build.
time { make -C build; }

# Install.
time { make -C build install; }

# Remove useless static library.
rm -fv /usr/lib/libz.a
```

### `11` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** Before `GNU Bash`, `NetBSD libedit`, `GNU Texinfo`, and `Util-linux`.
```bash
# Build.
time { make CFLAGS="$CFLAGS -fPIC"; }

# Install and create symlinks as `libtinfo` libraries.
time {
    make PREFIX=/usr install                  && \
    ln -sv libterminfo.a  /usr/lib/libtinfo.a && \
    ln -sv libterminfo.so /usr/lib/libtinfo.so
}

# Remove man colission with `attr`.
rm -fv /usr/share/man/man3/attr_{g,s}et.3
```

### `12` - NetBSD libedit
> #### `20210522-3.1` or newer
> The NetBSD libedit pacakage contains library providing line editing, history, and tokenisation functions.

> **Required!** Before `GNU Bash` and `GNU AWK`.
```bash
# Configure source.
CFLAGS="$CFLAGS -D__STDC_ISO_10646__" \
./configure --prefix=/usr

# Build.
time { make; }

# Install.
time { make install; }

# Create symlinks and fake headers to replace "readline".
for L in history readline; do
    ln -sv libedit.a  /usr/lib/lib${L}.a
    ln -sv libedit.so /usr/lib/lib${L}.so
done; unset L
ln -sv libedit.pc /usr/lib/pkgconfig/readline.pc
mkdir -v /usr/include/readline
ln -sv ../editline/readline.h /usr/include/readline/readline.h
```

### `13` - Flex
> #### `2.6.4` or newer
> The Flex package contains a utility for generating programs that recognize patterns in text.

> **Required!** Before `OpenBSD M4`, `IProute2`, `Kbd`, and `Kmod`.
```bash
# Configure source. Flex still expect `gcc` to configure.
ln -sv ${CC} /clang1-tools/bin/gcc       && \
ac_cv_func_malloc_0_nonnull=yes             \
ac_cv_func_realloc_0_nonnull=yes            \
HELP2MAN=/clang1-tools/bin/true ./configure \
    --prefix=/usr --disable-static          \
    --docdir=/usr/share/doc/flex-2.6.4   && \
unlink /clang1-tools/bin/gcc

# Build.
time { make; }

# Install and create symlink as `lex`.
time {
    make install && \
    ln -sv flex /usr/bin/lex
}

# A few programs do not know about `flex` yet and try to run its predecessor, `lex`.
# To support those programs, create a symbolic link named `lex` that runs `flex` in `lex` emulation mode.
```

### `14` - OpenBSD M4
> #### `6.7` or newer
> The OpenBSD M4 package contains a macro processor.

> **Required!** Before `GNU Autoconf`.
```bash
# Configure source.
./configure --prefix=/usr --enable-m4

# Build. Fails when using multiple jobs.
time { make -j1; }

# Install.
time { make install; }
```

### `15` - Attr
> #### `2.5.1` or newer
> The Attr package contains utilities to administer the extended attributes on filesystem objects.

> **Required!** Before `ACL` and `libcap`.
```bash
# Configure source.
./configure \
    --prefix=/usr     \
    --bindir=/bin     \
    --disable-static  \
    --sysconfdir=/etc \
    --docdir=/usr/share/doc/attr-2.5.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `16` - ACL
> #### `2.3.1` or newer
> The ACL package contains utilities to administer Access Control Lists, which are used to define more fine-grained discretionary access rights for files and directories.

> **Required!**
```bash
# Configure source.
./configure \
    --prefix=/usr         \
    --bindir=/bin         \
    --disable-static      \
    --docdir=/usr/share/doc/acl-2.3.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `17` - libcap
> #### `2.51` or newer
> The libcap package implements the user-space interfaces to the POSIX 1003.1e capabilities available in Linux kernels. These capabilities are a partitioning of the all powerful root privilege into a set of distinct privileges.

> **Required!** Before `IPRoute2` and `Shadow`.
```bash
# Prevent static libraries from being installed.
sed -i '/install -m.*STA/d' libcap/Makefile

# Build.
time { make CC=${CC} SBINDIR=/sbin prefix=/usr lib=lib; }

# Install.
time { make CC=${CC} SBINDIR=/sbin prefix=/usr lib=lib install; }
```

### `18` - Shadow
> #### `4.8.1` or newer
> The Shadow package contains programs for handling passwords in a secure way.

> **Required!** Currently, without cracklib.
```bash
# Disable the installation of the groups program and its man pages, as Coreutils (replaced by Toybox) provides a better version.
# Also, prevent the installation of manual pages.
sed -i 's|groups$(EXEEXT) ||' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;

# Instead of using the default crypt method, use the more secure SHA-512 method of password encryption, which also allows passwords longer than 8 characters.
# It is also necessary to change the obsolete "/var/spool/mail" location for user mailboxes that Shadow uses by default to the /var/mail location used currently.
sed -e 's|#ENCRYPT_METHOD DES|ENCRYPT_METHOD SHA512|' \
    -e 's|/var/spool/mail|/var/mail|' -i etc/login.defs

# Make a minor change to make the first group number generated by useradd 1000.
sed -i 's|1000|999|' etc/useradd

# Configure source.
touch /usr/bin/passwd && \
./configure \
    --sysconfdir=/etc \
    --with-group-name-max-length=32

# Build.
time { make; }

# Install.
time { make install; }

# This package contains utilities to add, modify, and delete users and groups; set and change their passwords; and perform other administrative tasks.
# For a full explanation of what password shadowing means, see the doc/HOWTO file within the unpacked source tree.
# If using Shadow support, keep in mind that programs which need to verify passwords (display managers, FTP programs, pop3 daemons, etc.) must be Shadow-compliant.
# That is, they need to be able to work with shadowed passwords.
# Enable shadowed passwords and groups.
pwconv; grpconv

# CREATE_MAIL_SPOOL=yes
# This parameter causes useradd to create a mailbox file for the newly created user.
# useradd will make the group ownership of this file to the mail group with 0660 permissions.
# If you would prefer that these mailbox files are not created by useradd, issue the following command.
sed -i 's|yes|no|' /etc/default/useradd

# Set the root password.
passwd root
```

### `19` - LLVM libunwind
> #### `12.0.0`
> C++ runtime stack unwinder from LLVM.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!** Before `LLVM libcxxabi`.
```bash
# Decompress the LLVM source, and rename.
tar xf llvm-12.0.0.src.tar.xz && mv -v llvm-12.0.0.src "$LLVM_SRC"

# Decompress `libunwind` to the correct directories. Requires `libcxx`.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/libunwind-12.0.0.src.tar.xz && mv -v libunwind-12.0.0.src libunwind
    tar xf ../../pkgs/libcxx-12.0.0.src.tar.xz    && mv -v libcxx-12.0.0.src libcxx
popd

# Configure source.
pushd ${LLVM_SRC}/projects/libunwind/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"  \
        -DCMAKE_C_FLAGS="-fPIC"        \
        -DCMAKE_CXX_FLAGS="-fPIC"      \
        -DLIBUNWIND_ENABLE_SHARED=ON   \
        -DLIBUNWIND_USE_COMPILER_RT=ON \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install and remove the `libunwind` source.
time {
    make -C build install && popd && \
    rm -rf ${LLVM_SRC}/projects/libunwind
}
```

### `20` - LLVM libcxxabi
> #### `12.0.0`
> Low level support for a standard C++ library from LLVM.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!** Before `LLVM libcxx`.
```bash
# Decompress `libcxxabi` to the correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/libcxxabi-12.0.0.src.tar.xz && mv -v libcxxabi-12.0.0.src libcxxabi
popd

# Configure source.
pushd ${LLVM_SRC}/projects/libcxxabi/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"                                     \
        -DLIBCXXABI_ENABLE_STATIC=ON                                      \
        -DLIBCXXABI_USE_COMPILER_RT=ON                                    \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON                                  \
        -DLIBCXXABI_LIBUNWIND_PATH="/usr/lib"                             \
        -DLIBCXXABI_LIBCXX_INCLUDES="${LLVM_SRC}/projects/libcxx/include" \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install. But don't remove the `libcxxabi` source, `libcxx` requires it.
time {
    make -C build install            && \
    cp -v include/*.h /usr/include/. && popd
}
```

### `21` - LLVM libcxx
> #### `12.0.0`
> New implementation of the C++ standard library, targeting C++11 from LLVM.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!** Before `Clang/LLVM`.
```bash
# Configure source.
pushd ${LLVM_SRC}/projects/libcxx/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"                 \
        -DLIBCXX_ENABLE_SHARED=ON                     \
        -DLIBCXX_ENABLE_STATIC=ON                     \
        -DLIBCXX_HAS_MUSL_LIBC=ON                     \
        -DLIBCXX_USE_COMPILER_RT=ON                   \
        -DLIBCXX_INSTALL_HEADERS=ON                   \
        -DLIBCXX_CXX_ABI=libcxxabi                    \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/usr/include" \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/usr/lib"      \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON              \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install and remove the `libcxx` also `libcxxabi` source.
time {
    make -C build install && popd && \
    rm -rf ${LLVM_SRC}/projects/libcxx{,abi}
}
```

### `22` - libexecinfo
> #### `1.1` or newer
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** Before `Clang/LLVM`.
```bash
# Apply patches (from Alpine Linux).
for P in {10-execinfo,20-define-gnu-source,30-linux-makefile}.patch; do
    patch -Np1 -i ../../extra/libexecinfo/patches/${P}
done; unset P

# Build.
time { make CC=${CC} AR=${AR} CFLAGS="$CFLAGS -fno-omit-frame-pointer"; }

# Install.
ln -sv libexecinfo.so{.1,}
install -vm755 -t /clang1-tools/include/ {execinfo,stacktraverse}.h
install -vm755 -t /clang1-tools/lib/ libexecinfo.{a,so{.1,}}
```

### `23` - Clang/LLVM
> #### `12.0.0`
> C language family frontend for LLVM.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!**
```bash
# Enter to the LLVM source directory.
cd "$LLVM_SRC"

# Decompress `clang`, `lld`, and `compiler-rt` to the correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/compiler-rt-12.0.0.src.tar.xz && mv -v compiler-rt-12.0.0.src compiler-rt
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.0.src.tar.xz && mv -v clang-12.0.0.src clang
    tar xf ../../pkgs/lld-12.0.0.src.tar.xz   && mv -v lld-12.0.0.src lld
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/stage1-appatch

# Disable sanitizers for musl, fixing "early build failure".
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Fix missing header for lld, [ https://bugs.llvm.org/show_bug.cgi?id=49228 ].
tar xf ../pkgs/libunwind-12.0.0.src.tar.xz && \
mkdir -pv tools/lld/include/mach-o         && \
cp -fv libunwind-12.0.0.src/include/mach-o/compact_unwind_encoding.h \
tools/lld/include/mach-o/. && rm -rf libunwind-12.0.0.src

# Update host/target triplet detection.
cp -fv ../extra/llvm/files/config.guess cmake/.

# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release                                  \
    -DCMAKE_INSTALL_PREFIX="/usr"                               \
    -DCMAKE_INSTALL_OLDINCLUDEDIR="/usr/include"                \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"         \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-musl"                   \
    -DLLVM_TARGETS_TO_BUILD="host;BPF;AMDGPU;X86"               \
    -DLLVM_TARGET_ARCH="X86"                                    \
    -DLLVM_LINK_LLVM_DYLIB=ON                                   \
    -DLLVM_BUILD_LLVM_DYLIB=ON                                  \
    -DLLVM_BUILD_TESTS=OFF                                      \
    -DLLVM_ENABLE_LIBEDIT=OFF                                   \
    -DLLVM_ENABLE_LIBXML2=OFF                                   \
    -DLLVM_ENABLE_LIBCXX=ON                                     \
    -DLLVM_ENABLE_LLD=ON                                        \
    -DLLVM_ENABLE_RTTI=ON                                       \
    -DLLVM_ENABLE_ZLIB=ON                                       \
    -DLLVM_INCLUDE_GO_TESTS=OFF                                 \
    -DLLVM_INCLUDE_TESTS=OFF                                    \
    -DLLVM_INCLUDE_DOCS=OFF                                     \
    -DLLVM_INCLUDE_EXAMPLES=OFF                                 \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                               \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF                          \
    -DCOMPILER_RT_BUILD_XRAY=OFF                                \
    -DCOMPILER_RT_BUILD_PROFILE=OFF                             \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                           \
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON                       \
    -DCOMPILER_RT_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"  \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                           \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                         \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                           \
    -DCLANG_DEFAULT_LINKER="/usr/bin/ld.lld"                    \
    -DDEFAULT_SYSROOT="/usr"                                    \
    -DBacktrace_INCLUDE_DIR="/usr/include"                      \
    -DBacktrace_LIBRARY="/usr/lib/libexecinfo.so"               \
    -DICONV_LIBRARY_PATH="/usr/lib/libc.so"                     \
    -DLLVM_INSTALL_BINUTILS_SYMLINKS=ON                         \
    -DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON                          \
    -DLLVM_INSTALL_UTILS=ON                                     \
    -DLLVM_ENABLE_BINDINGS=OFF                                  \
    -DLLVM_ENABLE_IDE=OFF                                       \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF

# Build.
time { make -C build; }

# Install and remove the build directory.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/usr" -P cmake_install.cmake && \
    popd && rm -rf build
}

# Create a symlink required by the FHS for "historical" reasons, and
# set `clang`, `clang++`, and `lld` as default toolchain compiler and linker.
ln -sv ../usr/bin/clang          /lib/cpp
ln -sv clang                     /usr/bin/cc
ln -sv lld                       /usr/bin/ld
sed -i 's|CC=.*|CC="clang"|'     ~/.bash_profile
sed -i 's|CXX=.*|CXX="clang++"|' ~/.bash_profile
source ~/.bash_profile

# Build useful utilities for BSD-compability.
time {
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/getconf.c -o getconf && \
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/getent.c -o getent   && \
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/iconv.c -o iconv
}

# Install above utilities, and the man files (from NetBSD).
install -vm755 -t /usr/bin/ get{conf,ent} iconv
install -vm644 -t /usr/share/man/man1/ ../extra/musl/files/musl-utils/get{conf,ent}.1

# Quick test.
echo "int main(){}" > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ": /lib"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /lib/ld-musl-x86_64.so.1]

grep "ld.lld:.*crt[1in].o" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /usr/bin/../lib/Scrt1.o
# |ld.lld: /usr/bin/../lib/crti.o
# |ld.lld: /usr/bin/../lib/crtn.o

# Check the headers path.
grep -B1 "^ /usr/include" dummy.log

# | The output should be:
# |-----------------------
# |#include <...> search starts here:
# | /usr/include

# Check the dynamic linker libraries path.
grep -o -- -L/usr/lib dummy.log

# | The output should be:
# |-----------------------
# |-L/usr/lib

# Back to "/sources/pkgs" directory.
cd /sources/pkgs/
```

### `24` - Bzip2
> #### `1.0.8` or newer
> The Bzip2 package contains programs for compressing and decompressing files. Compressing text files with bzip2 yields a much better compression percentage than with the traditional gzip.

> **Required!** Before `Perl`.
```bash
# Apply patches to fix docs installation and library soname.
patch -Np1 -i ../../extra/bzip2/patches/install_docs-1.patch
patch -Np1 -i ../../extra/bzip2/patches/soname.patch

# Fix the makefile to ensures installation of symlinks are relative and the man pages are installed into correct location.
sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile

# Prepare.
time { make CFLAGS="$CFLAGS -fPIC" -f Makefile-libbz2_so && make clean; }

# Build.
time { make CFLAGS="$CFLAGS -fPIC"; }

# Install (also shared libraries) and fix the symlinks.
time { make PREFIX=/usr install; }
ln -sv libbz2.so.1.0 libbz2.so
install -vm755 -t /usr/lib/ libbz2.so*
cp -fv bzip2-shared bzip2; ln -sv bzip2 bunzip2; ln -sv bzip2 bzcat
install -vm755 -t /usr/bin/ b{un,}zip2 bzcat

# Remove useless static library.
rm -fv /usr/lib/libbz2.a
```

### `25` - Perl
> #### `5.32.1`
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** Before `OpenSSL`.
```bash
# Ensure to build with Perl with the libraries installed on the system.
BUILD_ZLIB=0 BUILD_BZIP2=0
export BUILD_ZLIB BUILD_BZIP2
 
# Apply patches (from Alpine Linux) to fix "locale.c" error in programs like `rxvt-unicode`, and change stack size.
patch -Np1 -i ../../extra/perl/patches/musl-locale.patch
patch -Np1 -i ../../extra/perl/patches/musl-stack-size.patch

# Configure source.
./Configure -des \
    -Dprefix=/usr -Dvendorprefix=/usr -Dccdlflags="-rdynamic"    \
    -Dcccdlflags="-fPIC" -Dcccdlflags="-fPIC" -Duselargefiles    \
    -Dprivlib=/usr/lib/perl5/5.32.1/core_perl                    \
    -Darchlib=/usr/lib/perl5/5.32.1/core_perl                    \
    -Dsitelib=/usr/lib/perl5/5.32.1/site_perl                    \
    -Dsitearch=/usr/lib/perl5/5.32.1/site_perl                   \
    -Dvendorlib=/usr/lib/perl5/5.32.1/vendor_perl                \
    -Dvendorarch=/usr/lib/perl5/5.32.1/vendor_perl               \
    -Doptimize="$CFLAGS -DNO_POSIX_2008_LOCALE -D_GNU_SOURCE"    \
    -Dcf_by="Heiwa/Linux" -Dmyuname="heiwa" -Dmyhostname="heiwa" \
    -Dusethreads -Duseshrplib -Dman1ext=1 -Dman3ext=3pm          \
    -Dman1dir=/usr/share/man/man1 -Dman3dir=/usr/share/man/man3

# Build.
time { make; }

# Install and unset Perl specifics exported variables.
time { make install && unset BUILD_ZLIB BUILD_BZIP2; }
```

### `26` - OpenSSL
> #### `1.1.1k` or newer
> The OpenSSL package contains management tools and libraries relating to cryptography. These are useful for providing cryptographic functions to other packages, such as OpenSSH, email applications, and web browsers (for accessing HTTPS sites).

> **Required!** Before `Toybox`.
```bash
# Configure source.
./Configure linux-x86_64       \
    --prefix=/usr --libdir=lib \
    --openssldir=/etc/ssl      \
    shared no-ssl3-method      \
    enable-ec_nistp_64_gcc_128 \
    ${CFLAGS} -Wa,--noexecstack

# Build.
time { make; }

# Install.
time { make MANSUFFIX=ssl install; }

# Add the version to the documentation directory name, to be consistent with other packages.
mv -fv /usr/share/doc/openssl /usr/share/doc/openssl-1.1.1k
```

### `27` - Toybox (Bc, Coreutils, File, Findutils, Grep, Inetutils, Man, Procps-ng, Psmisc, Sed, Sysklogd, Tar)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!**
```bash
# Copy Toybox's .config file.
cp -v ../../extra/toybox/files/.config.bc_coreutils_file_findutils_grep_inetutils_man_procps_psmisc_sed_sysklogd_tar .config

# Make sure to enable `libcrypto` and `libz`.
grep -iE "libcrypto|libz" .config

export CFFGPT="bc base64 base32 basename cat chgrp chmod chown chroot cksum comm cp cut
date dd df dirname du echo env expand expr factor false fmt fold groups head hostid id
install link ln logname ls md5sum mkdir mkfifo mknod mktemp mv nice nl nohup nproc od
paste printenv printf pwd readlink realpath rm rmdir seq sha1sum sha224sum sha256sum
sha384sum sha512sum shred sleep sort split stat stty sync tac tail tee test timeout
touch tr true truncate tty uname uniq unlink wc who whoami yes file find xargs egrep
grep fgrep dnsdomainname ifconfig hostname ping telnet tftp traceroute man free pgrep
pidof pkill pmap ps pwdx sysctl top uptime vmstat w watch killall sed klogd tar"

# Checks 115 commands, and make sure is enabled (=y).
# Pipe to " | wc -l" at the right of "done" to checks total of commands.
for X in ${CFFGPT}; do
    grep -v '#' .config | grep -i --color=auto "_${X}=" || echo "* $X not CONFIGURED"
done

# Build.
time { make; }

# Checks compiled 115 commands.
./toybox | tr ' ' '\n'i | grep -xE --color=auto $(echo $CFFGPT | tr ' ' '|'i) | wc -l

# Checks commands that not configured but compiled.
# `[` (coreutils)
# `ping6` and `traceroute6` (inetutils)
./toybox | tr ' ' '\n'i | grep -vxE --color=auto $(echo $CFFGPT | tr ' ' '|'i)

# So, totally is 118 commands.
./toybox | wc -w

# Install.
time { make PREFIX=/ install && unset CFFGPT; }
```

### `28` - GNU Autoconf
> #### `2.71` or newer
> The GNU Autoconf package contains programs for producing shell scripts that can automatically configure source code.

> **Required!** Before `GNU Automake`.
```bash
# Configure source.
./configure --prefix=/usr

# Build.
time { make; }

# Install.
time { make install; }
```

### `29` - GNU Automake
> #### `1.16.3` or newer
> The GNU Automake package contains programs for generating Makefiles for use with Autoconf.

> **Required!** Before `Argp-standalone`.
```bash
# Configure source.
./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.16.3

# Build.
time { make; }

# Install.
time { make install; }
```

<!--
    ### `` - Argp-standalone
    > #### `1.4.1` or newer
    > The Argp-standalone package contains hierarchial argument parsing library broken out from glibc.

    > **Required!** Since using musl libc.
    ```bash
    # Apply patch to enable `-fgnu89-inline` compile flag (from Adélie Linux).
    patch -Np1 -i ../../extra/argp-standalone/patches/gnu89-inline.patch

    # Configure source.
    CFLAGS="$CFLAGS -fPIC" \
    ./configure \
        --prefix=/usr      \
        --disable-static   \
        --sysconfdir=/etc  \
        --localstatedir=/var

    # Build.
    time { make; }

    # Install.
    install -vm644 -t /usr/lib/  libargp.a
    install -vm644 -t /usr/include/ argp.h
    ```
-->
