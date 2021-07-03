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
    PATH="/bin:/usr/bin:/sbin:/usr/sbin:/clang1-tools/bin:/clang1-tools/usr/bin" \
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
ln -sv /clang1-tools/usr/bin/{env,install,dd}           /usr/bin
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

# Apply some persistent environment variables.
cat > ~/.bash_profile << "EOF"
# Stage-1 Clang/LLVM environment.
TRUPLE="x86_64-pc-linux-musl"
CC="${TRUPLE}-clang"
CXX="${TRUPLE}-clang++"
AR="llvm-ar"
AS="llvm-as"
RANLIB="llvm-ranlib"
LD="ld.lld"
STRIP="llvm-strip"
COMMON_FLAGS="-march=native -Oz -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
export TRUPLE CC CXX AR AS RANLIB LD STRIP COMMON_FLAGS CFLAGS CXXFLAGS
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

> **Required!** First, before musl libc.
```bash

# Apply patch to fix "swab.h" under musl libc.
patch -Np1 -i \
../../extra/linux-headers/patches/include-uapi-linux-swab-Fix-potentially-missing-__always_inline.patch

# Make sure there are no stale files embedded in the package.
time { make LLVM=1 LLVM_IAS=1 HOSTCC="$CC" mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time {
    make LLVM=1 LLVM_IAS=1 HOSTCC="$CC" headers_check && \
    make LLVM=1 LLVM_IAS=1 HOSTCC="$CC" headers
}

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
cp -rv usr/include /usr/.
```

### `6` - Iana-Etc
> #### `20210611` or newer
> The Iana-Etc package provides data for network services and protocols.

> **Required!**
```bash
# Only need to copy these files into correct place.
cp -fv services protocols /etc/
```

### `7` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** Second, after Linux API Headers.
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

# Create a ldd symlink to use to print shared object dependencies.
ln -sv ../usr/lib/libc.so /bin/ldd

# Configure PATH for dynamic linker.
cat > /etc/ld-musl-x86_64.path << "EOF"
/lib
/usr/lib
/usr/local/lib
EOF

# Install fully-featured musl ldconfig.
# https://code.foxkit.us/smaeul/packages/-/commit/c631e4fc5ab64a9ad668ed5f753348ce8eae5219?view=parallel
install -vm755 -t /sbin/ ../../extra/musl/files/ldconfig

# Install `musl-legacy-compat` (from Void Linux).
for B in {cdefs,queue,tree}.h; do
    install -vm644 -t /usr/include/sys/ \
    ../../extra/musl/files/musl-legacy-compat/${B}
done; unset B; install -vm644 -t /usr/include/ \
../../extra/musl/files/musl-legacy-compat/error.h
```

### `8` - Adjusting Toolchains
> **Required!**
```bash
# Re-configure Stage-1 Clang/LLVM to use "/usr".
ln -sv clang-12 /clang1-tools/bin/x86_64-heiwa-linux-musl-clang
ln -sv clang-12 /clang1-tools/bin/x86_64-heiwa-linux-musl-clang++
cat > /clang1-tools/bin/x86_64-heiwa-linux-musl.cfg << "EOF"
--sysroot=/usr -Wl,-dynamic-linker /lib/ld-musl-x86_64.so.1
EOF

# Set the new toolchain configuration.
sed -i 's|CC=".*"|CC="x86_64-heiwa-linux-musl-clang"|'     ~/.bash_profile
sed -i 's|CXX=".*"|CXX="x86_64-heiwa-linux-musl-clang++"|' ~/.bash_profile
source ~/.bash_profile

# Symlink Clang to be used as GCC tools.
ln -sv x86_64-heiwa-linux-musl-clang   /clang1-tools/bin/gcc
ln -sv gcc                             /clang1-tools/bin/cc
ln -sv x86_64-heiwa-linux-musl-clang++ /clang1-tools/bin/g++

# Quick test.
echo "int main(){}" > dummy.c
"$CC" dummy.c -v -Wl,--verbose &> dummy.log
llvm-readelf -l a.out | grep ": /lib"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /lib/ld-musl-x86_64.so.1]

grep "ld.lld:.*crt[1in]" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /usr/lib/Scrt1.o
# |ld.lld: /usr/lib/crti.o
# |ld.lld: /usr/lib/crtn.o

