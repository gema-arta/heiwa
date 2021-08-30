## `I` Preparation

> #### Announcement
> * Currently only focus on **x86_64** architecture, build with native CPU optimization and ThinLTO for the final toolchain. [Triplet at OSDev](https://wiki.osdev.org/Target_Triplet).

> #### Host System Requirements
> ```sh
> sh <(curl -s "https://raw.githubusercontent.com/heiwalinux/heiwa/main/version-check")
> ```

> #### * Beginning of as root!
### `1` - Prepare a volume/partition
```bash
# Formatting. EXT4 recommended for HDDs.
mkfs.ext4 -m 0 -L "Heiwa.Linux" /dev/sdxY

# Formatting. F2FS recommended for SSDs.
mkfs.f2fs -l "Heiwa.Linux" -O extra_attr,inode_checksum,sb_checksum,compression,encrypt /dev/sdxY
```
```bash
# Export the mount point variable and create the directory if not exist.
export HEIWA="/media/Heiwa"
mkdir -pv "$HEIWA"
```
```bash
# Mount the target volume/partition. EXT4 example on Linux-5.13.x or newer.
mount -vo noatime,discard /dev/sdxY "$HEIWA"

# Mount the target volume/partition. F2FS example on Linux-5.13.x or newer.
mount -vo noatime,gc_merge,compress_algorithm=lz4,compress_extension='*',compress_chksum,atgc /dev/sdxY "$HEIWA"
```

### `2` - Creating sources and toolchains directories
```bash
# Create directories to build Clang/LLVM with GCC libraries and the final toolchain without GCC libraries.
# As root, link them to host's root directory.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv ${HEIWA}/clang{0,1}-tools
    ln -sv ${HEIWA}/clang0-tools /
    ln -sv ${HEIWA}/clang1-tools /
    mkdir -pv ${HEIWA}/sources/{extra,pkgs}
fi
```

### `3` - Adding privileged user
```bash
# Setup privileged user.
groupadd heiwa
useradd -s /bin/bash -g heiwa -m -k /dev/null heiwa
passwd heiwa
```
```bash
# Setup directory permissions.
# Warning! This is danger, so check the variables before `chown`.
# echo {${HEIWA},}/clang{0,1}-tools
if [[ -n "$HEIWA" ]]; then
    chmod -vR a+wt ${HEIWA}/sources
    chown -Rv heiwa ${HEIWA}/sources
    chown -Rv heiwa {${HEIWA},}/clang{0,1}-tools
fi
```
```bash
# This is an optional section to make privileged user use 19 as default user level priority using linux-PAM.
# Ref: https://github.com/owl4ce/hmg/blob/main/etc/security/limits.conf#L65
cat >> /etc/security/limits.conf << "EOF"
heiwa            -       priority        -1
EOF
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
```bash
# Login as privileged user.
exec su - heiwa
```
```bash
cat > ~/.bash_profile << "EOF"
exec env -i HOME="$HOME" TERM="$TERM" \
COMMON_FLAGS="-march=native -Os -pipe" /bin/bash
EOF
```
```bash
cat > ~/.bashrc << EOF
set +h
umask 022
unalias -a
HEIWA="${HEIWA:-/media/Heiwa}"
LC_ALL="POSIX"
PATH="/clang0-tools/usr/bin:/clang0-tools/bin:/clang1-tools/usr/bin:/clang1-tools/bin:/usr/bin:/bin"
LLVM_SRC="\${HEIWA}/sources/llvm"
export HEIWA LC_ALL PATH LLVM_SRC
EOF
source ~/.bash_profile
```
```bash
C_TRIPLET="$(echo "$MACHTYPE" | \
    sed "s|$(echo "$MACHTYPE" | cut -d- -f2)|cross|")"
C_ARCH="x86"
C_CPU="x86-64"
L_TARGET="X86"
T_TRIPLET="x86_64-pc-linux-musl"
H_TRIPLET="$(echo "$T_TRIPLET" | \
    sed "s|$(echo "$T_TRIPLET" | cut -d- -f2)|heiwa|")"
```
```bash
# If you want multitasking responsiveness when using multiple jobs, set the load average to prevent slowdowned system (maybe OOM).
cat >> ~/.bashrc << EOF
C_TRIPLET="${C_TRIPLET}"
C_ARCH="${C_ARCH}"
C_CPU="${C_CPU}"
L_TARGET="${L_TARGET}"
T_TRIPLET="${T_TRIPLET}"
H_TRIPLET="${H_TRIPLET}"
export C_TRIPLET C_ARCH C_CPU L_TARGET T_TRIPLET H_TRIPLET
CFLAGS="\${COMMON_FLAGS}"
CXXFLAGS="\${COMMON_FLAGS}"
LDFLAGS="-Wl,-O2 -Wl,--as-needed"
MAKEFLAGS="-j\$(nproc) -l\$(nproc)"
export CFLAGS CXXFLAGS LDFLAGS MAKEFLAGS
EOF
source ~/.bashrc
```

> #### After Preparation
> Copy "[syscore/*](./../../syscore/)" to "${HEIWA}/sources/extra/" now!

<h2></h2>

Continue to [Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain](./2-Stage0_Clang_LLVM.md).
