## `I` Preparation

> **Notes!**
> Copy [`syscore/*`](./../../syscore/) to `${HEIWA}/sources/extra/.`

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

### `2` - Creating sources and toolchain directory
```bash
# Create directories to build clang with GCC libraries and the final toolchain without GCC libraries.
# As root, link them to host's root directory.
if [[ -n "$HEIWA" ]]; then
    mkdir -pv "${HEIWA}/"clang{0,1}-tools
    ln -sv "${HEIWA}/clang0-tools" /
    ln -sv "${HEIWA}/clang1-tools" /
    mkdir -pv "${HEIWA}/sources/"{extra,pkgs}
fi
```

### `3` - Adding privileged user
```bash
# Setup privileged user.
groupadd heiwa
useradd -s /bin/bash -g heiwa -m -k /dev/null heiwa
passwd heiwa

# Setting up directory permission.
# Warning! This is danger, so check its variables before `chown`.
# echo {"$HEIWA",}/clang{0,1}-tools
if [[ -n "$HEIWA" ]]; then
    chmod -vR a+wt "${HEIWA}/sources"
    chown -Rv heiwa "${HEIWA}/sources"
    chown -Rv heiwa {"$HEIWA",}/clang{0,1}-tools
fi
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
```bash
# Login as privileged user.
su - heiwa

cat > ~/.bash_profile << "EOF"
export COMMON_FLAGS="-march=native -Oz -pipe"
exec env -i HOME="$HOME" TERM="$TERM" PS1='\u:\w\n\$ ' \
COMMON_FLAGS="${COMMON_FLAGS}" CFLAGS="${COMMON_FLAGS}" \
CXXFLAGS="${COMMON_FLAGS}" /bin/bash
EOF

export HEIWA="/media/Heiwa"
cat > ~/.bashrc << EOF
set +h
umask 022
HEIWA="${HEIWA}"
LC_ALL="POSIX"
PATH="/clang0-tools/bin:/clang0-tools/usr/bin:/clang1-tools/bin:/clang1-tools/usr/bin:/bin:/usr/bin"
export HEIWA LC_ALL PATH
# CFLAGS and CXXFLAGS must not be set during the building of Stage-0 Clang/LLVM.
unset CFLAGS CXXFLAGS
export LLVM_SRC="\${HEIWA}/sources/llvm"
EOF
source ~/.bash_profile

export HEIWA_TARGET="x86_64-heiwa-linux-musl"
export TARGET_TRUPLE="x86_64-pc-linux-musl"
export HEIWA_ARCH="x86"
export HEIWA_CPU="x86-64"
export HEIWA_HOST="$(echo "$MACHTYPE" | \
    sed "s/$(echo "$MACHTYPE" | cut -d- -f2)/cross/")"

cat >> ~/.bashrc << EOF
export HEIWA_HOST="${HEIWA_HOST}"
export HEIWA_TARGET="${HEIWA_TARGET}"
export HEIWA_ARCH="${HEIWA_ARCH}"
export HEIWA_CPU="${HEIWA_CPU}"
export TARGET_TRUPLE="${TARGET_TRUPLE}"
# Make's multiple jobs based on CPU core/threads.
alias make="make -j\$(nproc) -l\$(nproc)"
EOF
source ~/.bashrc
```

<h2></h2>

Continue to [Stage-0 Clang/LLVM (ft. GNU) Cross-Compile Toolchains](./2-Stage0_Clang_LLVM.md).
