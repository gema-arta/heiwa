## `III` Stage-1 Clang/LLVM Toolchain
The purpose of this stage is to build stage 2 Clang/LLVM self-hosted toolchain and needed tools used to build the "Final System" afterwards.

> #### Compilation Instruction!
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

> #### Tracking Installed Files by Example
> Use the staged directory to browse where the files are installed, then reinstall into correct locations. Below are three examples.
> ```bash
> < extracting and enter the directory >
> 
> < compilation process >
> 
> heiwa@...pkgs/target-package $ make DESTDIR="$(pwd)/work" install
> heiwa@...pkgs/target-package $ make PREFIX="$(pwd)/work/clang2-tools" install
> heiwa@...pkgs/target-package/build $ cmake -DCMAKE_INSTALL_PREFIX="$(pwd)/work/clang2-tools" -P cmake_install.cmake 
>
> < exiting and cleaning up directory >
> ```

### `1` - Setup Clang/LLVM Environment Variables
> Apply toolchain environment, but don't set the compiler to heiwa triplet of Stage-0 Clang/LLVM in order to build musl libc first at this stage.
```bash
cat >> ~/.bashrc << "EOF"
CC="clang"
CXX="clang++"
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
EOF
source ~/.bashrc
```

### `2` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
> > **Build time:** <2m
```bash
# Configure source.
./configure --prefix=/              \
            --with-malloc=mallocng  \
            --enable-optimize=speed \
            --disable-gcc-wrapper   \
            --disable-static
```
```bash
# Build.
time { make; }
```
```bash
# Install and fix wrong shared object symlink.
time {
    make DESTDIR=/clang2-tools install
    ln -sfv libc.so /clang2-tools/lib/ld-musl-${TGT_ARCH}.so.1
}
```
```bash
# Optionally, create a `ldd` symlink to use to print shared object dependencies.
mkdir -v /clang2-tools/bin && \
ln -sfv ../lib/libc.so /clang2-tools/bin/ldd
```
```bash
# Configure path for the dynamic linker.
mkdir -v /clang2-tools/etc && \
cat > /clang2-tools/etc/ld-musl-${TGT_ARCH}.path << "EOF"
/clang2-tools/lib
EOF
```
```bash
# Now set the compiler to heiwa triplet from Stage-0 Clang/LLVM to use current musl libc built.
sed -e "/CXX=/s/${CXX}/${HEI_TRIPLET}-clang++/" \
    -e "/CC=/s/${CC}/${HEI_TRIPLET}-clang/" -i ~/.bashrc
source ~/.bashrc
```
```bash
# Sanity check for heiwa triplet of Stage-0 Clang/LLVM.
${CC} -x c++ - ${CFLAGS} ${LDFLAGS} -Wl,--verbose -v <<< 'int main(){}' &> dummy.log
${READELF} -l a.out | grep --color=auto "Req.*ter"
```
```bash
# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang2-tools/lib/ld-musl-x86_64.so.1]
```
```bash
# Ensure `ld.lld` invokes C runtime objects from prior musl libc built.
# [ https://wiki.osdev.org/Creating_a_C_Library#Program_Initialization ]
grep --color=auto "ld.lld:.*crt[1in].o" dummy.log
```
```bash
# | The output should be:
# |-----------------------
# |ld.lld: /clang1-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../Scrt1.o
# |ld.lld: /clang1-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../crti.o
# |ld.lld: /clang1-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../crtn.o
```

### `3` - Linux API Headers
> #### `5.14.x` or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
> > **Build time:** <20s
```bash
# The recommended make target `headers_install` cannot be used, because it requires `rsync` which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
```
```bash
# Ensure there are no stale files embedded in the package. Then build.
time {
    make LLVM=1 LLVM_IAS=1 mrproper && \
    make ARCH=${TGT_ARCH} LLVM=1 LLVM_IAS=1 HOSTCC=${CC} headers
}
```
```bash
# Remove unnecessary dotfiles and Makefile.
find usr/include \( -name '.*' -o -name 'Makefile' \) -exec rm -fv {} \;
```
```bash
# Install.
cp -rfv usr/include /clang2-tools/.
```

