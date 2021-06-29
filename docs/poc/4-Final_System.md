## `IV` Final System

> #### * Beginning of as root!
### `1` - Preparing Virtual Kernel File Systems
> Various file systems exported by the kernel are used to communicate to and from the kernel itself. These file systems are virtual in that no disk space is used for them. The content of the file systems resides in memory.
```bash
# When the kernel boots the system, it requires the presence of a few device nodes, in particular the console and null devices.
# The device nodes must be created on the hard disk so that they are available before the kernel populates "/dev", and additionally when Linux is started with init=/bin/bash.
# Create the directories and initial device nodes.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv "${HEIWA}/"{dev,proc,sys,run,tmp}   && \
    mknod -m 600 "${HEIWA}/dev/console" c 5 1 && \
    mknod -m 666 "${HEIWA}/dev/null" c 1 3
fi

# Mounting and Populating VKFS
# The recommended method of populating the "/dev" directory with devices is to mount a virtual filesystem (such as tmpfs) on the "/dev" directory, and allow the devices to be created dynamically on that virtual filesystem as they are detected or accessed.
# Device creation is generally done during the boot process by Udev.
# Since this new system does not yet have Udev and has not yet been booted, it is necessary to mount and populate "/dev" manually.
# This is accomplished by bind mounting the host system's "/dev" directory.
# A bind mount is a special type of mount that allows you to create a mirror of a directory or mount point to some other location.
# In some host systems, "/dev/shm" is a symbolic link to "/run/shm". The "/run" tmpfs was mounted above so in this case only a directory needs to be created.
if [[ -n "$HEIWA" ]]; then
    mount -Rv /dev        "${HEIWA}/dev"     && \
    mount -Rv /dev/pts    "${HEIWA}/dev/pts" && \
    mount -vt proc  proc  "${HEIWA}/proc"    && \
    mount -vt sysfs sysfs "${HEIWA}/sys"     && \
    mount -vt tmpfs tmpfs "${HEIWA}/run"     && \
    mount -vt tmpfs tmpfs "${HEIWA}/tmp"     && \
    if [[ -h "${HEIWA}/dev/shm" ]]; then
        mkdir -pv "${HEIWA}/$(readlink ${HEIWA}/dev/shm)"
    fi
fi
```

### `2` - Entering the Chroot Environment
> Now that all the packages which are required to build the rest of the needed tools are on the system.
```bash
# Term variable is set to `xterm` for better compability, instead of ${TERM} that will broken if using `rxvt-unicode`.
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
ln -sv /clang1-tools/usr/bin/{install,dd}               /usr/bin
ln -sv /clang1-tools/lib/libc++.so{,.1}                 /usr/lib
ln -sv /clang1-tools/lib/libc++abi.so{,.1}              /usr/lib
ln -sv /clang1-tools/lib/libunwind.so{,.1}              /usr/lib
ln -sv /clang1-tools/lib/libunwind.a                    /usr/lib
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
```

<br>

> **Compilation Instruction!**
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

> **Required!**
```bash
# Apply some persistent environment variables.
cat > ~/.bashrc << "EOF"
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
source ~/.bashrc

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

### `6` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!**
```bash
# Apply patch (from Void Linux).
# To prevent crash with a NULL pointer dereference when dcngettext() is called with NULL msgid[12] arguments.
patch -Np1 -i ../../extra/musl/patches/mo_lookup.patch

# Configure source.
LDFLAGS="-Wl,-soname,libc.musl-x86_64.so.1" \
./configure \
    --prefix=/usr        \
    --sysconfdir=/etc    \
    --localstatedir=/var \
    --disable-gcc-wrapper

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

# Install fully-featured musl ldconfig. (https://code.foxkit.us/smaeul/packages/-/commit/c631e4fc5ab64a9ad668ed5f753348ce8eae5219?view=parallel)
install -vm755 -t /sbin/ ../../extra/musl/files/ldconfig

# Install musl-legacy-compat (from Void Linux).
for B in {cdefs,queue,tree}.h; do
    install -vm644 -t /usr/include/sys/ \
    ../../extra/musl/files/musl-legacy-compat/${B}
done; install -vm644 -t /usr/include/ \
../../extra/musl/files/musl-legacy-compat/error.h
```

### `7` - Adjusting Toolchains
> **Required!**
```bash
# Re-configure Stage-1 Clang/LLVM to use "/usr".
ln -sv clang-12 /clang1-tools/bin/x86_64-heiwa-linux-musl-clang
ln -sv clang-12 /clang1-tools/bin/x86_64-heiwa-linux-musl-clang++
cat > /clang1-tools/bin/x86_64-heiwa-linux-musl.cfg << "EOF"
--sysroot=/usr -Wl,-dynamic-linker /lib/ld-musl-x86_64.so.1
EOF

CC="x86_64-heiwa-linux-musl-clang"
CXX="x86_64-heiwa-linux-musl-clang++"
export CC CXX

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

### `8` - TimeZone Utilities (tzdata)
> **Required!**
```bash
# Create a directory and decompress needed tarball.
mkdir -v tzdb && pushd tzdb && \
    tar xzf ../tzdata2021a.tar.gz && \
    tar xzf ../tzcode2021a.tar.gz && \
    tar xf  ../posixtz-0.5.tar.xz

# Apply patch to fix `lseek`.
patch -Np1 -i ../../extra/tzdata/patches/0001-posixtz-fix-up-lseek.patch

export timezones="africa antarctica asia australasia europe northamerica
southamerica etcetera backward factory"

# Build.
time {
    make CFLAGS="$CFLAGS -DHAVE_STDINT_H=1" TZDIR="/usr/share/zoneinfo" && \
    make -C posixtz-0.5 posixtz
}

# Install timezone data.
# Ignore all warnings.
./zic -y ./yearistype -d /usr/share/zoneinfo ${timezones}
./zic -y ./yearistype -d /usr/share/zoneinfo/right -L leapseconds ${timezones}
./zic -y ./yearistype -d /usr/share/zoneinfo -p America/New_York
install -vm444 -t /usr/share/zoneinfo/ iso3166.tab zone1970.tab zone.tab
install -vm755 -t /usr/bin/ tzselect
install -vm755 -t /usr/sbin/ zic zdump
install -vm644 -t /usr/share/man/man8/ zic.8 zdump.8
install -vm755 -t /usr/bin/ posixtz-0.5/posixtz 

unset timezones

# Use `tzselect` to determine <xxx>.
tzselect

cp -v /usr/share/zoneinfo/<xxx> /etc/localtime

# Back to "/sources/pkgs" directory.
popd
```
