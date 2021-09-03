## `IV` Final System

> #### * Beginning of as root!
### `1` - Preparing Virtual Kernel File Systems
> Various file systems exported by the kernel are used to communicate to and from the kernel itself. These file systems are virtual in that no disk space is used for them. The content of the file systems resides in memory.

> #### Creating Initial Device Nodes

> When the kernel boots the system, it requires the presence of a few device nodes, in particular the console and null devices. The device nodes must be created on the hard disk so that they are available before the kernel populates "/dev", and additionally when Linux is started with init=/bin/bash. Read more about [Linux device nodes](https://kernel.org/doc/Documentation/admin-guide/devices.txt).
```bash
# Create directories for the next step (Populating VKFS) and initial device nodes.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv ${HEIWA}/{dev,proc,sys,run,tmp} && \
    mknod -m 600 ${HEIWA}/dev/console c 5 1   && \
    mknod -m 666 ${HEIWA}/dev/null c 1 3
fi
```
> #### Mounting and Populating VKFS

> The recommended method of populating the "/dev" directory with devices is to mount a virtual filesystem (such as tmpfs) on the "/dev" directory, and allow the devices to be created dynamically on that virtual filesystem as they are detected or accessed. Device creation is generally done during the boot process by Udev.
> 
> Since this new system does not yet have Udev and has not yet been booted, it is necessary to mount and populate "/dev" manually. This is accomplished by bind mounting the host system's "/dev" directory. A bind mount is a special type of mount that allows you to create a mirror of a directory or mount point to some other location.
> 
> In some host systems, "/dev/shm" is a symbolic link to "/run/shm". The "/run" tmpfs was mounted above so in this case only a directory needs to be created.
```bash
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
```bash
# Term variable is set to `xterm` for better compability, instead of "$TERM" that will broken if using `rxvt-unicode`.
if [[ -n "$HEIWA" ]]; then
    chroot "$HEIWA" /clang1-tools/usr/bin/env -i                                 \
    HOME="/root" TERM="xterm" PS1='(heiwa chroot) \u: \w \$ '                    \
    PATH="/usr/sbin:/usr/bin:/sbin:/bin:/clang1-tools/usr/bin:/clang1-tools/bin" \
    /clang1-tools/bin/bash --login +h
fi

# The `-i` option given to the env command will clear all variables of the chroot environment.
# Note! that the bash prompt will say "I have no name!". This is normal because the "/etc/passwd" file has not been created yet.
```
> #### * End of as root!

> #### * Beginning of as root in a chroot env!
### `3` - Creating Directories
> It's time to create the full-structured filesystem. See [FHS version 3.0](https://refspecs.linuxfoundation.org/fhs.shtml).
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
ln -sv /clang1-tools/bin/{bash,cat,echo,ln,pwd,rm,stty} /bin/
ln -sv /clang1-tools/bin/perl                           /usr/bin/
ln -sv /clang1-tools/usr/bin/{env,file,install,dd}      /usr/bin/
ln -sv /clang1-tools/lib/libc++{,abi}.so{,.1}           /usr/lib/
ln -sv /clang1-tools/lib/libunwind.so{,.1}              /usr/lib/
ln -sv bash                                             /bin/sh

# Historically, Linux maintains a list of the mounted file systems in the file "/etc/mtab".
# Modern kernels maintain this list internally and exposes it to the user via the "/proc" filesystem.
# To satisfy utilities that expect the presence of "/etc/mtab", create the following symbolic link.
ln -sv /proc/self/mounts /etc/mtab
```
```bash
# In order for user root to be able to login and for the name `root` to be recognized, there must be relevant entries in the "/etc/passwd" and "/etc/group" files.
# Create the "/etc/passwd" file by running the following command.
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:2:2:daemon:/dev/null:/bin/false
messagebus:x:101:101:System user; messagebus:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/dev/null:/bin/false
EOF

# The actual password for "root" will be set later.
# Create the "/etc/group" file by running the following command.
cat > /etc/group << "EOF"
root:x:0:root
bin:x:1:root,bin,daemon
daemon:x:2:root,bin,daemon
sys:x:3:root,bin,adm
adm:x:4:root,bin,adm,daemon
tty:x:5:
disk:x:6:root,adm
lp:x:7:lp
kmem:x:9:
wheel:x:10:root
floppy:x:11:root
audio:x:18:
cdrom:x:19:
dialout:x:20:
tape:x:26:root
video:x:27:root
kvm:x:78:
usb:x:85:
input:x:97:
users:x:100:
messagebus:x:101:
utmp:x:406:
nogroup:x:65533:
nobody:x:65534:
EOF
```
```bash
# To remove the "I have no name!" prompt, start a new shell.
# Since the "/etc/passwd" and "/etc/group" files have been created, user name and group name resolution will now work.
exec /clang1-tools/bin/bash --login +h
```
```bash
# The login, agetty, and init programs (and others) use a number of log files to record information such as who was logged into the system and when.
# However, these programs will not write to the log files if they do not already exist.
# Initialize the log files and give them proper permissions.
touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```

### `5` - Setup Clang/LLVM Environment Variables
> Apply persistent toolchain environment variables, now set the compiler to Stage-1 Clang/LLVM default triplet (pc).
```bash
cat > ~/.bash_profile << "EOF"
# LLVM source directory.
export LLVM_SRC="/sources/llvm"

# Clang/LLVM Environment.
CC="x86_64-pc-linux-musl-clang"
CXX="x86_64-pc-linux-musl-clang++"
LD="ld.lld"
CC_LD="${LD}"
CXX_LD="${LD}"
AR="llvm-ar"
AS="llvm-as"
NM="llvm-nm"
OBJCOPY="llvm-objcopy"
OBJDUMP="llvm-objdump"
RANLIB="llvm-ranlib"
READELF="llvm-readelf"
SIZE="llvm-size"
STRIP="llvm-strip"
export CC CXX LD CC_LD CXX_LD AR AS NM OBJCOPY OBJDUMP RANLIB READELF SIZE STRIP

# Performance flags.
COMMON_FLAGS="-march=native -O2 -ftree-vectorize -pipe"
export COMMON_FLAGS

# Hardened flags. Only Buffer Overflow Detector.
export CPPFLAGS="-D_FORTIFY_SOURCE=2"

# Toolchain flags.
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
LDFLAGS="-Wl,-O2 -Wl,--as-needed"
MAKEFLAGS="-j$(nproc) -l$(nproc)"
export CFLAGS CXXFLAGS LDFLAGS MAKEFLAGS
EOF
source ~/.bash_profile
```

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

### `6` - Linux API Headers
> #### `5.13.x` (CacULE) or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
```bash
# Apply patch to fix `swab.h` under musl libc when building Linux kernel.
patch -Np1 -i ../../extra/linux-headers/patches/include-uapi-linux-swab-Fix-potentially-missing-__always_inline.patch

# Make sure there are no stale files embedded in the package.
time { make LLVM=1 LLVM_IAS=1 mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time { make LLVM=1 LLVM_IAS=1 HOSTCC=${CC} headers; }

# Remove unnecessary dotfiles and Makefile.
find usr/include \( -name '.*' -o -name 'Makefile' \) -exec rm -fv {} \;

# Install.
cp -afv usr/include /usr/.
```

### `7` - Iana-Etc
> #### `20210611` or newer
> The Iana-Etc package provides data for network services and protocols.

> **Required!** Before `Perl`.
```bash
# Only need to install these files into correct place.
install -vm644 -t /etc/ services protocols
```

### `8` - musl
> #### `1.2.2`
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Apply patches (from Void Linux and Alpine Linux).
for P in {epoll_cp,isascii,mo_lookup,handle-aux-at_base}.patch; do
    patch -Np1 -i ../../extra/musl/patches/${P}
done; unset P

# Configure source.
LDFLAGS="-Wl,-soname,libc.musl-x86_64.so.1" \
./configure --prefix=/usr                   \
            --sysconfdir=/etc               \
            --localstatedir=/var            \
            --disable-{gcc-wrapper,static}  \
            --with-malloc=mallocng          \
            --enable-optimize=speed

# Build.
time { make; }

# Install and create a `ldd` symlink to use to print shared object dependencies.
time {
    make install
    ln -sfv ../usr/lib/libc.so /bin/ldd
}
```
```bash
# Configure PATH for dynamic linker.
cat > /etc/ld-musl-x86_64.path << "EOF"
/lib
/usr/lib
/usr/local/lib
EOF
```
```bash
# Install fully-featured musl `ldconfig`.
# [ https://code.foxkit.us/smaeul/packages/-/commit/c631e4fc5ab64a9ad668ed5f753348ce8eae5219?view=parallel ]
install -vm755 -t /sbin/ ../../extra/musl/files/ldconfig

# Install `musl-legacy-compat` (from Void Linux).
for B in {cdefs,queue,tree}.h; do
    install -vm644 -t /usr/include/sys/ \
    ../../extra/musl/files/musl-legacy-compat/${B}
done; unset B; install -vm644 -t /usr/include/ \
../../extra/musl/files/musl-legacy-compat/error.h
```
```bash
# Configure Stage-1 Clang/LLVM with new triplet to produce binaries with "/lib/ld-musl-x86_64.so.1" and libraries from "/usr/*".
ln -sfv clang   /clang1-tools/bin/x86_64-heiwa-linux-musl-clang
ln -sfv clang++ /clang1-tools/bin/x86_64-heiwa-linux-musl-clang++
cat > /clang1-tools/bin/x86_64-heiwa-linux-musl.cfg << "EOF"
--sysroot=/usr -Wl,-dynamic-linker /lib/ld-musl-x86_64.so.1
EOF

# Set compiler to new triplet from Stage-1 Clang/LLVM.
sed -e "s|\"${CXX}\"|\"x86_64-heiwa-linux-musl-clang++\"|" \
    -e "s|\"${CC}\"|\"x86_64-heiwa-linux-musl-clang\"|" -i ~/.bash_profile
source                                                     ~/.bash_profile
```

<!--
### `8` - Microsoft mimalloc
> #### `2.0.2` or newer
> The Microsoft mimalloc package contains a compact general purpose allocator with excellent performance.

> **Required!** Microsoft mimalloc is for high perfomance purpose. We will build all packages to use mimalloc, except for libc.
```bash
# Configure mimalloc.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release    \
    -DCMAKE_INSTALL_PREFIX="/usr" \
    -DMI_BUILD_STATIC=OFF         \
    -DMI_BUILD_TESTS=OFF -Wno-dev
    
# Build mimalloc.
time { make -C build; }

# Install mimalloc.
time { make -C build install; }
```
```bash
# Set `mimalloc` as default C/C++ memory allocator, apply to compiler flags.
sed -i '/COMMON_FLAGS+=/s/ / -lmimalloc /'            ~/.bash_profile
printf '\n%s\n' 'export MIMALLOC_LARGE_OS_PAGES=1' >> ~/.bash_profile
source                                                ~/.bash_profile
```
-->
```bash
# Quick test for the new triplet of Stage-1 Clang/LLVM.
echo "int main(){}" > dummy.c
${CC} ${CFLAGS} dummy.c -v -Wl,--verbose &> dummy.log
${READELF} -l a.out | grep --color=auto "Req.*ter"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /lib/ld-musl-x86_64.so.1]