### `4` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** By Pigz at current stage and optionally enabled for Stage-1 Clang/LLVM builds.
> > **Build time:** <25s
```bash
# Configure source. Use optimization level 3.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev    \
    -DCMAKE_INSTALL_PREFIX="/clang2-tools" \
    -DCMAKE_C_FLAGS="-flto=thin $CFLAGS"   \
    -DBUILD_SHARED_LIBS=ON                 \
    -DZLIB_COMPAT=ON
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install.
time { make -C build install; }
```

### `5` - NetBSD curses
> #### `0.3.2` or newer
> The NetBSD curses package contains libraries for terminal-independent handling of character screens.

> **Required!** For the most programs that depends on `-ltinfo` or `-lterminfo` dynamic linker flags, including Stage-1 Clang/LLVM builds.
> > **Build time:** ~30s
```bash
# Build.
time { make CFLAGS="-flto=thin $CFLAGS" all-dynamic; }
```
```bash
# Install and create symlinks as `libtinfo` libraries (which actually replace GNU ncurses).
time {
    make PREFIX=/clang2-tools install-dynamic
    ln -sfv libterminfo.so /clang2-tools/lib/libtinfo.so
    ln -sfv libterminfo.so /clang2-tools/lib/libtinfow.so
}
```

### `6` - Clang/LLVM <kbd>stage 2</kbd>
> #### `12.x.x` or newer
> - C language family frontend for LLVM;  
> - C++ runtime stack unwinder from LLVM;  
> - Low level support for a standard C++ library from LLVM;  
> - New implementation of the C++ standard library, targeting C++11 from LLVM.