grep -B1 "^ /usr/include" dummy.log

# | The output should be:
# |-----------------------
# |#include <...> search starts here:
# | /usr/include

grep -o -- -L/usr/lib dummy.log && \
grep -o -- -L/lib dummy.log

# | The output should be:
# |-----------------------
# |-L/usr/lib

# Clean up.
rm -fv dummy.c a.out dummy.log
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
    make CC="$CC" CFLAGS="$CFLAGS -DHAVE_STDINT_H=1" \
    TZDIR="/usr/share/zoneinfo" && \
    make -C posixtz-0.5 CC="$CC" posixtz
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

> **Required!** Before Toybox.
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

> **Required!** Before NetBSD libedit and Toybox.
```bash
# Build.
time { make CFLAGS="$CFLAGS -Wall -fPIC" all-dynamic; }

# Install.
time { make PREFIX=/usr install-dynamic; }
```

### `12` - NetBSD libedit
> #### `20210522-3.1` or newer
> The NetBSD libedit pacakage contains library providing line editing, history, and tokenisation functions.

> **Required!** After NetBSD Curses; before Toybox.
```bash
# Configure source.
CFLAGS="$CFLAGS -D__STDC_ISO_10646__" \
./configure --prefix=/usr --disable-static

# Build.
time { make V=1; }

# Install.
time { make install; }

# Create some symlinks and fake headers to replace "readline".
ln -sv libedit.so /usr/lib/libreadline.so
ln -sv libedit.pc /usr/lib/pkgconfig/readline.pc
mkdir -v /usr/include/readline
touch /usr/include/readline/history.h
touch /usr/include/readline/tilde.h
ln -sv ../editline/readline.h /usr/include/readline/readline.h
```

### `13` - Toybox (Bc, File, Grep, Inetutils, Psmisc, Sed)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!** After Zlib-ng, NetBSD Curses, and NetBSD libedit.
```bash
# Copy Toybox's .config file.
cp -v ../../extra/toybox/files/.config.bc_file_grep_inetutils_psmisc_sed.nlns .config

# Make sure to enable zlib.
grep -i "libz" .config

export CFFGPT="bc file egrep grep fgrep dnsdomainname ifconfig hostname ping telnet tftp
traceroute killall sed"

# Checks 14 commands, and make sure is enabled (=y).
# Pipe to " | wc -l" at the right of "done" to checks total of commands.
for X in ${CFFGPT}; do
    grep -v '#' .config | grep -i "_${X}=" || echo "* $X not CONFIGURED"
done

# Build.
time { make; }

# Checks compiled 14 commands.
./toybox | tr ' ' '\n'i | grep -xE $(echo $CFFGPT | tr ' ' '|'i) | wc -l

# Checks commands that not configured but compiled.
# `ping6` and `traceroute6` (inetutils)
./toybox | tr ' ' '\n'i | grep -vxE $(echo $CFFGPT | tr ' ' '|'i)

# So, totally is 16 commands.
./toybox | wc -w

# Install.
time { make PREFIX=/ install && unset CFFGPT; }
```

### `14` - Flex
> #### `2.6.4` or newer
> The Flex package contains a utility for generating programs that recognize patterns in text.

> **Required!** Before OpenBSD M4.
```bash
# Configure source.
HELP2MAN=/clang1-t/bin/true \
./configure \
    --prefix=/usr --disable-static \
    --docdir=/usr/share/doc/flex-2.6.4

# Build.
time { make; }

# Install.
time { make install; }

# A few programs do not know about `flex` yet and try to run its predecessor, `lex`.
# To support those programs, create a symbolic link named `lex` that runs `flex` in `lex` emulation mode.
ln -sv flex /usr/bin/lex
```

### `15` - OpenBSD M4
> #### `6.7` or newer
> The OpenBSD M4 package contains a macro processor.

> **Required!** After Flex.
```bash
# Configure source.
./configure --prefix=/usr --enable-m4

# Build. Fails when using multiple jobs.
time { make -j1; }

# Install.
time { make install; }
```

### `16` - Attr
> #### `2.5.1` or newer
> The Attr package contains utilities to administer the extended attributes on filesystem objects.