grep --color=auto "ld.lld:.*crt[1in].o" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /usr/lib/Scrt1.o
# |ld.lld: /usr/lib/crti.o
# |ld.lld: /usr/lib/crtn.o

# Check the headers path.
grep --color=auto -A1 -B1 "^ /usr/include" dummy.log

# | The output should be:
# |-----------------------
# |#include <...> search starts here:
# | /usr/include
# | /clang1-tools/lib/clang/12.0.1/include

# Check the dynamic linker libraries path.
grep -oE "\-L/usr/lib|\-L/lib" dummy.log

# | The output should be:
# |-----------------------
# |-L/usr/lib
```

### `9` - TimeZone Database
> #### `2021a` and `0.5` for posixtz
> The TZDb package contains code and data that represent the history of local time for many representative locations around the globe. It is updated periodically to reflect changes made by political bodies to time zone boundaries, UTC offsets, and daylight-saving rules.

> *No need to decompress any package firstly. It will be done in this step.*

> **Required!** Since using musl libc.
```bash
# Create a directory and decompress needed tarball.
mkdir -v tzdata && pushd tzdata && \
tar xzf ../tzdata2021a.tar.gz   && \
tar xzf ../tzcode2021a.tar.gz   && \
tar xf  ../posixtz-0.5.tar.xz
```
```bash
# Apply patches (from Alpine Linux).
patch -Np1 -i ../../extra/tzdata/patches/0001-posixtz-ensure-the-file-offset-we-pass-to-lseek-is-o.patch
patch -Np1 -i ../../extra/tzdata/patches/0002-fix-implicit-declaration-warnings-by-including-strin.patch

# Export the timezones variable.
read -rd '' timezones << "EOF"
africa antarctica asia australasia europe northamerica southamerica etcetera backward factory
EOF

# Build. -> Ignore "pkg-config: No such file or directory" when building `posixtz`! <-
time {
    make CC=${CC} TZDIR="/usr/share/zoneinfo" CFLAGS="-DHAVE_STDINT_H=1 -flto=thin $CFLAGS" && \
    make -C posixtz-0.5 CC=${CC} CFLAGS="-flto=thin $CFLAGS" posixtz
}

# Install utilities and the manpages.
install -vm755 -t /usr/bin/ tzselect z{ic,dump} posixtz-0.5/posixtz
install -vm644 -t /usr/share/man/man8/ z{ic,dump}.8

# Install zoneinfo. -> Ignore all warnings! <-
install -vm444 -t /usr/share/zoneinfo/ {iso3166,zone{1970,}}.tab
zic -b fat -y ./yearistype -d /usr/share/zoneinfo ${timezones}
zic -b fat -y ./yearistype -d /usr/share/zoneinfo/right -L leapseconds ${timezones}
zic -b fat -y ./yearistype -d /usr/share/zoneinfo -p America/New_York
```
```bash
# Configure timezone.
# Use `tzselect` to determine <xxx>.
tzselect

# | Example output of Asia/Jakarta:
# |---------------------------------
# |Here is that TZ value again, this time on standard output so that you
# |can use the /usr/bin/tzselect command in shell scripts:
# |Asia/Jakarta

cp -fv /usr/share/zoneinfo/<xxx> /etc/localtime
```
```bash
# Back to "/sources/pkgs" directory.
popd && rm -rf tzdata; unset timezones
```

### `10` - musl-locales
> #### `?` (git)
> The musl-locales package contains "/usr/bin/locale" implementation, which works on musl libc (with limitations in musl itself).

> **Required!**
```bash
# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=None -Wno-dev \
    -DCMAKE_INSTALL_PREFIX="/usr"    \
    -DCMAKE_C_FLAGS="-flto=thin $CFLAGS"

# Build.
time { make -C build; }

# Install.
time { make -C build install; }
```
```bash
# Load musl locale path for current shell.
. /etc/profile.d/00locale.sh
```

### `11` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** Before `Clang/LLVM`, `Kmod`, `Perl`, and `Util-linux`.
```bash
# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev  \
    -DCMAKE_INSTALL_PREFIX="/usr"        \
    -DCMAKE_C_FLAGS="-flto=thin $CFLAGS" \
    -DWITH_NATIVE_INSTRUCTIONS=YES       \
    -DZLIB_COMPAT=ON                     \
    -DBUILD_SHARED_LIBS=ON

# Build.
time { make -C build; }

# Install.
time { make -C build install; }
```

### `12` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** Before `GNU Bash`, `GNU Readline`, `GNU Texinfo`, and `Util-linux`.
```bash
# Build.
time { make CFLAGS="-flto=thin $CFLAGS" all-dynamic; }