> **Required!** Build Stage-1 Clang/LLVM self-hosted toolchain.
> > **Build time:** ~4h-7h
```bash
# Exit from LLVM source directory if already entered after decompressing.
popd
```
```bash
# Rename LLVM source directory to "$LLVM_SRC", then enter.
mv -fv llvm-12.0.1.src "$LLVM_SRC" && pushd "$LLVM_SRC"
```
```bash
# Decompress `clang`, `lld`, `compiler-rt`, `libunwind`, `libcxxabi`, and `libcxx` to correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/compiler-rt-12.0.1.src.tar.xz && mv -fv compiler-rt{-12.0.1.src,}
    tar xf   ../../pkgs/libunwind-12.0.1.src.tar.xz && mv -fv libunwind{-12.0.1.src,}
    tar xf   ../../pkgs/libcxxabi-12.0.1.src.tar.xz && mv -fv libcxxabi{-12.0.1.src,}
    tar xf      ../../pkgs/libcxx-12.0.1.src.tar.xz && mv -fv libcxx{-12.0.1.src,}
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.1.src.tar.xz && mv -fv clang{-12.0.1.src,}
    tar xf   ../../pkgs/lld-12.0.1.src.tar.xz && mv -fv lld{-12.0.1.src,}
popd
```
```bash
# Apply patches (from Void Linux) and update config.guess for better platform detection.
../extra/llvm/patches/appatch
```
```bash
# Configure `libunwind` source. Use optimization level 3.
pushd ${LLVM_SRC}/projects/libunwind/ &&                \
cmake -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev      \
               -DCMAKE_INSTALL_PREFIX="/clang2-tools"   \
               -DCMAKE_C_FLAGS="-flto=thin $CFLAGS"     \
               -DCMAKE_CXX_FLAGS="-flto=thin $CXXFLAGS" \
               -DLLVM_PATH="$LLVM_SRC"                  \
               -DLIBUNWIND_ENABLE_ASSERTIONS=OFF        \
               -DLIBUNWIND_ENABLE_STATIC=OFF            \
               -DLIBUNWIND_USE_COMPILER_RT=ON
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install both libraries and headers.
time {
    make -C build install && \
    install -vm644 -t /clang2-tools/include/ include/*.h && \
    popd
}
```
```bash
# Configure `libcxxabi` source. Use optimization level 3.
pushd ${LLVM_SRC}/projects/libcxxabi/ &&                \
cmake -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev      \
               -DCMAKE_INSTALL_PREFIX="/clang2-tools"   \
               -DCMAKE_CXX_FLAGS="-flto=thin $CXXFLAGS" \
               -DLLVM_PATH="$LLVM_SRC"                  \
               -DLIBCXXABI_ENABLE_ASSERTIONS=OFF        \
               -DLIBCXXABI_ENABLE_STATIC=OFF            \
               -DLIBCXXABI_USE_COMPILER_RT=ON           \
               -DLIBCXXABI_USE_LLVM_UNWINDER=ON         \
               -DLIBCXXABI_LIBCXX_INCLUDES="${LLVM_SRC}/projects/libcxx/include"
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install both libraries and headers.
time {
    make -C build install && \
    install -vm644 -t /clang2-tools/include/ include/*.h && \
    popd
}
```
```bash
# Configure `libcxx` source. Use optimization level 3.
pushd ${LLVM_SRC}/projects/libcxx/ &&                                                  \
cmake -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev                                     \
               -DCMAKE_INSTALL_PREFIX="/clang2-tools"                                  \
               -DCMAKE_CXX_FLAGS="-isystem /clang2-tools/include -flto=thin $CXXFLAGS" \
               -DLLVM_PATH="$LLVM_SRC"                                                 \
               -DLIBCXX_ENABLE_STATIC=OFF                                              \
               -DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF                                \
               -DLIBCXX_INCLUDE_BENCHMARKS=OFF                                         \
               -DLIBCXX_CXX_ABI=libcxxabi                                              \
               -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/clang2-tools/include"                  \
               -DLIBCXX_CXX_ABI_LIBRARY_PATH="/clang2-tools/lib"                       \
               -DLIBCXX_USE_COMPILER_RT=ON                                             \
               -DLIBCXX_HAS_ATOMIC_LIB=OFF                                             \
               -DLIBCXX_HAS_MUSL_LIBC=ON
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install.
time { make -C build install && popd; }
```
```bash
# Remove `libunwind`, `libcxxabi`, and `libcxx` from LLVM source directory.
rm -rf projects/lib{unwind,cxx{abi,}}
```
```bash
# Configure Clang/LLVM source. Use optimization level 3.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev         \
    -DCMAKE_INSTALL_PREFIX="/clang2-tools"      \
    -DBUILD_SHARED_LIBS=ON                      \
    -DLLVM_CCACHE_BUILD=ON                      \
    -DLLVM_APPEND_VC_REV=OFF                    \
    -DLLVM_HOST_TRIPLE="$TGT_TRIPLET"           \
    -DLLVM_DEFAULT_TARGET_TRIPLE="$TGT_TRIPLET" \
    -DLLVM_ENABLE_BINDINGS=OFF                  \
    -DLLVM_ENABLE_EH=ON                         \
    -DLLVM_ENABLE_IDE=OFF                       \
    -DLLVM_ENABLE_LIBCXX=ON                     \
    -DLLVM_ENABLE_LLD=ON                        \
    -DLLVM_ENABLE_LTO=Thin                      \
    -DLLVM_ENABLE_RTTI=ON                       \
    -DLLVM_ENABLE_BACKTRACES=OFF                \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF             \
    -DLLVM_ENABLE_WARNINGS=OFF                  \
    -DLLVM_ENABLE_LIBEDIT=OFF                   \
    -DLLVM_ENABLE_LIBXML2=OFF                   \
    -DLLVM_ENABLE_OCAMLDOC=OFF                  \
    -DLLVM_ENABLE_ZLIB=ON                       \
    -DLLVM_ENABLE_Z3_SOLVER=OFF                 \
    -DLLVM_INCLUDE_BENCHMARKS=OFF               \
    -DLLVM_INCLUDE_EXAMPLES=OFF                 \
    -DLLVM_INCLUDE_TESTS=OFF                    \
    -DLLVM_INCLUDE_GO_TESTS=OFF                 \
    -DLLVM_INCLUDE_DOCS=OFF                     \
    -DLLVM_INSTALL_BINUTILS_SYMLINKS=ON         \
    -DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON          \
    -DLLVM_INSTALL_UTILS=ON                     \
    -DLLVM_TARGET_ARCH="$TGT_LLVM"              \
    -DLLVM_TARGETS_TO_BUILD="$TGT_LLVM"         \
    -DCLANG_VENDOR="Heiwa/Linux"                \
    -DCLANG_ENABLE_ARCMT=OFF                    \
    -DCLANG_ENABLE_STATIC_ANALYZER=OFF          \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++           \
    -DCLANG_DEFAULT_RTLIB=compiler-rt           \
    -DCLANG_DEFAULT_LINKER=lld                  \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind         \
    -DDEFAULT_SYSROOT="/clang2-tools"
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang2-tools" -P cmake_install.cmake && \
    popd
}
```
```bash
# Configure Stage-1 Clang/LLVM with pc triplet to produce binaries with "/clang2-tools/lib/ld-musl-${TGT_ARCH}.so.1".
ln -sfv clang   /clang2-tools/bin/${TGT_TRIPLET}-clang
ln -sfv clang++ /clang2-tools/bin/${TGT_TRIPLET}-clang++
cat > /clang2-tools/bin/${TGT_TRIPLET}.cfg << EOF
-Wl,-dynamic-linker /clang2-tools/lib/ld-musl-${TGT_ARCH}.so.1
EOF
```
```bash
# Configure Stage-1 Clang/LLVM with heiwa triplet to produce binaries with "/lib/ld-musl-${TGT_ARCH}.so.1" and "/usr" in chroot later.
ln -sfv clang                /clang2-tools/bin/${HEI_TRIPLET}-clang
ln -sfv ${HEI_TRIPLET}-clang /clang2-tools/bin/cc
ln -sfv clang++              /clang2-tools/bin/${HEI_TRIPLET}-clang++
cat > /clang2-tools/bin/${HEI_TRIPLET}.cfg << EOF
--sysroot=/usr -Wl,-dynamic-linker /lib/ld-musl-${TGT_ARCH}so.1
EOF
```
```bash
# Now set the compiler to pc triplet from Stage-1 Clang/LLVM to use current musl libc built and remove "/clang1-tools" from PATH.
sed -e "/CXX=/s/${CXX}/${TGT_TRIPLET}-clang++/" \
    -e "/CC=/s/${CC}/${TGT_TRIPLET}-clang/"     \
    -e 's|:/clang1-tools/bin:|:|' -i ~/.bashrc
source ~/.bashrc
```
```bash
# Sanity check for pc triplet of Stage-1 Clang/LLVM.
${CC} -x c++ - ${CFLAGS} ${LDFLAGS} -Wl,--verbose -v <<< 'int main(){}' &> dummy.log
${READELF} -l a.out | grep --color=auto "Req.*ter"
```
```bash
# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang2-tools/lib/ld-musl-x86_64.so.1]
```
```bash
# Ensure `ld.lld` invokes C runtime objects from current musl libc built.
# [ https://wiki.osdev.org/Creating_a_C_Library#Program_Initialization ]
grep --color=auto "ld.lld:.*crt[1in].o" dummy.log
```
```bash
# | The output should be:
# |-----------------------
# |ld.lld: /clang2-tools/lib/Scrt1.o
# |ld.lld: /clang2-tools/lib/crti.o
# |ld.lld: /clang2-tools/lib/crtn.o
```
```bash
# Back to "${HEIWA}/sources/pkgs" directory.
popd
```

