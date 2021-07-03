## `III` Stage-1 Clang/LLVM Toolchain

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

### `1` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Set default compiler to Stage-0 Clang/LLVM.
CC="clang" CXX="clang++"
export CC CXX

# Configure source.
./configure --prefix=/

# Build.
time { make; }

# Install.
time { make DESTDIR=/clang1-tools install; }

# Fix a wrong object symlink.
ln -sfv libc.so /clang1-tools/lib/ld-musl-x86_64.so.1

# Create a ldd symlink to use to print shared object dependencies.
mkdir -v /clang1-tools/bin && \
ln -sv ../lib/libc.so /clang1-tools/bin/ldd

# Configure PATH for dynamic linker.
mkdir -v /clang1-tools/etc && \
cat > /clang1-tools/etc/ld-musl-x86_64.path << "EOF"
/clang1-tools/lib
EOF

# Quick test for Stage-0 Clang/LLVM.
echo "int main(){}" > dummy.c
${HEIWA_TARGET}-clang dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ": /clang1-tools"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang1-tools/lib/ld-musl-x86_64.so.1]

grep "ld.lld:.*crt[1in].o" dummy.log

# | The output should be:
# |-----------------------
# |ld.lld: /clang0-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../Scrt1.o
# |ld.lld: /clang0-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../crti.o
# |ld.lld: /clang0-tools/lib/gcc/x86_64-heiwa-linux-musl/10.3.1/../../../crtn.o
```

### `2` - Linux API Headers
> #### Xanmod-CacULE, `5.12.x` or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
```bash
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time {
    make ARCH=${HEIWA_ARCH} LLVM=1 HOSTCC=${HEIWA_TARGET}-clang headers_check && \
    make ARCH=${HEIWA_ARCH} LLVM=1 HOSTCC=${HEIWA_TARGET}-clang headers
}

# @owl4ce don't know,
# why when HOSTCC is default in "LLVM=1" and/or using `clang` is failed to compile "scripts/basic/fixdep.c",
# but successful using `${TARGET_TRIPLET}-compiler`.

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
cp -rv usr/include /clang1-tools/.
```

### `3` - LLVM libunwind
> #### `12.0.0`
> C++ runtime stack unwinder from LLVM.

> *No need to re-decompress package.*

> **Required!** As mentioned in the description above.
```bash
# Set default compiler to new symlink from Stage-0 Clang/LLVM.
CC="${HEIWA_TARGET}-clang" CXX="${HEIWA_TARGET}-clang++"
export CC CXX

# Configure source.
pushd ${LLVM_SRC}/projects/libunwind/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"           \
        -DLIBUNWIND_ENABLE_SHARED=ON                     \
        -DCMAKE_C_FLAGS="-fPIC"                          \
        -DCMAKE_CXX_FLAGS="-fPIC"                        \
        -DCMAKE_AR="/clang0-tools/bin/llvm-ar"           \
        -DCMAKE_LINKER="/clang0-tools/bin/ld.lld"        \
        -DCMAKE_NM="/clang0-tools/bin/llvm-nm"           \
        -DCMAKE_OBJCOPY="/clang0-tools/bin/llvm-objcopy" \
        -DCMAKE_OBJDUMP="/clang0-tools/bin/llvm-objdump" \
        -DCMAKE_RANLIB="/clang0-tools/bin/llvm-ranlib"   \
        -DCMAKE_READELF="/clang0-tools/bin/llvm-readelf" \
        -DCMAKE_STRIP="/clang0-tools/bin/llvm-strip"     \
        -DLIBUNWIND_USE_COMPILER_RT=ON                   \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time { make -C build install && rm -rf build && popd; }