# Install and create symlinks as `libtinfo` libraries (which actually replace GNU Ncurses).
time {
    make PREFIX=/usr install-{dynamic,manpages}
    ln -sfv libterminfo.so /usr/lib/libtinfo.so
    ln -sfv libterminfo.so /usr/lib/libtinfow.so
}
```
```bash
# Remove the manpages that colission with `attr`.
rm -fv /usr/share/man/man3/attr_{g,s}et.3
```

### `13` - GNU Readline
> #### `8.1` or newer
> The GNU Readline pacakage contains library providing line editing, history, and tokenisation functions.

> **Required!** Before `GNU Bash` and `GNU AWK`.
```bash
# Reinstalling Readline will cause the old libraries to be moved to <libraryname>.old.
# While this is normally not a problem, in some cases it can trigger a linking bug in `ldconfig`.
# This can be avoided by issuing the following two seds.
sed -i '/MV.*old/d'    Makefile.in
sed -i '/{OLDSUFF}/c:' support/shlib-install

# Configure source.
CFLAGS="-flto=thin $CFLAGS"  \
./configure --prefix=/usr    \
            --with-curses    \
            --disable-static \
            --docdir=/usr/share/doc/readline-8.1

# Build. Force to link against with curses library.
time { make SHLIB_LIBS="-lcurses -lterminfo"; }

# Install.
time { make SHLIB_LIBS="-lcurses -lterminfo" install; }
```
```bash
# Optional, install some additional documentation.
install -vm644 -t /usr/share/doc/readline-8.1/ doc/*.{ps,pdf,html,dvi}
```

### `14` - GNU M4
> #### `1.4.19` or newer
> The GNU M4 package contains a macro processor.

> **Required!** Before `Flex`, `Attr`, `Acl`, `GNU Autoconf`, and `GNU Automake`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"     \
ac_cv_header_sys_cdefs_h=no     \
ac_cv_lib_error_at_line=no      \
./configure --prefix=/usr       \
            --enable-changeword \
            --with-packager="Heiwa/Linux"

# Build.
time { make; }

# Install.
time { make install; }
```

### `15` - Flex
> #### `2.6.4` or newer
> The Flex package contains a utility for generating programs that recognize patterns in text.

> **Required!** Before `IPRoute2`, `Kbd`, and `Kmod`.
```bash
# Configure source. Flex still expect `cc` or `gcc` to configure.
CFLAGS="-flto=thin $CFLAGS"      \
ac_cv_func_malloc_0_nonnull=yes  \
ac_cv_func_realloc_0_nonnull=yes \
./configure --prefix=/usr        \
            --disable-shared     \
            --disable-static     \
            --disable-libfl      \
            --docdir=/usr/share/doc/flex-2.6.4

# Build.
time { make; }

# Install and create symlink as `lex`.
time {
    make install 
    ln -sfv flex /usr/bin/lex
}

# A few programs do not know about `flex` yet and try to run its predecessor, `lex`.
# To support those programs, create a symbolic link named `lex` that runs `flex` in `lex` emulation mode.
```

### `16` - Attr
> #### `2.5.1` or newer
> The Attr package contains utilities to administer the extended attributes on filesystem objects.

> **Required!** Before `ACL`, `libcap`, and `Shadow`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"             \
LDFLAGS="-Wl,--no-gc-sections $LDFLAGS" \
./configure --prefix=/usr               \
            --bindir=/bin               \
            --sysconfdir=/etc           \
            --disable-static            \
            --docdir=/usr/share/doc/attr-2.5.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `17` - ACL
> #### `2.3.1` or newer
> The ACL package contains utilities to administer Access Control Lists, which are used to define more fine-grained discretionary access rights for files and directories.

> **Required!** Before `Shadow`.
```bash
# Configure source. Don't use LTO, it breaks POSIX ACL in the binaries.
./configure --prefix=/usr    \
            --bindir=/bin    \
            --disable-static \
            --docdir=/usr/share/doc/acl-2.3.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `18` - libcap
> #### `2.55` or newer
> The libcap package implements the user-space interfaces to the POSIX 1003.1e capabilities available in Linux kernels. These capabilities are a partitioning of the all powerful root privilege into a set of distinct privileges.

> **Required!** Before `IPRoute2` and `Shadow`. Currently without linux-PAM support.
```bash
# Prevent static libraries from being installed.
sed -i '/install -m.*STA/d' libcap/Makefile

# Build. Fails with LTO since version 2.52 onwards.
time { make CC=${CC} SBINDIR=/sbin prefix=/usr lib=lib; }

# Install.
time { make SBINDIR=/sbin prefix=/usr lib=lib install; }
```

### `19` - Shadow
> #### `4.9` or newer
> The Shadow package contains programs for handling passwords in a secure way.

> **Required!** Currently without linux-PAM support.
```bash
# Disable the installation of the groups program and its manpages, as Coreutils (replaced by Toybox) provides a better version.
# Also, prevent the installation of manpages that included by Linux `man-pages`.
sed -i 's|groups$(EXEEXT) ||' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;

# Instead of using the default crypt method, use the more secure SHA-512 method of password encryption, which also allows passwords longer than 8 characters.
# It is also necessary to change the obsolete "/var/spool/mail" location for user mailboxes that Shadow uses by default to the "/var/mail" location used currently.
sed -e 's|#ENCRYPT_METHOD DES|ENCRYPT_METHOD SHA512|' \
    -e 's|/var/spool/mail|/var/mail|' -i etc/login.defs

# Completely disable unportable `ruserok()` on musl libc.
sed -i '/RUSEROK/d' configure

# Configure source.
# The file "/usr/bin/passwd" needs to exist because its location is harcoded in some programs, and the default location if it does not exist is not right.
touch /usr/bin/passwd
CFLAGS="-flto=thin $CFLAGS"                \
./configure --sysconfdir=/etc              \
            --disable-account-tools-setuid \
            --without-{tcb,group-name-max-length}

# Build.
time { make; }

# Install.
time {
    make libdir=/usr/lib suidperms=4711 install
    make -C {,install-}man
}
```
```bash
# Create the default configuration.
mkdir -v /etc/default && useradd -D --gid 100 && \
sed -i '/CREATE_MAIL_SPOOL=.*/d' /etc/default/useradd

# This package contains utilities to add, modify, and delete users and groups; set and change their passwords; and perform other administrative tasks.
# For a full explanation of what password shadowing means, see the doc/HOWTO file within the unpacked source tree.
# If using Shadow support, keep in mind that programs which need to verify passwords (display managers, FTP programs, pop3 daemons, etc.) must be Shadow-compliant.
# That is, they need to be able to work with shadowed passwords.
# Enable shadowed passwords and groups.
pwconv; grpconv
```

<!--
### `20` - libexecinfo
> #### `1.1` or newer (from Heiwa/Linux fork)
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** Before `Clang/LLVM`.
```bash
# Build.
time { make CFLAGS="-flto=thin $CFLAGS" dynamic; }

# Install.
time { make PREFIX=/usr install-{header,dynamic}; }
```
-->

### `20` - Clang/LLVM + libunwind, libcxxabi, and libcxx
> #### `12.x.x` or newer
> - C language family frontend for LLVM;  
> - C++ runtime stack unwinder from LLVM;  
> - Low level support for a standard C++ library from LLVM;  
> - New implementation of the C++ standard library, targeting C++11 from LLVM.

> **Required!**
```bash
# Exit from LLVM source directory if already entered after decompressing.
popd

# Rename LLVM source directory to "$LLVM_SRC", then enter.
mv -fv llvm-12.0.1.src "$LLVM_SRC" && pushd "$LLVM_SRC"
```
```bash
# Decompress `clang`, `lld`, `compiler-rt`, `libunwind`, `libcxxabi`, and `libcxx` to correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/compiler-rt-12.0.1.src.tar.xz && mv -fv compiler-rt{-12.0.1.src,}
    tar xf ../../pkgs/libunwind-12.0.1.src.tar.xz   && mv -fv libunwind{-12.0.1.src,}
    tar xf ../../pkgs/libcxxabi-12.0.1.src.tar.xz   && mv -fv libcxxabi{-12.0.1.src,}
    tar xf ../../pkgs/libcxx-12.0.1.src.tar.xz      && mv -fv libcxx{-12.0.1.src,}
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.1.src.tar.xz && mv -fv clang{-12.0.1.src,}
    tar xf ../../pkgs/lld-12.0.1.src.tar.xz   && mv -fv lld{-12.0.1.src,}
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/appatch