### `7` - Xz
> #### `5.2.5` or newer
> The Xz package contains programs for compressing and decompressing files. It provides capabilities for the lzma and the newer xz compression formats. Compressing text files with xz yields a better compression percentage than with the traditional gzip or bzip2 commands.

> **Required!** As the default ".xz" and ".lzma" files de/compressor at current and later stage (chroot environment).
```bash
# Configure source. Use optimization level 3.
CFLAGS="-flto=thin $(tr Os O3 <<< "$CFLAGS")" \
./configure --prefix=/clang2-tools            \
            --build=$(build-aux/config.guess) \
            --host=${TGT_TRIPLET}             \
            --enable-threads=posix            \
            --disable-doc                     \
            --disable-static
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `8` - Pigz
> #### `2.6` or newer
> The Pigz package contains parallel implementation of gzip, is a fully functional replacement for GNU zip that exploits multiple processors and multiple cores to the hilt when compressing data.

> **Required!** As the default ".gz" files de/compressor at current and later stage (chroot environment).
```bash
# Build. Use optimization level 3.
time { make CC=${CC} CFLAGS="-flto=thin $(tr Os O3 <<< "$CFLAGS")"; }
```
```bash
# Install and create symlinks as `gzip` tools.
time {
    install -vm755 -t /clang2-tools/bin/ pigz
    ln -sfv pigz   /clang2-tools/bin/unpigz
    ln -sfv pigz   /clang2-tools/bin/gzip
    ln -sfv unpigz /clang2-tools/bin/gunzip
}
```

### `9` - Gettext-tiny
> #### `0.3.2` or newer
> The Gettext-tiny package contains utilities for internationalization and localization. These allow programs to be compiled with NLS (Native Language Support), enabling them to output messages in the user's native language. A lightweight replacements for tools typically used from the GNU gettext suite, which is incredibly bloated and takes a lot of time to build (in the order of an hour on slow devices).

> **Required!** To allow programs compiled with NLS (Native Language Support) in the later stage (chroot environment).
```bash
# Build.
time { make LIBINTL=MUSL CFLAGS="-flto=thin $CFLAGS"; }
```
```bash
# Only install the tools.
time { install -vm755 -t /clang2-tools/bin/ autopoint msg{fmt,merge} xgettext; }
```

### `10` - Toybox (Coreutils, File, Findutils, Grep, Sed, Tar)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!** For current and later stage (chroot environment).
```bash
# Copy the Toybox .config file. Ensure `libz` is enabled in the config.
cp -v ../../extra/toybox/files/.config.toolchain.nolibcrypto .config
grep --color=auto "CONFIG_TOYBOX_LIBZ=y" .config
```
```bash
# Export commands as TOYBOX variable that will be used to verify Toybox .config.
read -rd '' TOYBOX << "EOF"
base64 base32 basename cat chgrp chmod chown chroot cksum comm cp cut date dd
df dirname du echo env expand expr factor false fmt fold groups head hostid id
install link ln logname ls md5sum mkdir mkfifo mknod mktemp mv nice nl nohup
nproc od paste printenv printf pwd readlink realpath rm rmdir seq sha1sum shred
sleep sort split stat stty sync tac tail tee test timeout touch tr true truncate
tty uname uniq unlink wc who whoami yes file find xargs egrep grep fgrep sed tar
EOF
```
```bash
# Verify 87 commands, and ensure enabled (=y).
# Pipe to ` | wc -l` at the right of `done` to check the total of commands.
for X in ${TOYBOX}; do
    grep -v '#' .config | grep --color=auto -i "CONFIG_${X}=" \
    || >&2 echo "> ${X} not CONFIGURED!"
