## `I` Preparation

> #### Announcement
> * Currently only focus on **x86_64** architecture, build with native CPU optimization and ThinLTO for Stage-1 Clang/LLVM toolchain.  

> #### * Beginning of as root!
### `1` - Prepare a volume/partition
```bash
# Formatting.
mkfs.ext4 -m 0 -L "Heiwa_Linux" /dev/sdaX

# Export mountpoint variable and create the directory if not exist.
export HEIWA="/media/Heiwa"
mkdir -pv "$HEIWA"

# Mount the target volume/partition.
mount -vo noatime,discard /dev/sdaX "$HEIWA"
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
# Setting up directory permission.
# Warning! This is danger, so check its variables before `chown`.
# echo {${HEIWA},}/clang{0,1}-tools
if [[ -n "$HEIWA" ]]; then
    chmod -vR a+wt ${HEIWA}/sources
    chown -Rv heiwa ${HEIWA}/sources
    chown -Rv heiwa {${HEIWA},}/clang{0,1}-tools
fi
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
> **Warning!** Always set multiple jobs with load average to prevent hangs nor system freeze.
```bash
# Login as privileged user.
su - heiwa

cat > ~/.bash_profile << "EOF"
exec env -i HOME="$HOME" TERM="$TERM" \
COMMON_FLAGS="-march=native -Oz -pipe" /bin/bash
EOF

cat > ~/.bashrc << EOF
set +h
umask 022
unalias grep 2>/dev/null
HEIWA="${HEIWA:-/media/Heiwa}"
LC_ALL="POSIX"
PATH="/clang0-tools/usr/bin:/clang0-tools/bin:/clang1-tools/usr/bin:/clang1-tools/bin:/usr/bin:/bin"
LLVM_SRC="\${HEIWA}/sources/llvm"
export HEIWA LC_ALL PATH LLVM_SRC
EOF
source ~/.bash_profile

export C_TRIPLET="$(echo "$MACHTYPE" | \
    sed "s|$(echo "$MACHTYPE" | cut -d- -f2)|cross|")"
export C_ARCH="x86"
export C_CPU="x86-64"
export L_TARGET="X86"
export T_TRIPLET="x86_64-pc-linux-musl"
export H_TRIPLET="$(echo "$T_TRIPLET" | \
    sed "s|$(echo "$T_TRIPLET" | cut -d- -f2)|heiwa|")"

cat >> ~/.bashrc << EOF

C_TRIPLET="${C_TRIPLET}"
C_ARCH="${C_ARCH}"
C_CPU="${C_CPU}"
L_TARGET="${L_TARGET}"
T_TRIPLET="${T_TRIPLET}"
H_TRIPLET="${H_TRIPLET}"
MAKEFLAGS="-j\$(nproc) -l\$(($(nproc)+1))"
export C_TRIPLET C_ARCH C_CPU L_TARGET T_TRIPLET H_TRIPLET MAKEFLAGS
EOF
source ~/.bashrc
```

> #### After Preparation
> Copy [`syscore/*`](./../../syscore/) to `${HEIWA}/sources/extra/.` now!

<h2></h2>

Continue to [Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain](./2-Stage0_Clang_LLVM.md).
