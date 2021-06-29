## `IV` Final System
> **Compilation Instruction!**
> ```bash
> heiwa@localh3art /media/Heiwa/sources/pkgs $ tar xf target-package.tar.xz
> heiwa@localh3art /media/Heiwa/sources/pkgs $ tar xzf target-package.tar.gz
> heiwa@localh3art /media/Heiwa/sources/pkgs $ pushd target-package
> 
> < compilation process >
> 
> heiwa@localh3art /media/Heiwa/sources/pkgs/target-package $ popd
> heiwa@localh3art /media/Heiwa/sources/pkgs $ rm -rf target-package
> ```

> #### * Beginning of as root!
### `1` - Preparing Virtual Kernel File Systems
> Various file systems exported by the kernel are used to communicate to and from the kernel itself. These file systems are virtual in that no disk space is used for them. The content of the file systems resides in memory.
```bash
# When the kernel boots the system, it requires the presence of a few device nodes, in particular the console and null devices.
# The device nodes must be created on the hard disk so that they are available before the kernel populates /dev), and additionally when Linux is started with init=/bin/bash.
# Create the directories and initial device nodes.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv "${HEIWA}/"{dev,proc,sys,run}   && \
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
```sh
# Term variable is set to `xterm` for better compability, instead of ${TERM} that will broken if using `rxvt-unicode`.
[[ -n "$HEIWA" ]] && \
chroot "$HEIWA" /clang1-tools/usr/bin/env -i                                     \
    HOME="/root" TERM="xterm" PS1='(heiwa chroot) \u: \w \$ '                    \
    PATH="/bin:/usr/bin:/sbin:/usr/sbin:/clang1-tools/bin:/clang1-tools/usr/bin" \
    TRUPLE="x86_64-pc-linux-musl" CC="${TRUPLE}-clang" CXX="${TRUPLE}-clang++"   \
    /clang1-tools/bin/bash --login +h

# The `-i` option given to the env command will clear all variables of the chroot environment.
# Note! that the bash prompt will say I have no name! This is normal because the "/etc/passwd" file has not been created yet.
```