done
```
```bash
# Build with verbose. Toybox will use `cc` that breaks the build, so point HOSTCC to current CC.
time { make HOSTCC=${CC} CFLAGS="-flto=thin $CFLAGS" V=1; }
```
```bash
# Verify compiled 87 commands.
./toybox | tr ' ' '\n' \
| grep --color=auto -xE $(echo ${TOYBOX} | tr ' ' '|') | wc -l
```
```bash
# Verify unconfigured commands, but compiled.
# `[` (coreutils)
./toybox | tr ' ' '\n' \
| grep --color=auto -vxE $(echo ${TOYBOX} | tr ' ' '|')
```
```bash
# So, totally is 88 commands.
./toybox | wc -w
```
```bash
# Install to single "bin" directory rather than {,usr}/{bin,sbin}.
time { make HOSTCC=${CC} PREFIX=/clang2-tools/bin install_flat && unset X TOYBOX; }
```

### `11` - GNU Awk
> #### `5.1.0` or newer
> The GNU Awk (gawk) package contains programs for manipulating text files.

> **Required!** For current and later stage (chroot environment) that most build systems depends on GNU-style permutation.
```bash
# Ensure some unneeded files are not installed, use symlinks rather than hardlinks, and disable version links.
sed -e '/^LN =/s/=.*/= $(LN_S)/'         \
    -e '/install-exec-hook:/s/$/\nfoo:/' \
    -e 's|extras||' -si {,doc/}Makefile.in
