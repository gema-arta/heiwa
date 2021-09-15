## `I` Preparation

> #### Announcement
> * Currently only focus on **x86_64** architecture, build with native CPU optimization and ThinLTO for the final toolchain. [Triplet at OSDev](https://wiki.osdev.org/Target_Triplet).

> #### Host System Requirements
> ```sh
> sh <(curl -s "https://raw.githubusercontent.com/heiwalinux/heiwa/main/version-check")
> ```
> <details>
> <summary>Screenshot of Gentoo/Linux host</summary>
> 
> <br>
> <p align="center"><img src="https://i.imgur.com/ZRNPehJ.png" alt=""/></p>
> 
> </details>

> #### * Beginning of as root!
### `1` - Prepare a volume/partition
> #### Formatting

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

> An example for **EXT4** on **Linux-5.13.x** or newer.
```bash
mount -vo noatime,discard /dev/sdxY "$HEIWA"
```
> An example for **F2FS** on **Linux-5.13.x** or newer.
```bash
mount -vo noatime,gc_merge,compress_algorithm=lz4,compress_extension='*',compress_chksum,atgc /dev/sdxY "$HEIWA"
```

### `2` - Creating sources and toolchains directories
> Create directories to build Clang/LLVM with GCC and the final toolchain without GCC libraries. As root, link them to host's root directory.

> The "/clang1-tools" should use "/usr" merge with relative paths. **Why?** Because we implement it.
```bash
if [[ -d "$HEIWA" ]]; then
    if mkdir -pv ${HEIWA}/clang{0,1}-tools; then
        ln -sfv ${HEIWA}/clang0-tools /
        ln -sfv ${HEIWA}/clang1-tools /
        ln -sfv ./bin /clang1-tools/sbin
        if mkdir -v /clang1-tools/usr; then
            ln -sfv ../bin /clang1-tools/usr/bin
            ln -sfv ../sbin /clang1-tools/usr/sbin
        fi
    fi
    mkdir -pv ${HEIWA}/sources/{extra,pkgs}
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
if [[ -d "${HEIWA}/sources" ]]; then
    chmod -vR a+wt ${HEIWA}/sources
    chown -Rv heiwa ${HEIWA}/sources
    chown -Rv heiwa {${HEIWA},}/clang{0,1}-tools
fi
```
> #### Setup default process priorites

> This is an optional section to make privileged user use **19** as default user-level priority through linux-PAM. Don't use **RT** priorities! It's bad.
> > The below rules can slow down the system, especially GUI if run.
```bash
if ! grep -qo 'heiwa.*priority' /etc/security/limits.conf; then
    echo "heiwa            -       priority        -1" >> \
    /etc/security/limits.conf
fi
```
> #### * End of as root!

> #### * Beginning of as privileged user!
### `4` - Setup privileged user's environment
> First, login as privileged user in place of the current shell PID.
```bash
exec su - heiwa
```
> Then, setup the bash environment variables.
```bash
cat > ~/.bash_profile << EOF
exec env -i HOME="\$HOME" TERM="\$TERM" \
COMMON_FLAGS="-g0 -march=native -Os -pipe" $(command -v bash)
EOF
```
```bash
cat > ~/.bashrc << EOF
set +h
umask 022
unalias -a
HEIWA="${HEIWA:-/media/Heiwa}"
LC_ALL="POSIX"
PATH="/clang0-tools/bin:/clang1-tools/bin:/usr/bin:/bin"
LLVM_SRC="\${HEIWA}/sources/llvm"
export HEIWA LC_ALL PATH LLVM_SRC
EOF
source ~/.bash_profile
```
```bash
C_TRIPLET="$(sed 's|-[^-]*|-cross|' <<<  "$MACHTYPE")" # Host cross-triplet, to bootstrap cross-libc GCC.
T_ARCH="$(cut -d- -f1 <<< "$MACHTYPE")"                # Target CPU architecture, use native host's arch.
C_ARCH="$(cut -d_ -f1 <<< "$T_ARCH")"                  # CPU arch, to be used to build Linux API headers.
C_CPU="$(sed 's|_|-|' <<< "$T_ARCH")"                  # CPU type, to be used to build static GCC.
L_TARGET="X86"                                         # LLVM-specific architecture build target.
T_TRIPLET="${T_ARCH}-pc-linux-musl"                    # Target triplet for final toolchain.
H_TRIPLET="$(sed 's|-[^-]*|-heiwa|' <<< "$T_TRIPLET")" # Target triplet for cross-libraries.
```
> Let's check if the above variables are all correct.
```bash
printf '%b\n' $C_{ARCH,CPU,TRIPLET} $L_TARGET $T_{ARCH,TRIPLET} $H_TRIPLET
```
```bash
# | On the glibc host, the output should be:
# |------------------------------------------
# |x86
# |x86-64
# |x86_64-cross-linux-gnu
# |X86
# |x86_64
# |x86_64-pc-linux-musl
# |x86_64-heiwa-linux-musl
```
> If you want multitasking responsiveness when using multiple jobs, set the load average to prevent slowdown system e.g core/threads + 2.
```bash
cat >> ~/.bashrc << EOF
C_TRIPLET="${C_TRIPLET}"
T_ARCH="${T_ARCH}"
C_ARCH="${C_ARCH}"
C_CPU="${C_CPU}"
L_TARGET="${L_TARGET}"
T_TRIPLET="${T_TRIPLET}"
H_TRIPLET="${H_TRIPLET}"
export C_TRIPLET T_ARCH C_ARCH C_CPU L_TARGET T_TRIPLET H_TRIPLET
CFLAGS="\${COMMON_FLAGS}"
CXXFLAGS="\${COMMON_FLAGS}"
LDFLAGS="-Wl,-O2 -Wl,--as-needed"
MAKEFLAGS="-j\$(nproc) -l\$((\$(nproc)+2))"
export CFLAGS CXXFLAGS LDFLAGS MAKEFLAGS
EOF
source ~/.bashrc
```

> #### After Preparation
> Copy "[syscore/*](./../../syscore/)" to "${HEIWA}/sources/extra/" now!

<h2></h2>

Continue to [Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain](./2-Stage0_Clang_LLVM.md).