> **Required!** Before ACL.
```bash
# Configure source.
./configure \
    --prefix=/usr     \
    --disable-static  \
    --sysconfdir=/etc \
    --docdir=/usr/share/doc/attr-2.5.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `17` - ACL
> #### `2.3.1` or newer
> The ACL package contains utilities to administer Access Control Lists, which are used to define more fine-grained discretionary access rights for files and directories.

> **Required!** After Attr; before Shadow.
```bash
# Configure source.
./configure \
    --prefix=/usr         \
    --disable-static      \
    --docdir=/usr/share/doc/acl-2.3.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `18` - libcap
> #### `2.51` or newer
> The libcap package implements the user-space interfaces to the POSIX 1003.1e capabilities available in Linux kernels. These capabilities are a partitioning of the all powerful root privilege into a set of distinct privileges.

> **Required!** Before Shadow.
```bash
# Prevent static libraries from being installed.
sed -i '/install -m.*STA/d' libcap/Makefile

# Build.
time { make prefix=/usr lib=lib; }

# Install.
time { make prefix=/usr lib=lib install; }
```

### `19` - Shadow
> #### `4.8.1` or newer
> The Shadow package contains programs for handling passwords in a secure way.

> **Required!**
```bash
# Disable the installation of the groups program and its man pages, as Coreutils (replaced by Toybox) provides a better version.
# Also, prevent the installation of manual pages.
sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;

# Instead of using the default crypt method, use the more secure SHA-512 method of password encryption, which also allows passwords longer than 8 characters.
# It is also necessary to change the obsolete /var/spool/mail location for user mailboxes that Shadow uses by default to the /var/mail location used currently.
sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD SHA512:' \
    -e 's:/var/spool/mail:/var/mail:' -i etc/login.defs

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

### `20` - LLVM libunwind
> #### `12.0.0`
> C++ runtime stack unwinder from LLVM.

> **Required!**
```bash
```

<!--
    ### `` - Red Hat libcap-ng
    > #### `0.8.2` or newer
    > The Red Hat libcap-ng package implements the user-space interfaces to the POSIX 1003.1e capabilities available in Linux kernels. These capabilities are a partitioning of the all powerful root privilege into a set of distinct privileges. The library is intended to make programming with POSIX capabilities much easier than the traditional libcap library. It includes utilities that can analyse all currently running applications and print out any capabilities and whether or not it has an open ended bounding set.
    
    > **Required!** Before Shadow.
    ```bash
    # Apply patch to remove error codes.
    patch -Np1 -i ../../extra/libcap-ng/patches/apply-disable.patch
    
    # Configure source.
    ./configure \
        --prefix=/usr             \
        --sysconfdir=/etc         \
        --mandir=/usr/share/man   \
        --infodir=/usr/share/info \
        --without-python          \
        --without-python3         \
        --disable-static
    
    # Build.
    time { make; }
    
    # Install.
    time { make install; }
    ```

    ### `` - Bzip2
    > #### `1.0.8` or newer
    > The Bzip2 package contains programs for compressing and decompressing files. Compressing text files with bzip2 yields a much better compression percentage than with the traditional gzip.

    > **Required!** Before libexecinfo.
    ```bash
    # Apply patches to fix docs installation and library soname.
    patch -Np1 -i ../../extra/bzip2/patches/install_docs-1.patch
    patch -Np1 -i ../../extra/bzip2/patches/soname.patch

    # The following command ensures installation of symbolic links are relative, and ensure the man pages are installed into the correct location.
    sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
    sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile

    # Prepare.
    time { make -f Makefile-libbz2_so && make clean; }

    # Build.
    time { make; }

    # Install (also shared libraries) and fix the symlinks.
    time { make PREFIX=/usr install; }
    cp -av libbz2.so* /usr/lib/
    ln -sv libbz2.so.1.0 /usr/lib/libbz2.so
    rm -fv /usr/bin/{bunzip2,bzcat,bzip2}
    install -vm755 bzip2-shared /usr/bin/bzip2
    ln -sv bzip2 /usr/bin/bunzip2
    ln -sv bzip2 /usr/bin/bzcat

    # Remove useless static library.
    rm -fv /usr/lib/libbz2.a
    ```

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