# Disable sanitizers for musl, it's broken since duplicates some libc bits.
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Update config.guess for better platform detection.
cp -fv ../extra/llvm/files/config.guess cmake/.
```
```bash
# Configure `libunwind` source.
pushd ${LLVM_SRC}/projects/libunwind/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"                      \
        -DCMAKE_C_FLAGS="-fPIC -flto=thin -g0 $CFLAGS"     \
        -DCMAKE_CXX_FLAGS="-fPIC -flto=thin -g0 $CXXFLAGS" \
        -DLLVM_PATH="$LLVM_SRC"                            \
        -DLIBUNWIND_ENABLE_STATIC=OFF                      \
        -DLIBUNWIND_USE_COMPILER_RT=ON

# Build.
time { make -C build; }

# Install, also the headers.
time {
    make -C build install             && \
    cp -fv include/*.h /usr/include/. && popd
}
```
```bash
# Configure `libcxxabi` source.
pushd ${LLVM_SRC}/projects/libcxxabi/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"                \
        -DCMAKE_CXX_FLAGS="-flto=thin -g0 $CXXFLAGS" \
        -DLLVM_PATH="$LLVM_SRC"                      \
        -DLIBCXXABI_ENABLE_STATIC=OFF                \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON             \
        -DLIBCXXABI_USE_COMPILER_RT=ON               \
        -DLIBCXXABI_LIBCXX_INCLUDES="${LLVM_SRC}/projects/libcxx/include"

# Build.
time { make -C build; }

# Install, also the headers.
time {
    make -C build install             && \
    cp -fv include/*.h /usr/include/. && popd
}
```
```bash
# Configure `libcxx` source.
pushd ${LLVM_SRC}/projects/libcxx/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/usr"                 \
        -DCMAKE_CXX_FLAGS="-flto=thin -g0 $CXXFLAGS"  \
        -DLLVM_PATH="$LLVM_SRC"                       \
        -DLIBCXX_ENABLE_STATIC=OFF                    \
        -DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF      \
        -DLIBCXX_CXX_ABI=libcxxabi                    \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/usr/include" \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/usr/lib"      \
        -DLIBCXX_HAS_MUSL_LIBC=ON                     \
        -DLIBCXX_USE_COMPILER_RT=ON

# Build.
time { make -C build; }

# Install.
time { make -C build install && popd; }
```
```bash
# Remove `libunwind`, `libcxxabi`, and `libcxx` from LLVM source directory.
rm -rf projects/lib{unwind,cxx{abi,}}
```
```bash
# Configure Clang/LLVM source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev                 \
    -DCMAKE_INSTALL_PREFIX="/usr"                       \
    -DCMAKE_C_FLAGS="-g0 $CFGLAGS"                      \
    -DCMAKE_CXX_FLAGS="-g0 $CXXFLAGS"                   \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-musl"           \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl" \
    -DLLVM_ENABLE_BINDINGS=OFF                          \
    -DLLVM_ENABLE_EH=ON                                 \
    -DLLVM_ENABLE_IDE=OFF                               \
    -DLLVM_ENABLE_LIBCXX=ON                             \
    -DLLVM_ENABLE_LLD=ON                                \
    -DLLVM_ENABLE_RTTI=ON                               \
    -DLLVM_ENABLE_BACKTRACES=OFF                        \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF                     \
    -DLLVM_ENABLE_WARNINGS=OFF                          \
    -DLLVM_ENABLE_LIBEDIT=OFF                           \
    -DLLVM_ENABLE_LIBXML2=OFF                           \
    -DLLVM_ENABLE_OCAMLDOC=OFF                          \
    -DLLVM_ENABLE_Z3_SOLVER=OFF                         \
    -DLLVM_ENABLE_LTO=Thin                              \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                       \
    -DLLVM_INCLUDE_EXAMPLES=OFF                         \
    -DLLVM_INCLUDE_TESTS=OFF                            \
    -DLLVM_INCLUDE_GO_TESTS=OFF                         \
    -DLLVM_INCLUDE_DOCS=OFF                             \
    -DLLVM_INSTALL_BINUTILS_SYMLINKS=ON                 \
    -DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON                  \
    -DLLVM_INSTALL_UTILS=ON                             \
    -DLLVM_LINK_LLVM_DYLIB=ON                           \
    -DLLVM_OPTIMIZED_TABLEGEN=ON                        \
    -DLLVM_TARGET_ARCH="X86"                            \
    -DLLVM_TARGETS_TO_BUILD="host;BPF;AMDGPU;X86"       \
    -DCLANG_VENDOR="Heiwa/Linux"                        \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                   \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                   \
    -DCLANG_DEFAULT_LINKER=lld                          \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                 \
    -DDEFAULT_SYSROOT="/usr"

# Build.
time { make -C build; }

# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/usr" -P cmake_install.cmake && \
    popd
}
```
```bash
# Create a symlink that required by FHS for "historical" reasons, and set `clang`, `clang++`, and `lld` as default toolchain compiler and linker.
ln -sfv ../usr/bin/clang /lib/cpp
ln -sfv clang            /usr/bin/cc
ln -sfv lld              /usr/bin/ld
sed -e "s|\"${CXX}\"|\"clang++\"|" \
    -e "s|\"${CC}\"|\"clang\"|" -i ~/.bash_profile    
source                             ~/.bash_profile
```
```bash
# Build useful utilities for BSD-compability.
time {
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/getconf.c -o getconf && \
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/getent.c -o getent   && \
    cc ${CFLAGS} -fpie ../extra/musl/files/musl-utils/iconv.c -o iconv
}

# Install above utilities, and the manpages (from NetBSD).
install -vm755 -t /usr/bin/ get{conf,ent} iconv
install -vm644 -t /usr/share/man/man1/ ../extra/musl/files/musl-utils/get{conf,ent}.1
```
```bash
# Quick test.
echo "int main(){}" > dummy.c
cc ${CFLAGS} dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep --color=auto "Req.*ter"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /lib/ld-musl-x86_64.so.1]

grep --color=auto "ld.lld:.*crt[1in].o" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /usr/bin/../lib/Scrt1.o
# |ld.lld: /usr/bin/../lib/crti.o
# |ld.lld: /usr/bin/../lib/crtn.o

# Check the headers path.
grep --color=auto -A1 -B1 "^ /usr/include" dummy.log

# | The output should be:
# |-----------------------
# |#include <...> search starts here:
# | /usr/include
# | /usr/lib/clang/12.0.1/include

# Check the dynamic linker libraries path.
grep -oE "\-L/usr/lib|\-L/lib" dummy.log

# | The output should be:
# |-----------------------
# |-L/usr/lib
```
```bash
# Back to "/sources/pkgs" directory.
popd
```

### `21` - Bzip2
> #### `1.0.8` or newer
> The Bzip2 package contains programs for compressing and decompressing files. Compressing text files with bzip2 yields a much better compression percentage than with the traditional gzip.

> **Required!** Before `Perl`.
```bash
# Apply patches to fix docs installation and library soname.
patch -Np1 -i ../../extra/bzip2/patches/install_docs-1.patch
patch -Np1 -i ../../extra/bzip2/patches/soname.patch

# Ensure installation of symlinks are relative and the manpages are installed into correct location.
# Also prevent to install static library.
sed -e 's|(PREFIX)/man|(PREFIX)/share/man|g'   \
    -e 's|\(ln -s -f \)$(PREFIX)/bin/|\1|'     \
    -e '/chmod a+r $(PREFIX)\/lib\/libbz2.a/d' \
    -e '/cp -f libbz2.a $(PREFIX)\/lib/d' -i Makefile
    
# Prepare.
time {
    make CFLAGS="-fPIC -flto=thin $CFLAGS" \
    -f Makefile-libbz2_so && make clean
}

# Build.
time { make CFLAGS="-fPIC -flto=thin $CFLAGS"; }

# Install (also shared libraries) and fix the symlinks.
time {
    make PREFIX=/usr install
    ln -sfv libbz2.so.1.0 libbz2.so && \
    install -vm755 -t /usr/lib/ libbz2.so*
    cp -fv  bzip2-shared bzip2      && \
    ln -sfv bzip2        bunzip2    && \
    ln -sfv bzip2        bzcat      && \
    install -vm755 -t /usr/bin/ b{un,}zip2 bzcat
}
```

### `22` -  Xz
> #### `5.2.5`
> The Xz package contains programs for compressing and decompressing files. It provides capabilities for the lzma and the newer xz compression formats. Compressing text files with xz yields a better compression percentage than with the traditional gzip or bzip2 commands.

> **Required!** Before `Kmod` and `Eudev`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/usr          \
            --enable-threads=posix \
            --disable-static       \
            --docdir=/usr/share/doc/xz-5.2.5

# Build.
time { make; }

# Install.
time { make install; }
```

### `23` - LZ4
> #### `1.9.3` or newer
> The LZ4 package contains library for lossless compression algorithm, providing compression speed > 500 MB/s per core, scalable with multi-cores CPU. It features an extremely fast decoder, with speed in multiple GB/s per core, typically reaching RAM speed limits on multi-core systems.

> **Required!** Before `zstd` and `Linux`.
```bash
# Configure source.
cmake -S build/cmake -B build \
    -DCMAKE_INSTALL_PREFIX="/usr" \
    -DBUILD_STATIC_LIBS=NO

# Build. Fails with LTO.
time { make -C build; }

# Install.
time { make -C build install; }
```

### `24` - Zstd
> #### `1.5.0` or newer
> The Zstd (Zstandard) package contains real-time compression algorithm, providing high compression ratios. It offers a very wide range of compression / speed trade-offs, while being backed by a very fast decoder.

> **Required!**  Before `Kmod`.
```bash
# Build zstd and pzstd (parallel zstandard).
time {
    make CFLAGS="-flto=thin $CFLAGS" && \
    make -C contrib/pzstd CFLAGS="-flto=thin $CFLAGS"
}

# Install.
time {
    make PREFIX=/usr install && \
    make -C contrib/pzstd PREFIX=/usr install
}
```
```bash
# Delete useless static library.
rm -fv /usr/lib/libzstd.a
```

### `25` - Pigz
> #### `2.6` or newer
> The Pigz package contains parallel implementation of gzip, is a fully functional replacement for GNU zip that exploits multiple processors and multiple cores to the hilt when compressing data.

> **Required!**
```bash
# Make sure to use symlink instead of hardlink for `unpigz`.
sed -i 's|ln -f|ln -sf|' Makefile

# Build.
time { make CC=${CC} CFLAGS="-flto=thin $CFLAGS"; }

# Install and create symlinks as `gzip` tools.
ln -sfv pigz gzip; ln -sfv unpigz gunzip
install -vm755 -t /usr/bin/ {,un}pigz g{,un}zip
```

### `26` - Pkgconf
> #### `1.8.0` or newer
> The Pkgconf package contains a tool for passing the include path and/or library paths to build tools during the configure and make phases of package installations.

> **Required!** Before `Kmod`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"   \
./configure --prefix=/usr     \
            --sysconfdir=/etc \
            --disable-static  \
            --docdir=/usr/share/doc/pkgconf-1.8.0

# Build.
time { make; }

# Install and create symlink as `pkg-config`.
time {
    make install
    ln -sfv pkgconf /usr/bin/pkg-config
}
```

### `27` - Gettext-tiny
> #### `0.3.2` or newer
> The Gettext-tiny package contains utilities for internationalization and localization. These allow programs to be compiled with NLS (Native Language Support), enabling them to output messages in the user's native language. A lightweight replacements for tools typically used from the GNU gettext suite, which is incredibly bloated and takes a lot of time to build (in the order of an hour on slow devices).

> **Required!** Before `GNU Bison` and `GNU Automake`.
```bash
# Apply patches to fix some issues.
patch -Np1 -i ../../extra/gettext-tiny/patches/line-length.patch
patch -Np1 -i ../../extra/gettext-tiny/patches/flip-macro-logic.patch

# Build.
time { make LIBINTL=MUSL CFLAGS="-flto=thin $CFLAGS"; }

# Install.
time { make LIBINTL=MUSL prefix=/usr install; }
```

### `28` - GNU Bison
> #### `3.7.6` or newer
> The GNU Bison package contains a parser generator.

> **Required!** Before `Kmod`, `Kbd`, and `Util-Linux`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
ac_cv_header_sys_cdefs_h=no \
ac_cv_lib_error_at_line=no  \
./configure --prefix=/usr   \
            --docdir=/usr/share/doc/bison-3.7.6

# Build.
time { make; }

# Install.
time { make install; }
```
```bash
# Delete useless static library.
rm -fv /usr/lib/liby.a
```

### `29`- GDBM
> #### `1.20` or newer
> The GDBM package contains the GNU Database Manager. It is a library of database functions that use extensible hashing and works similar to the standard UNIX dbm. The library provides primitives for storing key/data pairs, searching and retrieving the data by its key and deleting a key along with its data.

> **Required!** Before `Perl`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"  \
./configure --prefix=/usr    \
            --disable-static \
            --enable-libgdbm-compat

# Build.
time { make; }

# Install.
time { make install; }
```

### `30` - Perl
> #### `5.34.0` or newer
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** Before `OpenSSL` and `GNU Autoconf`.
<!--
* BUILD FAILS SINCE USING ZLIB-NG.
Ensure to build Perl with libraries installed on the system.
BUILD_BZIP2=0 BUILD_ZLIB=False
BZIP2_LIB="/usr/lib" ZLIB_LIB="/usr/lib"
BZIP2_INCLUDE="/usr/include" ZLIB_INCLUDE="/usr/include"
export BUILD_{BZIP2,ZLIB} {BZIP2,ZLIB}_{LIB,INCLUDE}
-->
```bash
# Configure source.
HOSTLDFLAGS="-pthread" HOSTCFLAGS="-D_GNU_SOURCE" ./Configure -des    \
    -Dusethreads                                                      \
    -Duseshrplib                                                      \
    -Dusesoname                                                       \
    -Dusevendorprefix                                                 \
    -Duselargefiles                                                   \
    -Dprefix=/usr                                                     \
    -Dvendorprefix=/usr                                               \
    -Dprivlib=/usr/lib/perl5/5.34/core_perl                           \
    -Darchlib=/usr/lib/perl5/5.34/core_perl                           \
    -Dsitelib=/usr/lib/perl5/5.34/site_perl                           \
    -Dsitearch=/usr/lib/perl5/5.34/site_perl                          \
    -Dvendorlib=/usr/lib/perl5/5.34/vendor_perl                       \
    -Dvendorarch=/usr/lib/perl5/5.34/vendor_perl                      \
    -Doptimize="-DNO_POSIX_2008_LOCALE -D_GNU_SOURCE $CFLAGS"         \
    -Dccflags="-DNO_POSIX_2008_LOCALE -D_GNU_SOURCE $CFLAGS"          \
    -Dcccdlflags="-fPIC"                                              \
    -Dldflags="-Wl,-z,stack-size=2097152 -pthread $LDFLAGS"           \
    -Dlddlflags="-shared -Wl,-z,stack-size=2097152 -pthread $LDFLAGS" \
    -Dperl_static_inline="static __inline__"                          \
    -Dd_static_inline                                                 \
    -Dcf_by="Heiwa/Linux"                                             \
    -Dmyhostname="localh3art"                                         \
    -Dperladmin="root@localh3art"                                     \
    -Dman1ext=1                                                       \
    -Dman3ext=3pm                                                     \
    -Dman1dir=/usr/share/man/man1                                     \
    -Dman3dir=/usr/share/man/man3                                     \
    -Dd_semctl_semun -Ud_csh

# Build. Fails with LTO since v5.28. This will display a lot of compiler warnings.
time { make; }

# Install.
time { make install; }
```

### `31` - OpenSSL
> #### `1.1.1l` or newer
> The OpenSSL package contains management tools and libraries relating to cryptography. These are useful for providing cryptographic functions to other packages, such as OpenSSH, email applications, and web browsers (for accessing HTTPS sites).

> **Required!** Before `Toybox` and `Kmod`.
```bash
# Configure source. Optimized for x86_64. No need to specify flags variable in configure, since Makefile respect flags.
./Configure linux-x86_64        \
    --prefix=/usr               \
    --libdir=lib                \
    --openssldir=/etc/ssl       \
    shared threads zlib-dynamic \
    no-ssl3-method no-async     \
    enable-ec_nistp_64_gcc_128  \
    -DOPENSSL_NO_BUF_FREELISTS  \
    -fno-strict-aliasing

# Prevent to install static library.
sed -i '/INSTALL_LIBS=libcrypto.a libssl.a/d' Makefile

# Build. Fails with LTO.
time { make; }

# Install.
time { make MANSUFFIX=ssl install; }
```
```bash
# Add the version to the documentation directory name, to be consistent with other packages.
mv -fv /usr/share/doc/openssl{,-1.1.1l}

# Optional, install some additional documentation.
cp -rfv doc/* /usr/share/doc/openssl-1.1.1l/.
```

### `32` - Toybox (Bc, Coreutils, File, Findutils, Grep, Inetutils, Man, Psmisc, Sed, Sysklogd, Tar)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!**
```bash
# Copy the Toybox .config file.
cp -v ../../extra/toybox/files/.config.finalsys .config

# Ensure `libcrypto` and `libz` are enabled in the config.
grep --color=auto -E "CONFIG_TOYBOX_(LIBCRYPTO|LIBZ)=y" .config

# Export commands as TOYBOX variable that will be use to verify Toybox .config.
read -rd '' TOYBOX << "EOF"
bc base64 base32 basename cat chgrp chmod chown chroot cksum comm cp cut date dd df
dirname du echo env expand expr factor false fmt fold groups head hostid id install
link ln logname ls md5sum mkdir mkfifo mknod mktemp mv nice nl nohup nproc od paste
printenv printf pwd readlink realpath rm rmdir seq sha1sum sha224sum sha256sum sha384sum
sha512sum shred sleep sort split stat stty sync tac tail tee test timeout touch tr true
truncate tty uname uniq unlink wc who whoami yes file find xargs egrep grep fgrep
dnsdomainname ifconfig hostname ping telnet tftp traceroute man killall sed klogd tar
EOF

# Verify 102 commands, and make sure enabled (=y).
# Pipe to ` | wc -l` at the right of `done` to check the total of commands.
for X in ${TOYBOX}; do
    grep -v '#' .config | grep -i --color=auto "CONFIG_${X}=" \
    || 2>&1 echo "> ${X} not CONFIGURED!"
done

# Build with verbose.
time { make CFLAGS="-flto=thin $CFLAGS" V=1; }

# Verify compiled 102 commands.
./toybox | tr ' ' '\n'i \
| grep -xE --color=auto $(echo ${TOYBOX} | tr ' ' '|'i) | wc -l

# Verify unconfigured commands, but compiled.
# `[` (coreutils)
# `ping6` and `traceroute6` (inetutils)
./toybox | tr ' ' '\n'i \
| grep -vxE --color=auto $(echo ${TOYBOX} | tr ' ' '|'i)

# So, totally is 105 commands.
./toybox | wc -w

# Install.
time { make PREFIX=/ install && unset X TOYBOX; }
```

### `33` - GNU AWK
> #### `5.1.0` or newer
> The GNU AWK (gawk) package contains programs for manipulating text files.

> **Required!**
```bash
# Ensure some unneeded files are not installed, use symlinks rather than hardlinks, and disable version links.
sed -e '/^LN =/s|=.*|= $(LN_S)|'         \
    -e '/install-exec-hook:/s|$|\nfoo:|' \
    -e 's|extras||' -i Makefile.in doc/Makefile.in

# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr

# Build.
time { make; }

# Install and create symlink as `awk`, not automatically linked since disabled version links.
time {
    make install
    ln -sfv gawk /usr/bin/awk
}
```
```bash
# Optional, install some additional documentation.
mkdir -pv /usr/share/doc/gawk-5.1.0 && \
cp -fv doc/{awkforai.txt,*.{eps,pdf,jpg}} /usr/share/doc/gawk-5.1.0/.
```

### `34` - GNU Diffutils
> #### `3.8` or newer
> The GNU Diffutils package contains programs that show the differences between files or directories.

> **Required!**
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
ac_cv_header_sys_cdefs_h=no \
ac_cv_lib_error_at_line=no  \
./configure --prefix=/usr   \
            --with-packager="Heiwa/Linux"

# Build.
time { make; }

# Install.
time { make install; }
```

### `35` - GNU Make
> #### `4.3` or newer
> The GNU Make package contains a program for controlling the generation of executables and other non-source files of a package from source files.
 
> **Required!**
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr   \
            --without-guile

# Build.
time { make; }

# Install.
time { make install; }
```

### `35` - GNU Patch
> #### `2.7.6` or newer
> The GNU Patch package contains a program for modifying or creating files by applying a patch file typically created by the diff program.

> **Required!**
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
ac_cv_header_sys_cdefs_h=no \
ac_cv_lib_error_at_line=no  \
./configure --prefix=/usr

# Build.
time { make; }

# Install.
time { make install; }
```

### `37` - GNU Texinfo
> #### `6.8` or newer
> The Texinfo package contains programs for reading, writing, and converting info pages.

> **Required!** Before `GNU Bash`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                   \
ac_cv_header_sys_cdefs_h=no                   \
ac_cv_lib_error_at_line=no                    \
./configure --prefix=/usr                     \
            --disable-perl-xs                 \
            --without-external-libintl-perl   \
            --without-external-Text-Unidecode \
            --without-external-Unicode-EastAsianWidth

# Build.
time { make; }

# Install.
time { make install; }
```

### `38` - GNU Bash
> #### `5.1` (with patch level 8) or newer
> The GNU Bash package contains the Bourne-Again SHell.

> **Required!**
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"           \
./configure --prefix=/usr             \
            --with-curses             \
            --without-bash-malloc     \
            --with-installed-readline \
            --disable-profiling       \
            --enable-readline         \
            --enable-net-redirections \
            --docdir=/usr/share/doc/bash-5.1

# Build.
time { make; }

# Install and move the `bash` binary to correct place.
time {
    make install
    mv -fv /usr/bin/bash /bin/.
}
```

### `39` - Kmod
> #### `29` or newer
> The Kmod package contains libraries and utilities for loading kernel modules

> **Required!** Before `Eudev`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"   \
./configure --prefix=/usr     \
            --bindir=/bin     \
            --sysconfdir=/etc \
            --with-zlib       \
            --with-xz         \
            --with-zstd       \
            --with-openssl

# Build.
time { make; }

# Install and create symlinks for compatibility with Module-Init-Tools (the package that previously handled Linux kernel modules).
time {
    make install
    for B in {dep,ins,rm}mod modprobe; do
        ln -sfv ../bin/kmod /sbin/${B}
    done; unset B
    for B in lsmod modinfo; do
        ln -sfv kmod /bin/${B}
    done; unset B
}
```

### `40` - GNU libtool
> #### `2.4.6` or newer
> The GNU libtool package contains the GNU generic library support script. It wraps the complexity of using shared libraries in a consistent, portable interface.

> **Required!** Before `musl-fts` and `musl-obstack`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr   \
            --disable-ltdl-install

# Build.
time { make; }

# Install.
time { make install; }
```

### `41` - GNU Autoconf
> #### `2.71` or newer
> The GNU Autoconf package contains programs for producing shell scripts that can automatically configure source code.

> **Required!** Before `GNU Automake`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr

# Build.
time { make; }

# Install.
time { make install; }
```

### `42` - GNU Automake
> #### `1.16.4` or newer
> The GNU Automake package contains programs for generating Makefiles for use with Autoconf.

> **Required!** Before `musl-fts` and `musl-obstack`.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr   \
            --docdir=/usr/share/doc/automake-1.16.4

# Build.
time { make; }

# Install.
time { make install; }
```

### `43` - argp-standalone
> #### `1.3` or newer
> The argp-standalone package contains hierarchial argument parsing library broken out from GNU libc (glibc).

> **Required!** Before `Elfutils - libelf`.
```bash
# Apply patch (from Alpine Linux) to remove `__THROW` in function implementation.
patch -Np1 -i ../../extra/argp-standalone/001-throw-in-funcdef.patch

# Configure source.
CFLAGS="-fPIC -flto=thin $CFLAGS -fgnu89-inline" \
./configure --prefix=/usr                        \
            --sysconfdir=/etc

# Build.
time { make; }

# Install.
time {
    install -vm644 -t /usr/include/ argp.h
    install -vm755 -t /usr/lib/  libargp.a
}
```

### `44` - musl-fts
> #### `1.2.7` or newer
> The musl-fts package contains implementation of fts(3) for musl libc.

> **Required!** Before `Elfutils - libelf`.
```bash
# Prepare.
sed -i '/pkgconfig_DATA/i pkgconfigdir=$(libdir)/pkgconfig' Makefile.am
./bootstrap.sh

# Configure source.
CFLAGS="-fPIC -flto=thin $CFLAGS" \
./configure --prefix=/usr         \
            --sysconfdir=/etc

# Build.
time { make; }

# Install.
time { make install; }
```
```bash
# Delete useless static library.
rm -fv /usr/lib/libfts.a
```

### `45` - musl-obstack
> #### `1.2.2` or newer
> The musl-obstack package contains a standalone library to implement GNU libc obstack.

> **Required!** Before `Elfutils - libelf`.
```bash
# Prepare.
sed -i '/pkgconfig_DATA/i pkgconfigdir=$(libdir)/pkgconfig' Makefile.am
./bootstrap.sh

# Configure source.
CFLAGS="-fPIC -flto=thin $CFLAGS" \
./configure --prefix=/usr         \
            --sysconfdir=/etc

# Build.
time { make; }

# Install.
time { make install; }
```
```bash
# Delete useless static library.
rm -fv /usr/lib/libobstack.a
```

### `46` - Elfutils - libelf
> #### `0.185` or newer
> The Elfutils package contains library for handling ELF (Executable and Linkable Format) files.

> **Required!** Before `IPRoute2`.
```bash
# Apply patch to allow build with Clang under musl libc.
patch -Np1 -i ../../extra/elfutils/patches/elfutils-musl-clang.patch

# Prevent to build static library.
sed -e '/^lib_LIBRARIES/s:=.*:=:' \
    -e '/^%.os/s:%.o$::' -i lib{asm,dw,elf}/Makefile.in

# Configure source.
CFLAGS="-Wno-error -Wno-null-dereference -DFNM_EXTMATCH=0 -flto=thin $CFLAGS" \
CXXFLAGS="-Wno-error -Wl,-z,stack-size=2097152 -flto=thin $CXXFLAGS"          \
./configure --prefix=/usr                                                     \
            --disable-{{,lib}debuginfod,werror} ac_cv_c99=yes

# Only build `libelf`.
time { make -C lib && make -C libelf; }

# Install.
time {
    make -C libelf install && \
    install -vm644 -t /usr/lib/pkgconfig/ config/libelf.pc 
}
```

### `47` - IPRoute2
> #### `5.13.0` or newer
> The IPRoute2 package contains programs for basic and advanced IPV4-based networking.

> **Required!**
```bash
# The arpd program included in this package will not be built since it is dependent on Berkeley DB, which currently not installed. 
# However, a directory for `arpd` and a man page will still be installed.
# Prevent this by running the commands below.
sed -i /ARPD/d Makefile
rm -fv man/man8/arpd.8

# Prevent build two modules that require `iptables`.
sed -i 's|.m_ipt.o||' tc/Makefile

# Build with verbose.
time { make CC=${CC} CCOPTS="-D_GNU_SOURCE" V=1; }

# Install.
time { make install; }
```
```bash
# Optional, install some additional documentation.
mkdir -pv /usr/share/doc/iproute2-5.13.0 && \
cp -fv COPYING README* /usr/share/doc/iproute2-5.13.0
```

### `48` - KBD
> #### `2.4.0` or newer
> The Kbd package contains key-table files, console fonts, and keyboard utilities.

> **Required!**
```bash
# The behaviour of the backspace and delete keys is not consistent across the keymaps in the Kbd package.
# The following patch fixes this issue for i386 keymaps.
patch -Np1 -i ../../extra/kbd/patches/kbd-2.4.0-backspace-1.patch

# After patching, the backspace key generates the character with code 127, and the delete key generates a well-known escape sequence.

# Remove the redundant resizecons program (it requires the defunct svgalib to provide the video mode files - for normal use setfont sizes the console appropriately) together with its manpage.
sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in

# Rename keymap files with the same names.
# This is needed because when only name of keymap is specified loadkeys loads the first keymap it can find, which is bad.
# This should be removed when upstream adopts the change.
mv -fv data/keymaps/i386/qwertz/cz{,-qwertz}.map
mv -fv data/keymaps/i386/olpc/es{,-olpc}.map
mv -fv data/keymaps/i386/olpc/pt{,-olpc}.map
mv -fv data/keymaps/i386/fgGIod/trf{,-fgGIod}.map
mv -fv data/keymaps/i386/colemak/{en-latin9,colemak}.map

# 7-bit maps are obsolete, so are non-euro maps.
pushd data/keymaps/i386/ && \
    cp -fv qwerty/pt-latin9.map qwerty/pt.map        && \
    cp -fv qwerty/sv-latin1.map qwerty/se-latin1.map && \
    mv -fv azerty/fr.map azerty/fr-old.map           && \
    cp -fv azerty/fr-latin9.map azerty/fr.map        && \
    cp -fv azerty/fr-latin9.map azerty/fr-latin0.map && \
popd

# Configure source.
CFLAGS="-flto=thin $CFLAGS" \
./configure --prefix=/usr   \
            --disable-{static,vlock}

# Build.
time { make; }

# Install.
time { make install; }

# Remove keymaps for `sun`, `amiga` and `atari`.
for K in sun amiga atari; do
    rm -rfv /usr/share/keymaps/${K}
done; unset K
```
```bash
# Optional, install some additional documentation.
mkdir -pv /usr/share/doc/kbd-2.4.0 && \
cp -rfv docs/doc/* /usr/share/doc/kbd-2.4.0
```

### `49` - Util-linux
> #### `2.37.2` or newer
> The Util-linux package contains miscellaneous utility programs.

> **Required!** Before `E2fsprogs` and `Eudev`.
```bash
# Apply patch (from Void Linux) to allow compile under musl libc.
patch -Np1 -i ../../extra/util-linux/patches/fix-musl.patch

# Fix `more.c` since fails to build againts NetBSD Curses.
sed -i 's|, 0, 0|, 0, 0, 0, 0, 0, 0, 0, 0, 0|' text-utils/more.c

# The FHS recommends using the "/var/lib/hwclock" directory instead of the usual "/etc" directory as the location for the adjtime file. 
mkdir -pv /var/lib/hwclock

# Generate configure script.
NOCONFIGURE=1 ./autogen.sh

# Configure source. musl needs `-D_DIRENT_HAVE_D_TYPE` for switch_root(8).
CFLAGS="-D_DIRENT_HAVE_D_TYPE -flto=thin $CFLAGS"   \
./configure ADJTIME_PATH=/var/lib/hwclock/adjtime   \
            --libdir=/usr/lib                       \
            --docdir=/usr/share/doc/util-linux-2.37 \
            --disable-chfn-chsh                     \
            --disable-login                         \
            --disable-nologin                       \
            --disable-su                            \
            --disable-setpriv                       \
            --disable-runuser                       \
            --disable-pylibmount                    \
            --disable-static                        \
            --without-python                        \
            --without-systemd                       \
            --without-systemdsystemunitdir          \
            runstatedir=/run

# Build.
time { make; }

# Install and fix permissions (add SUID).
time {
    make install
    chmod -v u+s /usr/bin/{newgrp,chsh,chfn}
}
```

### `50` - E2fsprogs
> #### `1.46.2` or newer
> The E2fsprogs package contains the utilities for handling the ext2 file system. It also supports the ext3 and ext4 journaling file systems.

> **Required!**
```bash
# Create a dedicated directory and configure source.
mkdir -v build && cd build
../configure --prefix=/usr       \
             --sysconfdir=/etc   \
             --enable-elf-shlibs \
             --disable-libblkid  \
             --disable-libuuid   \
             --disable-uuidd     \
             --disable-fsck      \
             --disable-rpath

# Build.
time { make; }

# Install (fixed dir installation).
time {
    make MKDIR_P="install -d" install
    for B in e2fsck mke2fs {mkfs,fsck}.ext{2..4}; do
        mv -fv /usr/sbin/${B} /sbin/.
    done; unset B
}

# This package installs a gzipped .info file but doesn't update the system-wide dir file.
# Unzip this file and then update the system dir file using the following commands.
gunzip -v /usr/share/info/libext2fs.info.gz
install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info
```

### `52` - OpenRC and additional services
> #### `0.43.3` or newer
> OpenRC is a dependency-based init system that works with the system-provided init program, normally /sbin/init.

> **Required!**
```bash
# Apply patches (from Alpine Linux).
patch -Np1 -i ../../extra/openrc/patches/0009-Support-early-loading-of-keymap-if-kdb-is-installed.patch
patch -Np1 -i ../../extra/openrc/patches/0014-time_t-64bit.patch

# Build.
MAKE_ARGS="
    LIBNAME=/usr/lib
    LIBDIR=/usr/lib
    PKGCONFIGDIR=/usr/lib/pkgconfig
    LIBEXECDIR=/lib/rc
    MKBASHCOMP=yes
    MKNET=no
    MKPREFIX=no
    MKSELINUX=no
    MKSTATICLIBS=no
    MKSYSVINIT=yes
    MKTERMCAP=ncurses
    MKZSHCOMP=yes
    PKG_PREFIX=''
    BRANDING=Heiwa/Linux
    OS=Linux
    SH=/bin/sh"

time { make ${MAKE_ARGS}; }

# Install.
time { make ${MAKE_ARGS} DESTDIR=/ install; }
```
```bash
# Decompress opentmpfiles.
tar xzf ../opentmpfiles-0.3.1.tar.gz && \
pushd opentmpfiles-0.3.1/

# Install opentmpfiles.
time {
    make install
    for C in opentmpfiles-dev opentmpfiles-setup; do
        install -vm644 openrc/${C}.confd /etc/conf.d/${C}
        install -vm755 openrc/${C}.initd /etc/init.d/${C}
    done && popd; unset C
}
```
```bash
# Decompress udev-gentoo-scripts.
tar xzf ../udev-gentoo-scripts-34.tar.gz && \
pushd udev-gentoo-scripts-34/

# Install udev-gentoo-scripts.
time { make install && popd; }
```

### `53` - GNU perf
> The GNU perf (gperf) package contains utility that generates a perfect hash function from a key set.
> #### `3.1` or newer

> **Required!** Before `Eudev`.
```bash
# Configure source.
./configure --prefix=/usr \
            --docdir=/usr/share/doc/gperf-3.1

# Build.
time { make; }

# Install.
time { make install; }
```

### `54` - Eudev
> #### `3.2.10` or newer
> The Eudev package contains programs for dynamic creation of device nodes.

> **Required!**
```bash
# Generate configure script.
autoreconf -fvi

# Configure source.
./configure --prefix=/usr                   \
            --bindir=/sbin                  \
            --sbindir=/sbin                 \
            --sysconfdir=/etc               \
            --with-rootlibexecdir=/lib/udev \
            --enable-manpages               \
            --disable-static

# Build.
time { make; }

# Install.
time { make install; }
```
```bash
# Information about hardware devices is maintained in the "/etc/udev/hwdb.d" and "/usr/lib/udev/hwdb.d" directories.
# Eudev needs that information to be compiled into a binary database "/etc/udev/hwdb.bin". Create the initial database.
udevadm hwdb --update
```

### `55` - Cmake
> #### `3.20.5` or newer
> The CMake package contains a modern toolset used for generating Makefiles. It is a successor of the auto-generated configure script and aims to be platform- and compiler-independent. A significant user of CMake is KDE since version 4.

> **Optional!** Currently not really required since using from "/clang1-tools", but useful for future usage.
```bash
# Disable applications that using Cmake from attempting to install files in "/usr/lib64".
sed -i '/"lib64"/s/64//' Modules/GNUInstallDirs.cmake

# Configure source using system installed libraries.
./bootstrap --prefix=/usr       \
            --system-zlib       \
            --system-bzip2      \
            --system-liblzma    \
            --system-zstd       \
            --mandir=/share/man \
            --parallel=$(nproc) \
            --docdir=/share/doc/cmake-3.20.5

# Build.
time { make; }

# Install.
time { make install; }
```

### `56` - cpio
> #### `2.13` or newer
> The cpio package contains tools for archiving.

> **Required!** Before `Linux`.
```bash
# Configure source.
ac_cv_lib_error_at_line=no  \
CFLAGS="-fcommon $CFLAGS"   \
./configure --prefix=/usr   \
            --bindir=/bin   \
            --disable-mt    \
            --disable-rpath \
            --enable-largefile

# Build.
time { make; }

# Install.
time { make install; }
```

### `57` - Python3
> #### `3.9.6` or newer
> The Python3 package contains the Python development environment. It is useful for object-oriented programming, writing scripts, prototyping large programs, or developing entire applications.

> **Optional!** Currently not required, but useful for future usage.
```bash
# Apply patch (from Void Linux) to allow compile under musl libc.
patch -Np1 -i ../../extra/python3/patches/musl-find_library.patch

# Configure source using provided libraries (built-in).
ax_cv_c_float_words_bigendian=no         \
./configure --prefix=/usr                \
            --enable-shared              \
            --with-ensurepip=yes         \
            --with-computed-gotos        \
            --with-lto                   \
            --enable-ipv6                \
            --enable-optimizations       \
            --with-dbmliborder=gdbm:ndbm \
            --enable-loadable-sqlite-extensions

# Build. -> Ignore all issues! <-
time { make; }

# Install.
time { make install; }
```

### `58` - GNU Nano
> #### `5.8` or newer
> The Nano package contains a small, simple text editor which aims to replace Pico, the default editor in the Pine package.

> **Optional!** For text editing (configuration).
```bash
# Configure source.
./configure --prefix=/usr     \
            --sysconfdir=/etc \
            --enable-utf8     \
            --docdir=/usr/share/doc/nano-5.8

# Build.
time { make; }

# Install.
time { make install; }
```

### `59` - Cleaning Up
> #### This section is optional!

> If the intended user is not a programmer and does not plan to do any debugging on the system software, the system size can be decreased by removing the debugging symbols from binaries and libraries. This causes no inconvenience other than not being able to debug the software fully anymore.
```bash
# The libtool .la files are only useful when linking with static libraries.
# They are unneeded, and potentially harmful, when using dynamic shared libraries, specially when using non-autotools build systems.
# Remove those files.
find /usr/{lib,libexec} -name \*.la -exec rm -rfv {} \;

# Strip off debugging symbols from binaries using `llvm-strip`.
# A large number of files will be reported "The file was not recognized as a valid object file".
# These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
find /usr/lib -type f -name \*.a -exec llvm-strip --strip-debug {} \;
find /lib /usr/lib -type f -name \*.so* ! -name \*dbg -exec llvm-strip --strip-unneeded {} \;
/clang1-tools/usr/bin/find /{,s}bin /usr/{{,s}bin,libexec} -type f -exec /clang1-tools/bin/llvm-strip --strip-all {} \;
```
```bash
# Cleaning leftover files.
rm -rfv /tmp/* ~/.python*

# The clang0 and clang1 toolchains is still partially installed and not needed anymore.
# Now safe to remove "/clang0-tools" and "/clang1-tools" directories as they're not required anymore.
rm -rf /clang{0,1}-tools
```
> #### * End of as root in a chroot env!

> #### * Beginning of as root!
```bash
# Now log out and reenter the chroot environment with an updated chroot command.
# From now on, use this updated chroot command any time you need to reenter the chroot environment after exiting.
logout

# From now on, use this updated chroot command any time you need to reenter the chroot environment after exiting.
# Term variable is set to `xterm` for better compability, instead of "$TERM" that will broken if using `rxvt-unicode`.
if [[ -n "$HEIWA" ]]; then
    chroot "$HEIWA" /usr/bin/env -i      \
    HOME="/root" TERM="xterm"            \
    PS1='(heiwa chroot) \u: \w \$ '      \
    PATH="/usr/sbin:/usr/bin:/sbin:/bin" \
    /bin/bash --login
fi
```
> #### * End of as root!

> #### * Beginning of as root in a chroot env!

<h2></h2>

~Continue to [System Configuration](./5-System_Configuration.md).~ (under developments)
