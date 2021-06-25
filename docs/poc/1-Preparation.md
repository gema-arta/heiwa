## `I` Preparation
> #### * Beginning of as root!
### `1` - Prepare the volume/partition
```sh
# Formatting.
mkfs.ext4 -m 0 -L "Heiwa_Linux" /dev/sdaX

# Export mountpoint variable and create the directory if not exists.
export HEIWA="/media/Heiwa"
mkdir -pv ${HEIWA}

# Mount the target volume/partition.
mount -vo noatime,discard /dev/sdaX ${HEIWA}
```

### `2` - Creating packages and toolchain directory
```sh
# Create directories to build clang with GCC libraries and the final toolchain without GCC libraries.
# As root, Link them to host's root directory.
if [ ! -z $HEIWA ]; then
    mkdir -pv ${HEIWA}/{clang0,llvm}-tools
    ln -sv ${HEIWA}/clang0-tools /
    ln -sv ${HEIWA}/llvm-tools /
    mkdir -pv ${HEIWA}/sources/{patches,files,pkgs}
fi
```

### `3` - Adding privileged user
```sh
# Setup privileged user.
groupadd heiwa
useradd -s /bin/bash -g heiwa -m -k /dev/null heiwa
passwd heiwa

# Setting up directory permission.
# Warning! This is danger, so check its variables before chown.
# echo {${HEIWA},}/{clang0,llvm}-tools
if [ ! -z $HEIWA ]; then
    chmod -vR a+wt ${HEIWA}/sources
    chown -Rv heiwa ${HEIWA}/sources
    chown -Rv heiwa {${HEIWA},}/{clang0,llvm}-tools
fi
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
```sh
# Login as privileged user.
su - heiwa

cat > ~/.bash_profile << "EOF"
export COMMON_FLAGS="-march=native -Os -pipe"
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
PATH="/cross-tools/bin:/cross-tools/usr/bin:/clang0-tools/bin:/clang0-tools/usr/bin:/llvm-tools/bin:/llvm-tools/usr/bin:/bin:/usr/bin"
export HEIWA LC_ALL PATH
# CFLAGS and CXXFLAGS must not be set during the building of cross-tools.
unset CFLAGS CXXFLAGS
# Build PATH.
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
EOF
source ~/.bashrc
```

<br>

Continue to [2-Clang_Stage0.md](./2-Clang_Stage0.md).