```
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"           \
./configure --prefix=/clang2-tools    \
            --build=$(./config.guess) \
            --host=${TGT_TRIPLET}
```
```bash
# Build.
time { make; }
```
```bash
# Install and create symlink as `awk` which not linked because version links is disabled.
time {
    make install
    ln -sfv gawk /clang2-tools/bin/awk
}
```

### `12` - GNU Diffutils
> #### `3.8` or newer
> The GNU Diffutils package contains programs that show the differences between files or directories.

> **Required!** For current and later stage (chroot environment) that most build systems depends on GNU-style permutation.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                   \
./configure --prefix=/clang2-tools            \
            --build=$(build-aux/config.guess) \
            --host=${TGT_TRIPLET}             \
            --with-packager="Heiwa/Linux"
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `13` - GNU Make
> #### `4.3` or newer
> The GNU Make package contains a program for controlling the generation of executables and other non-source files of a package from source files.

> **Required!** For current and later stage (chroot environment) that most build systems depends on GNU-style permutation.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                   \
./configure --prefix=/clang2-tools            \
            --build=$(build-aux/config.guess) \
            --host=${TGT_TRIPLET}             \
            --without-guile
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `14` - GNU Patch
> #### `2.7.6` or newer
> The GNU Patch package contains a program for modifying or creating files by applying a patch file typically created by the diff program.

> **Required!** For current and later stage (chroot environment). The GNU's `patch` can handle offset lines, which is powerful feature.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                   \
./configure --prefix=/clang2-tools            \
            --build=$(build-aux/config.guess) \
            --host=${TGT_TRIPLET}
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `15` - GNU Texinfo
> #### `6.8` or newer
> The Texinfo package contains programs for reading, writing, and converting info pages.

> **Required!** For most packages in the later stage (chroot environment). Nothing is GNU-free.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                   \
./configure --prefix=/clang2-tools            \
            --build=$(build-aux/config.guess) \
            --host=${TGT_TRIPLET}             \
            --disable-perl-xs                 \
            --without-external-libintl-perl   \
            --without-external-Text-Unidecode \
            --without-external-Unicode-EastAsianWidth
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `16` - GNU Bash
> #### `5.1` (with patch level 8) or newer
> The GNU Bash package contains the Bourne-Again SHell.

> **Required!** As the default shell in the later stage (chroot environment).
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"                 \
./configure --prefix=/clang2-tools          \
            --build=$(support/config.guess) \
            --host=${TGT_TRIPLET}           \
            --without-bash-malloc           \
            --enable-net-redirections       \
            --disable-profiling
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `17` - Perl (+ cross)
> #### `5.34.0` (+ `1.3.6` for cross)
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** To build required packages in the later stage (chroot environment). 
```bash
# Decompress, then copy `perl-cross` over the source.
tar xzf ../perl-cross-1.3.6.tar.gz && \
cp -af ./perl-cross-1.3.6/* .
```
```bash
# Configure source (from perl-cross).
CFLAGS="-DNO_POSIX_2008_LOCALE -D_GNU_SOURCE $CFLAGS" \
HOSTLDFLAGS="-pthread" HOSTCFLAGS="-D_GNU_SOURCE"     \
LDFLAGS="-Wl,-z,stack-size=2097152 -pthread $LDFLAGS" \
./configure --prefix=/clang2-tools                    \
            --build=$(cnf/config.guess)
```
```bash
# Build. Fails with LTO since version 5.28.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `18` - Python3
> #### `3.9.7` or newer
> The Python3 package contains the Python development environment. It is useful for object-oriented programming, writing scripts, prototyping large programs, or developing entire applications.

> **Required!** To build Clang/LLVM and other required packages in the later stage (chroot environment).
```bash
# Prevent main script that uses hard-coded paths to host "/usr/include" and "/usr/lib" directories and use ThinLTO.
sed -i '/def add_multiarch_paths/a \        return' setup.py
sed -i 's|-flto|-flto=thin|' configure
```
```bash
# Configure source using provided libraries (built-in).
ax_cv_c_float_words_bigendian=no      \
./configure --prefix=/clang2-tools    \
            --build=$(./config.guess) \
            --host=${TGT_TRIPLET}     \
            --with-lto                \
            --without-ensurepip       \
            --enable-shared
