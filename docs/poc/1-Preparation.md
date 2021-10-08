## `I` Preparation

> #### Announcement
> * Currently only built pure **x86_64** Arch/ABI with native CPU-EXTENSION and use ThinLTO started from Stage-1. [Triplet at OSDev](https://wiki.osdev.org/Target_Triplet).

> #### Host System Requirements
> ```sh
> sh <(curl -s "https://raw.githubusercontent.com/heiwalinux/heiwa/main/version-check")
> ```
> <details>
> <summary>Screenshot of Gentoo/Linux host</summary>
> 
> <br>
> <p align="center"><img src="https://i.imgur.com/PlotBPA.png" alt=""/></p>
> 
> </details>

> #### * Beginning of as root!
### `1` - Prepare a volume/partition
> #### Choose preferred filesystem

> **EXT4.** Recommended for HDDs.
```bash
mkfs.ext4 -m 0 -L "Heiwa.Linux" /dev/sdxY
```
> **F2FS.** Recommended for SSDs.
```bash
mkfs.f2fs -l "Heiwa.Linux" -O extra_attr,inode_checksum,sb_checksum,compression,encrypt /dev/sdxY
```
> Then, export the mount point variable and create the directory if not exist. **Why "/media"?** It's easily detected with GVFS via D-Bus.
```bash
export HEIWA="/media/Heiwa"
mkdir -pv "$HEIWA"
```
> Next, mount the target volume/partition.

> An example for **EXT4** on **Linux-5.14.x** or newer.
```bash
mount -vo noatime,discard /dev/sdxY "$HEIWA"
```
> An example for **F2FS** on **Linux-5.14.x** or newer.
```bash
mount -vo noatime,gc_merge,compress_algorithm=lz4,compress_extension='*',compress_chksum,compress_cache,atgc /dev/sdxY "$HEIWA"
```

### `2` - Creating sources and toolchains directories
> Create directories to build Clang/LLVM with GCC and the final toolchain without GCC libraries. As root, link them to host's root directory.
```bash
if [[ -d "$HEIWA" ]]; then
    if mkdir -pv "$HEIWA"/clang{1,2}-tools; then
        ln  -sfv "$HEIWA"/clang1-tools /
        ln  -sfv "$HEIWA"/clang2-tools /
        ln  -sfv lib /clang1-tools/lib64
    fi
    mkdir -pv "$HEIWA"/sources/{ccache,syscore,pkgs}
fi
```

### `3` - Adding privileged user
> #### Setup privileged user

> **Why?** When logged in as user root, making a single mistake can damage or destroy a system. That's it.
```bash
groupadd heiwa
useradd -s $(command -v bash) -g heiwa -m -k /dev/null heiwa
passwd heiwa
```
> #### Setup directory permissions

```bash
if [[ -d "${HEIWA}/sources" && -d "${HEIWA}/clang1-tools" && -d "${HEIWA}/clang2-tools" ]]; then
    chmod -Rv  a+wt  "$HEIWA"/sources
    chown -hRv heiwa "$HEIWA"/sources
    chown -hRv heiwa {"$HEIWA",}/clang{1,2}-tools
fi
```
> #### Setup default process priorites

> This is an optional section to make privileged user use **22** as default user-level priority through linux-PAM. Don't use **RT** priorities! It's bad.
> > The reason why doing this is to anticipate system freezes in certain situations, especially building Clang/LLVM on low-end devices.
```bash
if ! grep -qo 'heiwa.*priority' /etc/security/limits.conf; then
cat >> /etc/security/limits.conf << "EOF"
heiwa            -       priority        2
EOF
fi
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
> First, login as privileged user in place of the current shell PID.
```bash
exec su - heiwa
```
> Then, setup the toolchain environment variables.
```bash
HST_TRIPLET="$(sed 's|-[^-]*|-cross|' <(cc -dumpmachine))"
case $(uname -m) in
    x86_64) GCC_MCPU="x86-64"
            TGT_LLVM="X86"
            TGT_ARCH="x86_64"
            TGT_TRIPLET="${TGT_ARCH}-pc-linux-musl"
            HEI_TRIPLET="${TGT_ARCH}-heiwa-linux-musl"
            ADDON_FLAGS="-march=native"
    ;;
    *)      echo 'Any architecture other than x86_64 currently not implemented yet.'
    ;;
esac
```
> Let's check if the above environment variables are all correct.
```bash
printf '%s\n' $HST_TRIPLET $GCC_MCPU $TGT_{LLVM,ARCH,TRIPLET} $HEI_TRIPLET $ADDON_FLAGS
```
```bash
# | On the x86_64 glibc host, the output should be:
# |------------------------------------------------
# |x86_64-cross-linux-gnu
# |x86-64
# |X86
# |x86_64
# |x86_64-pc-linux-musl
# |x86_64-heiwa-linux-musl
# |-march=native
```
> Now apply the above environment variables into bash startup profile.
```bash
cat > ~/.bash_profile << EOF
exec env -i HOME="\$HOME" TERM="\$TERM" HOST_PATH="\$PATH" \
COMMON_FLAGS=" $ADDON_FLAGS -O2 -pipe -w -g0 " $(command -v bash)
EOF
cat > ~/.bashrc << EOF
set +h
umask 022
unalias -a
LC_ALL="POSIX"
HEIWA="${HEIWA:-/media/Heiwa}"
PATH="/clang2-tools/bin:/clang1-tools/bin:\${HOST_PATH}"
LLVM_SRC="\${HEIWA}/sources/llvm"
CCACHE_DIR="\${HEIWA}/sources/ccache"
export LC_ALL HEIWA PATH LLVM_SRC CCACHE_DIR
HST_TRIPLET="$HST_TRIPLET"
GCC_MCPU="$GCC_MCPU"
TGT_LLVM="$TGT_LLVM"
TGT_ARCH="$TGT_ARCH"
TGT_TRIPLET="\${TGT_ARCH}-pc-linux-musl"
HEI_TRIPLET="\${TGT_ARCH}-heiwa-linux-musl"
export HST_TRIPLET GCC_MCPU TGT_LLVM TGT_ARCH TGT_TRIPLET HEI_TRIPLET
CFLAGS="\${COMMON_FLAGS}"
CXXFLAGS="\${COMMON_FLAGS}"
CPPFLAGS="-DNDEBUG"
LDFLAGS="-Wl,-O2 -Wl,--as-needed"
JOBS="\$(nproc)"
MAKEFLAGS="-j\${JOBS} -l\$((\${JOBS}+2))"
export CFLAGS CXXFLAGS CPPFLAGS LDFLAGS JOBS MAKEFLAGS
EOF
source ~/.bash_profile
```
> If you want multitasking responsiveness when using multiple jobs, set the load average to prevent system overload, e.g core/threads + 2.

> #### After Preparation
> Copy "[syscore/*](./../../syscore/)" to "${HEIWA}/sources/syscore/" now!

<h2></h2>

Continue to [Stage-0 Clang/LLVM (feat. GNU) Cross-Toolchain](./2-Stage0_Clang_LLVM.md).