```

### `4` - LLVM libcxxabi
> #### `12.0.0`
> Low level support for a standard C++ library from LLVM.

> *No need to re-decompress package.*

> **Required!** As mentioned in the description above.
```bash
# Configure source.
pushd ${LLVM_SRC}/projects/libcxxabi/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                            \
        -DLIBCXXABI_ENABLE_STATIC=ON                                      \
        -DLIBCXXABI_USE_COMPILER_RT=ON                                    \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON                                  \
        -DLIBCXXABI_LIBUNWIND_PATH="/clang1-tools/lib"                    \
        -DLIBCXXABI_LIBCXX_INCLUDES="${LLVM_SRC}/projects/libcxx/include" \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time {
    make -C build install && \
    cp -v include/*.h /clang1-tools/include/ && \
    rm -rf build && popd
}
```

### `5` - LLVM libcxx
> #### `12.0.0`
> New implementation of the C++ standard library, targeting C++11 from LLVM.

> *No need to re-decompress package.*

> **Required!** As mentioned in the description above.
```bash
# Deletes atomic detection for Linux, to build libcxx with "libatomic.so*" free (provided by GCC).
sed -i '/check_library_exists(atomic __atomic_fetch_add_8 "" LIBCXX_HAS_ATOMIC_LIB)/d' \
${LLVM_SRC}/projects/libcxx/cmake/config-ix.cmake

# Configure source.
pushd ${LLVM_SRC}/projects/libcxx/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                 \
        -DLIBCXX_ENABLE_SHARED=ON                              \
        -DLIBCXX_ENABLE_STATIC=ON                              \
        -DLIBCXX_HAS_MUSL_LIBC=ON                              \
        -DLIBCXX_USE_COMPILER_RT=ON                            \
        -DLIBCXX_INSTALL_HEADERS=ON                            \
        -DLIBCXX_CXX_ABI=libcxxabi                             \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON                       \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/clang1-tools/include" \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/clang1-tools/lib"      \
        -DCMAKE_CXX_FLAGS="-isystem /clang1-tools/include"     \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time { make -C build install && rm -rf build && popd; }
```
> #### ^ NOTE!
> Now, you can safely remove ${LLVM_SRC} directory after the [above step](#5---llvms-libcxx).

### `6` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** To build Stage-1 Clang/LLVM and for most programs that depends on `-ltinfo` or `-lterminfo` linker's flags.
```bash
# Build.
time { make CC=${HEIWA_TARGET}-clang CFLAGS="$COMMON_FLAGS -Wall -fPIC" all-dynamic; }

# Install.
time { make PREFIX=/ DESTDIR=/clang1-tools install-dynamic; }
```

### `7` - libexecinfo
> #### `1.1` or newer
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** To build Stage-1 Clang/LLVM, since using musl libc.
```bash
# Apply patches (from Alpine Linux).
patch -Np1 -i ../../extra/libexecinfo/patches/10-execinfo.patch
patch -Np1 -i ../../extra/libexecinfo/patches/20-define-gnu-source.patch
patch -Np1 -i ../../extra/libexecinfo/patches/30-linux-makefile.patch

# Build.
time { make CC=clang AR=llvm-ar CFLAGS="$COMMON_FLAGS -fno-omit-frame-pointer"; }

# Install.
install -vm755 -t /clang1-tools/include/ execinfo.h stacktraverse.h
install -vm755 -t /clang1-tools/lib/ libexecinfo.a libexecinfo.so.1
ln -sv libexecinfo.so.1 /clang1-tools/lib/libexecinfo.so
```

### `8` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** By Pigz and optionally enabled to build Stage-1 Clang/LLVM.
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools \
    --zlib-compat --native

# Build.
time { make; }

# Install.
time { make install; }

# Remove an useless static library.
rm -fv /clang1-tools/lib/libz.a
```

### `9` - Clang/LLVM
> #### `12.0.0`
> C language family frontend for LLVM.

> **Required!** As default toolchain. This will build Stage-1 Clang/LLVM toolchain with `libgcc_s.so*` and `libstdc++.so*` free.
```bash
# Exit from "${HEIWA}/sources/pkg/llvm-12.0.0.src" directory if already in.
popd

# Rename the LLVM source directory to "$LLVM_SRC", then enters.
mv -v llvm-12.0.0.src "$LLVM_SRC" && cd "$LLVM_SRC"

# Decompress clang, lld, and compiler-rt to correct directories.
pushd ${LLVM_SRC}/projects" && \
    tar xf ../../pkgs/compiler-rt-12.0.0.src.tar.xz && mv -v compiler-rt-12.0.0.src compiler-rt
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.0.src.tar.xz && mv -v clang-12.0.0.src clang
    tar xf ../../pkgs/lld-12.0.0.src.tar.xz   && mv -v lld-12.0.0.src lld
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/stage1-appatch

# Disable sanitizers for musl, fixing "early build failure".
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Set default compiler to new symlink from Stage-0 Clang/LLVM.
# Sets C and C++ compiler's build flags to reduce debug symbols.
CC="${HEIWA_TARGET}-clang" CFLAGS="-g -g1"
CXX="${HEIWA_TARGET}-clang++" CXXFLAGS="-g -g1"
export CC CXX CFLAGS CXXFLAGS

# Update host/target triple detection.
cp -fv ../extra/llvm/files/config.guess cmake/.

# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release                                  \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"                      \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"         \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-musl"                   \
    -DLLVM_TARGET_ARCH="X86"                                    \
    -DLLVM_TARGETS_TO_BUILD="X86"                               \
    -DLLVM_LINK_LLVM_DYLIB=ON                                   \
    -DLLVM_BUILD_LLVM_DYLIB=ON                                  \
    -DLLVM_BUILD_TESTS=OFF                                      \
    -DLLVM_ENABLE_LIBEDIT=OFF                                   \
    -DLLVM_ENABLE_LIBXML2=OFF                                   \
    -DLLVM_ENABLE_LIBCXX=ON                                     \
    -DLLVM_ENABLE_LLD=ON                                        \
    -DLLVM_ENABLE_RTTI=ON                                       \
    -DLLVM_ENABLE_ZLIB=ON                                       \
    -DLLVM_INCLUDE_GO_TESTS=OFF                                 \
    -DLLVM_INCLUDE_TESTS=OFF                                    \
    -DLLVM_INCLUDE_DOCS=OFF                                     \
    -DLLVM_INCLUDE_EXAMPLES=OFF                                 \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                               \
    -DCOMPILER_RT_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"  \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF                          \
    -DCOMPILER_RT_BUILD_XRAY=OFF                                \
    -DCOMPILER_RT_BUILD_PROFILE=OFF                             \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                           \
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON                       \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                           \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                         \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                           \
    -DCLANG_DEFAULT_LINKER="/clang1-tools/bin/ld.lld"           \
    -DDEFAULT_SYSROOT="/clang1-tools"                           \
    -DBacktrace_INCLUDE_DIR="/clang1-tools/include"             \
    -DBacktrace_LIBRARY="/clang1-tools/lib/libexecinfo.so"      \
    -DCMAKE_C_COMPILER_AR="/clang0-tools/bin/llvm-ar"           \
    -DCMAKE_CXX_COMPILER_AR="/clang0-tools/bin/llvm-ar"         \
    -DCMAKE_C_COMPILER_RANLIB="/clang0-tools/bin/llvm-ranlib"   \
    -DCMAKE_CXX_COMPILER_RANLIB="/clang0-tools/bin/llvm-ranlib" \
    -DCMAKE_INSTALL_OLDINCLUDEDIR="/clang1-tools/include"       \
    -DCMAKE_LINKER="/clang0-tools/bin/ld.lld"                   \
    -DCMAKE_NM="/clang0-tools/bin/llvm-nm"                      \
    -DCMAKE_OBJCOPY="/clang0-tools/bin/llvm-objcopy"            \
    -DCMAKE_READELF="/clang0-tools/bin/llvm-readelf"            \
    -DCMAKE_STRIP="/clang0-tools/bin/llvm-strip"                \
    -DICONV_LIBRARY_PATH="/clang1-tools/lib/libc.so"

# Build.
time { make -C build; }

# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang1-tools" -P cmake_install.cmake && \
    popd && rm -rf build
}

# Since Binutils won't be used, create a symlink to LLVM tools and set lld as default toolchain's linker.
for B in as ar ranlib readelf nm objcopy objdump size strip; do
    ln -sv llvm-${B} /clang1-tools/bin/${B}
done; unset B; ln -sv lld /clang1-tools/bin/ld

# Configure Stage-1 Clang to build binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1" instead of "/lib/ld-musl-x86_64.so.1".
ln -sv clang-12 /clang1-tools/bin/x86_64-pc-linux-musl-clang
ln -sv clang-12 /clang1-tools/bin/x86_64-pc-linux-musl-clang++
cat > /clang1-tools/bin/x86_64-pc-linux-musl.cfg << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF

# Set the new PATH since "/clang0-tools" won't be used anymore.
export PATH="/clang1-tools/bin:/clang1-tools/usr/bin:/bin:/usr/bin"

# Configure new Stage-1 Clang/LLVM environment.
sed -i "s|PATH=.*|PATH=\"${PATH}\"|" ~/.bashrc
sed -i '/unset CFLAGS CXXFLAGS/d' ~/.bashrc
cat >> ~/.bashrc << "EOF"
# Stage-1 Clang/LLVM environment.
CC="${TARGET_TRUPLE}-clang"
CXX="${TARGET_TRUPLE}-clang++"
AR="llvm-ar"
AS="llvm-as"
RANLIB="llvm-ranlib"
LD="ld.lld"
STRIP="llvm-strip"
export CC CXX AR AS RANLIB LD STRIP
EOF
source ~/.bash_profile

# Back to "${HEIWA}/sources/pkg" directory.
cd ${HEIWA}/sources/pkgs/
```

### `10` - OpenBSD M4
> #### `6.7` or newer
> The OpenBSD M4 package contains a macro processor.

> **Required!** As default front end to any language (e.g: C, ratfor, fortran, lex, and yacc).
```bash
# Configure source.
./configure --prefix=/clang1-tools --enable-m4

# Build. Fails when using multiple jobs.
time { make -j1; }

# Install.
time { make install; }
```

### `11` - GNU Bash
> #### `5.1` (with patch level 8) or newer
> The GNU Bash package contains the Bourne-Again SHell.

> **Required!** As default shell for the next stage (chroot environment).
```bash
# Fix the configure script that doesn't determine correct values in cross compiling.
cat > config.cache << "EOF"
ac_cv_func_mmap_fixed_mapped=yes
ac_cv_func_strcoll_works=yes
ac_cv_func_working_mktime=yes
bash_cv_func_sigsetjmp=present
bash_cv_getcwd_malloc=yes
bash_cv_job_control_missing=present
bash_cv_printf_a_format=yes
bash_cv_sys_named_pipes=present
bash_cv_ulimit_maxfds=yes
bash_cv_under_sys_siglist=yes
bash_cv_unusable_rtsigs=no
gt_cv_int_divbyzero_sigfpe=yes
EOF

# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}  \
    --without-bash-malloc    \
    --cache-file=config.cache

# Build.
time { make; }

# Install.
time { make install; }
```

### `12` - Toybox (Coreutils, File, Findutils, Grep, Sed, and Tar)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!** For the current and next stage (chroot environment).
```bash
# Copy Toybox's .config file.
cp -v ../../extra/toybox/files/.config.coreutils_file_findutils_grep_sed_tar.nlns .config

export CFFGPT="base64 base32 basename cat chgrp chmod chown chroot cksum comm cp cut date
dd df dirname du echo env expand expr factor false fmt fold groups head hostid id install
link ln logname ls md5sum mkdir mkfifo mknod mktemp mv nice nl nohup nproc od paste printenv
printf pwd readlink realpath rm rmdir seq sha1sum shred sleep sort split stat stty sync tac
tail tee test timeout touch tr true truncate tty uname uniq unlink wc who whoami yes file
find xargs egrep grep fgrep sed tar"

# Checks 87 commands, and make sure is enabled (=y).
# Pipe to " | wc -l" at the right of "done" to checks total of commands.
for X in ${CFFGPT}; do
    grep -v '#' .config | grep -i "_${X}=" || echo "* $X not CONFIGURED"
done

# Build.
time { make; }

# Checks compiled 87 commands.
./toybox | tr ' ' '\n'i | grep -xE $(echo $CFFGPT | tr ' ' '|'i) | wc -l

# Checks commands that not configured but compiled.
# `[` (coreutils)
./toybox | tr ' ' '\n'i | grep -vxE $(echo $CFFGPT | tr ' ' '|'i)

# So, totally is 88 commands.
./toybox | wc -w

# Install.
time { make PREFIX=/clang1-tools install && unset CFFGPT; }
```

### `13` - GNU Diffutils
> #### `3.7` or newer
> The GNU Diffutils package contains programs that show the differences between files or directories.

> **Required!** For the current and next stage (chroot environment) that most build systems depends on GNU implementation style.
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}

# Build.
time { make; }

# Install.
time { make install; }
```

### `14` - GNU AWK
> #### `5.1.0` or newer
> The GNU AWK (gawk) package contains programs for manipulating text files.

> **Required!** For the current and next stage (chroot environment) that most build systems depends on GNU implementation style.
```bash
# Ensure some unneeded files are not installed.
sed -i 's|extras||' Makefile.in

# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}

# Build.
time { make; }

# Install.
time { make install; }
```

### `15` - Pigz
> #### `2.6` or newer
> The Pigz package contains parallel implementation of gzip, is a fully functional replacement for GNU zip that exploits multiple processors and multiple cores to the hilt when compressing data.

> **Required!** As default ".gz" files de/compressor for the current and next stage (chroot environment).
```bash
# Build.
time { make CC="$CC" CFLAGS="$CFLAGS"; }

# Install and make symlink as `gzip` programs.
install -vm755 -t /clang1-tools/bin/ pigz
ln -sv pigz /clang1-tools/bin/unpigz
ln -sv pigz /clang1-tools/bin/gzip
ln -sv unpigz /clang1-tools/bin/gunzip
```

### `16` - GNU Make
> #### `4.3` or newer
> The GNU Make package contains a program for controlling the generation of executables and other non-source files of a package from source files.
 
> **Required!** For the current and next stage (chroot environment) that most build systems depends on GNU implementation style.
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}  \
    --without-guile

# Build.
time { make; }

# Install.
time { make install; }
```

### `17` - GNU Patch
> #### `2.7.6` or newer
> The GNU Patch package contains a program for modifying or creating files by applying a patch file typically created by the diff program.

> **Required!** For the current and next stage (chroot environment). The GNU implementation of `patch` is can handle offset lines, which is powerful feature.
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}

# Build.
time { make; }

# Install.
time { make install; }
```

### `18` - Xz
> #### `5.2.5` or newer
> The Xz package contains programs for compressing and decompressing files. It provides capabilities for the lzma and the newer xz compression formats. Compressing text files with xz yields a better compression percentage than with the traditional gzip or bzip2 commands.

> **Required!** As default ".xz" and ".lzma" files de/compressor for the current and next stage (chroot environment).
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}  \
    --disable-static

# Build.
time { make; }

# Install.
time { make install; }
```

### `19` - Gettext-tiny
> #### `0.3.2` or newer
> The Gettext-tiny package provides lightweight replacements for tools typically used from the GNU gettext suite, which is incredibly bloated and takes a lot of time to build (in the order of an hour on slow devices).

> **Required!** To allow programs compiled with NLS (Native Language Support) in the next stage (chroot environment).
```bash
# Build.
time { make LIBINTL=MUSL prefix=/clang1-tools; }

# Install the msgfmt, msgmerge and xgettext.
install -vm755 -t /clang1-tools/bin/ msgfmt msgmerge xgettext 
```

### `20` - OpenBSD Yacc
> #### `6.6` or newer
> The OpenBSD Yacc package contains a parser generator.

> **Required!** To build required packages in the next stage (chroot environment). 
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools --enable-yacc \
    --mandir=/clang1-tools/share/man/man1

# Build.
time { make; }

# Install.
time { make BINDIR=/clang1-tools/bin install; }
```

### `21` - Perl (cross)
> #### `5.32.1` and `1.3.5` for cross
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** To build required packages in the next stage (chroot environment). 
```bash
# Copy `perl-cross` over the source.
pushd ../ && \
    tar xzf perl-cross-1.3.5.tar.gz       && \
popd && \
cp -rf ../perl-cross-1.3.5/*       ./     && \
cp -rf ../perl-cross-1.3.5/utils/* utils/ && \
rm -rf ../perl-cross-1.3.5

# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --target=${TARGET_TRUPLE}

# Build.
time { make; }

# Install.
time { make install; }
```

### `22` - libffi
> #### `3.3` or newer
> The libffi library provides a portable, high level programming interface to various calling conventions. This allows a programmer to call any function specified by a call interface description at run time.

> **Required!** By Python3 for the current and next stage (chroot environment).
```bash
# Apply patches (from Void Linux) to fix some issues.
patch -Np1 -i ../../extra/libffi/patches/libffi-race-condition.patch
patch -Np1 -i ../../extra/libffi/patches/no-toolexeclibdir.patch

# Configure source.
./configure \
    --prefix=/clang1-tools       \
    --build=${TARGET_TRUPLE}     \
    --host=${TARGET_TRUPLE}      \
    --disable-static --with-pic  \
    --disable-multi-os-directory \
    --with-gcc-arch=native

# Build.
time { make; }

# Install.
time { make install; }
```

### `23` - Python3
> #### `3.9.6` or newer
> The Python 3 package contains the Python development environment. It is useful for object-oriented programming, writing scripts, prototyping large programs, or developing entire applications.

> **Required!** To build Clang/LLVM and other required packages in the next stage (chroot environment).
```bash
# Apply patches (from Void Linux).
patch -Np1 -i ../../extra/python3/patches/musl-find_library.patch
patch -Np1 -i ../../extra/python3/patches/tweak-MULTIARCH-for-powerpc-linux-musl.patch

# Make sure to use installed `libffi`, not built-in.
rm -rfv Modules/_ctypes/{darwin,libffi}*

# Prevent main script that uses hard-coded paths to the host "/usr/include" and "/usr/lib" directories.
sed -i '/def add_multiarch_paths/a \        return' setup.py

# Configure source.
ax_cv_c_float_words_bigendian=no \
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}  \
    --enable-shared          \
    --without-ensurepip 

# Build.
time { make; }

# Install.
time { make install; }
```

### `24` - GNU Texinfo
> #### `6.7` or newer
> The Texinfo package contains programs for reading, writing, and converting info pages.

> **Required!** For the next stage (chroot environment). Nothing is GNU-free.
```bash
# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}

# Build.
time { make; }

# Install.
time { make install; }
```

### `25` - Bzip2
> #### `1.0.8` or newer
> The Bzip2 package contains programs for compressing and decompressing files. Compressing text files with bzip2 yields a much better compression percentage than with the traditional gzip.

> **Required!** As default ".bz*" files de/compressor for the next stage (chroot environment).
```bash
# Fix the makefile to ensures installation of symlinks are relative and the man pages are installed into correct location.
sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile

# Prepare.
time { make -f Makefile-libbz2_so CC="$CC" AR="$AR" RANLIB="$RANLIB" && make clean; }

# Build.
time { make CC="$CC" AR="$AR" RANLIB="$RANLIB"; }

# Install.
time { make PREFIX=/clang1-tools install; }

# Remove an useless static library.
rm -fv /clang1-tools/lib/libbz2.a
```

### `26` - libuv
> #### `1.41.0` or newer
> The libuv package is a multi-platform support library with a focus on asynchronous I/O.

> **Required!** By Cmake.
```bash
# Generate configure script.
export LDFLAGS="-pthread"
NOCONFIGURE=1 ./autogen.sh

# Configure source.
./configure \
    --prefix=/clang1-tools   \
    --build=${TARGET_TRUPLE} \
    --host=${TARGET_TRUPLE}  \
    --disable-static

# Build.
time { make; }

# Install.
time { make install && unset LDFLAGS; }
```

### `27` - Cmake
> #### `3.20.5` or newer
> The CMake package contains a modern toolset used for generating Makefiles. It is a successor of the auto-generated configure script and aims to be platform- and compiler-independent. A significant user of CMake is KDE since version 4.

> **Required!** To build Clang/LLVM in the next stage (chroot environment).
```bash
# Disable applications using cmake from attempting to install files in "/usr/lib64".
sed -i '/"lib64"/s/64//' Modules/GNUInstallDirs.cmake

# Configure source using provided programs (built-in).
./bootstrap \
    --prefix=/clang1-tools           \
    --mandir=/share/man              \
    --parallel=$(nproc)              \
    --docdir=/share/doc/cmake-3.20.5 \
    -- -DCMAKE_USE_OPENSSL=OFF

# Build.
time { make; }

# Install.
time { make install; }
```

### `28` - Cleaning Up and Changing Ownership
> **This section is optional!**

> If the intended user is not a programmer and does not plan to do any debugging on the system software, the system size can be decreased by removing the debugging symbols from binaries and libraries. This causes no inconvenience other than not being able to debug the software fully anymore.
```bash
# The libtool .la files are only useful when linking with static libraries. Remove those files.
# They are unneeded, and potentially harmful, when using dynamic shared libraries, specially when using non-autotools build systems.
find /clang1-tools/{lib,libexec} -name \*.la -exec rm -rfv {} \;

# Remove the documentation.
rm -rf /clang1-tools/share/{info,man,doc}/*

# Strip off debugging symbols from binaries using `llvm-strip`.
# A large number of files will be reported "The file was not recognized as a valid object file".
# These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
find /clang1-tools/lib/ -maxdepth 1 -type f -exec llvm-strip --strip-debug {} \;
find /clang1-tools/{,usr/}{,s}bin/ -maxdepth 1 -type f -exec /clang0-tools/bin/llvm-strip --strip-unneeded {} \;

# Exit from privileged user.
exit

# Change the ownership of the "${HEIWA}/clang1-tools" directory to root by running the following command.
# Warning! This is danger, so check its variables before `chown`.
# echo ${HEIWA}/clang1-tools
[[ -n "$HEIWA" ]] && chown -R root:root ${HEIWA}/clang1-tools
```
> #### * End of as privileged user!

<h2></h2>

Continue to [Final System](./4-Final_System.md).