```
```bash
# Build. -> Ignore all issues at the end! We won't build them. <-
time { make; }
```
```bash
# Install.
time { make install; }
```

### `19` - Cmake
> #### `3.21.3` or newer
> The CMake package contains a modern toolset used for generating Makefiles. It is a successor of the auto-generated configure script and aims to be platform- and compiler-independent. A significant user of CMake is KDE since version 4.

> **Required!** To build Clang/LLVM and other packages in the later stage (chroot environment).
```bash
# Disable applications that using Cmake from attempting to install files in "/usr/lib64".
sed -i '/"lib64"/s/64//' Modules/GNUInstallDirs.cmake
```
```bash
# Configure source using provided libraries (built-in). Use optimization level 3.
CFLAGS="-flto=thin $CFLAGS" CXXFLAGS="$CFLAGS" \
./bootstrap --prefix=/clang2-tools             \
            --mandir=/share/man                \
            --docdir=/share/doc/cmake-3.21.3   \
            --parallel=$(nproc)                \
            -- -DCMAKE_BUILD_TYPE=Release      \
            -Wno-dev -DCMAKE_USE_OPENSSL=OFF
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```

### `20` - Cleaning Up, Stripping Unneeded Symbols, and Saving Clang/LLVM Toolchain
> #### This section is recommended!

> Remove the documentation, manpages, and all unnecessary files.
```bash
rm -rf /clang2-tools/share/{bash-completion,doc,emacs,info,man,vim}
```
> The libtool ".la" files are only useful when linking with static libraries. They are unneeded and potentially harmful when using dynamic shared libraries, specially when using non-autotools build systems. So, remove those files.
```bash
find /clang2-tools/lib{,exec}/ -name '*.la' -exec rm -fv {} \;
```
> Strip off all unneeded symbols from binaries using `llvm-strip`. A large number of files will be reported "The file was not recognized as a valid object file". These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
```bash
find /clang2-tools/lib/ -type f \( -name '*.a' -o -name '*.so*' \) -exec llvm-strip --strip-unneeded {} \;
```
```bash
if cp -v $(command -v llvm-strip) ./; then
    find /clang2-tools/{libexec,bin}/ -type f -exec ./llvm-strip --strip-unneeded {} \;
fi && rm -v ./llvm-strip
```
> Now exit from privileged user.
```bash
exit
```
> #### * End of as privileged user!

> #### * Beginning of as root!

> #### Changing the directory ownership

> Change the ownership of the "${HEIWA}/clang2-tools" directory to root by running the following command.
```bash
if [[ -d "${HEIWA}/clang2-tools" ]]; then
    chown -R root:root ${HEIWA}/clang2-tools
fi
```
> #### Backup

> At this point the essential programs and libraries have been created and your current toolchain is in a good state. Your toolchain can now be backed up for later reuse. In case of fatal failures in the subsequent chapters, it often turns out that removing everything and starting over (more carefully) is the best option to recover. Unfortunately, all the temporary files will be removed, too. To avoid spending extra time to redo something which has been built successfully, prepare a backup.
```bash
if [[ -d "${HEIWA}/clang1-tools" && -d "${HEIWA}/clang2-tools" ]]; then
    pushd "$HEIWA" && export XZ_OPT="-9e -T2"      && \
        tar -cJpf clang1-tools.tar.xz clang1-tools && \
        tar -cJpf clang2-tools.tar.xz clang2-tools && \
    popd && unset XZ_OPT
fi
```
> #### Restore

> In case some mistakes have been made and you need to start over, you can use this backup to restore the system and save some recovery time. Since the sources are located under "$HEIWA", they are included in the backup archive as well, so they do not need to be downloaded again.
```bash
if [[ -d "${HEIWA}/clang1-tools" && -d "${HEIWA}/clang2-tools" ]]; then
    pushd "$HEIWA" && rm -rf clang{0,1}-tools        && \
        tar -xpf clang1-tools.tar.xz --numeric-owner && \
        tar -xpf clang2-tools.tar.xz --numeric-owner && \
    popd
fi
```
> #### * End of as root!

<h2></h2>

~Continue to [Stage-2 Final System](./4-Stage2_Final_System.md).~ (under developments)
