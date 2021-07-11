## `III` Stage-1 Clang/LLVM Toolchain

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

### `0` - Setting Up Clang/LLVM Environment Variables
> Apply persistent toolchain environment variables, but currently don't set the compiler to the new triplet of Stage-0 Clang/LLVM. Meaning to use libc from Stage-0 Clang/LLVM firstly, in case to build musl libc in this stage.
```bash
cat >> ~/.bashrc << "EOF"
# Stage-1 Clang/LLVM environment.
CC="clang"
CXX="clang++"
AR="llvm-ar"
AS="llvm-as"
LD="ld.lld"
NM="llvm-nm"
OBJCOPY="llvm-objcopy"
OBJDUMP="llvm-objdump"
RANLIB="llvm-ranlib"
READELF="llvm-readelf"
SIZE="llvm-size"
STRIP="llvm-strip"
export CC CXX AR AS LD NM OBJCOPY OBJDUMP RANLIB READELF SIZE STRIP
EOF
source ~/.bashrc
```

### `1` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Configure source.
CFLAGS="-Oz -pipe" CXXFLAGS="-Oz -pipe" \
./configure --prefix=/ --enable-optimize=speed

# Build.
time { make; }

# Install and fix a wrong shared object symlink, also create a `ldd` symlink to use to print shared object dependencies.
time {
    make DESTDIR=/clang1-tools install
    ln -sfv libc.so /clang1-tools/lib/ld-musl-x86_64.so.1
    mkdir -v /clang1-tools/bin && \
    ln -sv ../lib/libc.so /clang1-tools/bin/ldd
}
```
```bash
# Set compiler to the new triplet from Stage-0 Clang/LLVM to use current libc.
sed -i 's|CC=.*|CC="${HEIWA_TARGET}-clang"|'     ~/.bashrc
sed -i 's|CXX=.*|CXX="${HEIWA_TARGET}-clang++"|' ~/.bashrc
source ~/.bashrc
```
```bash
# Configure PATH for dynamic linker.
mkdir -v /clang1-tools/etc && \
cat > /clang1-tools/etc/ld-musl-x86_64.path << "EOF"
/clang1-tools/lib
EOF

# Quick test for the new triplet of Stage-0 Clang/LLVM.
echo "int main(){}" > dummy.c
${CC} dummy.c -v -Wl,--verbose &> dummy.log
${READELF} -l a.out | grep ": /clang1-tools"

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
> #### `5.13.x` (CacULE) or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
```bash
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time { make ARCH=${HEIWA_ARCH} LLVM=1 HOSTCC=${CC} headers; }

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
cp -rfv usr/include /clang1-tools/.
```
```bash
# @owl4ce don't know ..
# why when HOSTCC is defaults since use "LLVM=1" (clang) it fails to compile "scripts/basic/fixdep.c",
# but successful using `${TARGET_TRIPLET}-compiler`.
```

### `3` - LLVM libunwind, libcxxabi, and libcxx
> #### `12.x.x` or newer
> 1. C++ runtime stack unwinder from LLVM;  
> 2. Low level support for a standard C++ library from LLVM;  
> 3. New implementation of the C++ standard library, targeting C++11 from LLVM.

> **Required!** As mentioned in the description above.
```bash
# Exit from the LLVM source directory if already entered after decompressing.
popd
```
```bash
# Rename the LLVM source directory to "$LLVM_SRC", then enter.
mv -fv llvm-12.0.1.src "$LLVM_SRC" && pushd "$LLVM_SRC"

# Decompress `libunwind`, `libcxxabi`, and `libcxx` to the correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/libunwind-12.0.1.src.tar.xz && mv -fv libunwind-12.0.1.src libunwind
    tar xf ../../pkgs/libcxxabi-12.0.1.src.tar.xz && mv -fv libcxxabi-12.0.1.src libcxxabi
    tar xf ../../pkgs/libcxx-12.0.1.src.tar.xz    && mv -fv libcxx-12.0.1.src libcxx
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/appatch L_LLVM

# Deletes atomic detection for Linux to build `libcxx` with "libatomic.so*" free (which is provided by GCC).
sed -i '/check_library_exists(atomic __atomic_fetch_add_8 "" LIBCXX_HAS_ATOMIC_LIB)/d' \
${LLVM_SRC}/projects/libcxx/cmake/config-ix.cmake

# Update config.guess for better platform detection.
cp -fv ../extra/llvm/files/config.guess cmake/.
```
```bash
# Configure `libunwind` source.
pushd ${LLVM_SRC}/projects/libunwind/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"  \
        -DCMAKE_C_FLAGS="-0z -pipe -g0 -fPIC"   \
        -DCMAKE_CXX_FLAGS="-0z -pipe -g0 -fPIC" \
        -DLIBUNWIND_ENABLE_SHARED=ON            \
        -DLIBUNWIND_USE_COMPILER_RT=ON          \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time { make -C build install && popd; }
```
```bash
# Configure `libcxxabi` source.
pushd ${LLVM_SRC}/projects/libcxxabi/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                            \
        -DCMAKE_CXX_FLAGS="-0z -pipe -g0"                                 \
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
    make -C build install                      && \
    cp -fv include/*.h /clang1-tools/include/. && popd
}
```
```bash
# Configure `libcxx` source.
pushd ${LLVM_SRC}/projects/libcxx/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                               \
        -DCMAKE_CXX_FLAGS="-0z -pipe -g0 -isystem /clang1-tools/include"     \
        -DLIBCXX_ENABLE_SHARED=ON                                            \
        -DLIBCXX_ENABLE_STATIC=ON                                            \
        -DLIBCXX_HAS_MUSL_LIBC=ON                                            \
        -DLIBCXX_INSTALL_HEADERS=ON                                          \
        -DLIBCXX_USE_COMPILER_RT=ON                                          \
        -DLIBCXX_CXX_ABI=libcxxabi                                           \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/clang1-tools/include"               \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/clang1-tools/lib"                    \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON                                     \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time { make -C build install && popd; }
```
```bash
# Back to "${HEIWA}/sources/pkgs" directory.
popd
```

### `4` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** To build Stage-1 Clang/LLVM and for the most programs that depends on `-ltinfo` or `-lterminfo` linker's flags.
```bash
# Build.
time { make CFLAGS="-Oz -pipe -fPIC" all-dynamic; }

# Install.
time { make PREFIX=/clang1-tools install-dynamic; }
```

### `5` - libexecinfo
> #### `1.1` or newer (from Heiwa/Linux fork)
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** To build Stage-1 Clang/LLVM, since using musl libc.
```bash
# Build.
time { make CFLAGS="-Oz -pipe"; }

# Install.
time { make PREFIX=/clang1-tools install; }
```

### `6` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** By Pigz in the current stage and optionally enabled to build Stage-1 Clang/LLVM.
```bash
# Configure source.
CFLAGS="-Oz -pipe" ./configure \
--prefix=/clang1-tools --zlib-compat --native

# Build.
time { make; }

# Install.
time { make install; }
```

### `7` - Clang/LLVM
> #### `12.x.x` or newer
> C language family frontend for LLVM.

> **Required!** Build Stage-1 Clang/LLVM toolchain with `libgcc_s.so*` and `libstdc++.so*` free (which is provided by GCC).
```bash
# Exit from the LLVM source directory if already entered after decompressing.
popd
```
```bash
# Rename the LLVM source directory to "$LLVM_SRC", then enter.
mv -fv llvm-12.0.1.src "$LLVM_SRC" && cd "$LLVM_SRC"

# Decompress `clang`, `lld`, and `compiler-rt` to the correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/compiler-rt-12.0.1.src.tar.xz && mv -fv compiler-rt-12.0.1.src compiler-rt
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.1.src.tar.xz && mv -fv clang-12.0.1.src clang
    tar xf ../../pkgs/lld-12.0.1.src.tar.xz   && mv -fv lld-12.0.1.src lld
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/appatch C_LLVM

# Disable sanitizers for musl, it's broken since it duplicates some libc bits.
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Update config.guess for better platform detection.
cp -fv ../extra/llvm/files/config.guess cmake/.
```
```bash
# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release                                  \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"                      \
    -DCMAKE_INSTALL_OLDINCLUDEDIR="/clang1-tools/include"       \
    -DLLVM_DEFAULT_TARGET_TRIPLE="$TARGET_TRUPLE"               \
    -DLLVM_HOST_TRIPLE="$TARGET_TRUPLE"                         \
    -DLLVM_TARGETS_TO_BUILD="X86"                               \
    -DLLVM_TARGET_ARCH="X86"                                    \
    -DLLVM_LINK_LLVM_DYLIB=ON                                   \
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
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF                          \
    -DCOMPILER_RT_BUILD_XRAY=OFF                                \
    -DCOMPILER_RT_BUILD_PROFILE=OFF                             \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                           \
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON                       \
    -DCOMPILER_RT_DEFAULT_TARGET_TRIPLE="$TARGET_TRUPLE"        \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                           \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                         \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                           \
    -DCLANG_DEFAULT_LINKER=ld.lld                               \
    -DDEFAULT_SYSROOT="/clang1-tools"                           \
    -DBacktrace_INCLUDE_DIR="/clang1-tools/include"             \
    -DBacktrace_LIBRARY="/clang1-tools/lib/libexecinfo.so"      \
    -DICONV_LIBRARY_PATH="/clang1-tools/lib/libc.so"            \
    -DLLVM_INSTALL_BINUTILS_SYMLINKS=ON                         \
    -DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON                          \
    -DLLVM_INSTALL_UTILS=ON                                     \
    -DLLVM_ENABLE_BINDINGS=OFF                                  \
    -DLLVM_ENABLE_IDE=OFF                                       \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF

# Build.
time { make -C build; }

# Install and remove the build directory.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang1-tools" -P cmake_install.cmake && \
    popd && rm -rf build
}

# Configure Stage-1 Clang with default triplet (pc) to produce binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1".
ln -sv clang   /clang1-tools/bin/${TARGET_TRUPLE}-clang
ln -sv clang++ /clang1-tools/bin/${TARGET_TRUPLE}-clang++
cat > /clang1-tools/bin/${TARGET_TRUPLE}.cfg << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF

# Set the new PATH since "/clang0-tools" won't be used anymore and the Stage-1 Clang default triplet (pc),
# also its time to enable optimization.
sed -i 's|/clang0-tools/usr/bin:/clang0-tools/bin:||' ~/.bashrc
sed -i '/unset CFLAGS CXXFLAGS/d'                     ~/.bashrc
sed -i 's|CC=.*|CC="${TARGET_TRUPLE}-clang"|'         ~/.bashrc
sed -i 's|CXX=.*|CXX="${TARGET_TRUPLE}-clang++"|'     ~/.bashrc
source ~/.bash_profile

# Back to "${HEIWA}/sources/pkgs" directory.
cd ${HEIWA}/sources/pkgs/
```

### `8` - Gettext-tiny
> #### `0.3.2` or newer
> The Gettext-tiny package contains utilities for internationalization and localization. These allow programs to be compiled with NLS (Native Language Support), enabling them to output messages in the user's native language. A lightweight replacements for tools typically used from the GNU gettext suite, which is incredibly bloated and takes a lot of time to build (in the order of an hour on slow devices).

> **Required!** To allow programs compiled with NLS (Native Language Support) in the next stage (chroot environment).
```bash
# Build.
time { make LIBINTL=MUSL prefix=/clang1-tools; }

# Only install `msgfmt`, `msgmerge` and `xgettext`.
install -vm755 -t /clang1-tools/bin/ msgfmt msgmerge xgettext 
```

### `9` - GNU AWK
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

### `10` - GNU Bash
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

### `11` - Toybox (Coreutils, File, Findutils, Grep, Sed, Tar)
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
# Pipe to " | wc -l" at the right of `done` to checks total of commands.
for X in ${CFFGPT}; do
    grep -v '#' .config | grep -i --color=auto "_${X}=" || echo "* $X not CONFIGURED"
done

# Build.
time { make; }

# Checks compiled 87 commands.
./toybox | tr ' ' '\n'i | grep -xE --color=auto $(echo $CFFGPT | tr ' ' '|'i) | wc -l

# Checks commands that not configured but compiled.
# `[` (coreutils)
./toybox | tr ' ' '\n'i | grep -vxE --color=auto $(echo $CFFGPT | tr ' ' '|'i)

# So, totally is 88 commands.
./toybox | wc -w

# Install.
time { make PREFIX=/clang1-tools install && unset CFFGPT; }
```

### `12` - GNU Diffutils
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

### `13` - GNU Make
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

### `14` - GNU Patch
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

### `15` - GNU Texinfo
> #### `6.8` or newer
> The Texinfo package contains programs for reading, writing, and converting info pages.

> **Required!** For the most packages next stage (chroot environment). Nothing is GNU-free.
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

### `16` - OpenBSD Yacc
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

# Install and create symlink as `byacc`.
time {
    make BINDIR=/clang1-tools/bin install && \
    ln -sv yacc /clang1-tools/bin/byacc
}
```

### `17` - Perl (cross)
> #### `5.32.1` and `1.3.5` for cross
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** To build required packages in the next stage (chroot environment). 
```bash
# Copy `perl-cross` over the source.
pushd ../ && tar xzf perl-cross-1.3.5.tar.gz && popd && \
cp -rf ../perl-cross-1.3.5/*       ./                && \
cp -rf ../perl-cross-1.3.5/utils/* utils/            && \
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

### `18` - Pigz
> #### `2.6` or newer
> The Pigz package contains parallel implementation of gzip, is a fully functional replacement for GNU zip that exploits multiple processors and multiple cores to the hilt when compressing data.

> **Required!** As default ".gz" files de/compressor for the current and next stage (chroot environment).
```bash
# Build.
time { make CC=${CC} CFLAGS="$CFLAGS"; }

# Install and create symlinks as `gzip` tools.
ln -sfv pigz unpigz; ln -sv pigz gzip; ln -sv unpigz gunzip
install -vm755 -t /clang1-tools/bin/ pigz unpigz gzip gunzip
```

### `19` - libffi
> #### `3.3` or newer
> The libffi library provides a portable, high level programming interface to various calling conventions. This allows a programmer to call any function specified by a call interface description at run time.

> **Required!** By Python3 in the current stage.
```bash
# Apply patches (from Void Linux) to fix some issues.
patch -Np1 -i ../../extra/libffi/patches/libffi-race-condition.patch
patch -Np1 -i ../../extra/libffi/patches/no-toolexeclibdir.patch

# Configure source.
./configure \
    --prefix=/clang1-tools       \
    --build=${TARGET_TRUPLE}     \
    --host=${TARGET_TRUPLE}      \
    --disable-static             \
    --with-pic                   \
    --disable-multi-os-directory \
    --with-gcc-arch=native

# Build.
time { make; }

# Install.
time { make install; }
```

### `20` - Python3
> #### `3.9.6` or newer
> The Python3 package contains the Python development environment. It is useful for object-oriented programming, writing scripts, prototyping large programs, or developing entire applications.

> **Required!** To build Clang/LLVM and other required packages in the next stage (chroot environment).
```bash
# Apply patch (from Void Linux) to allow compile under musl libc.
patch -Np1 -i ../../extra/python3/patches/musl-find_library.patch

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

### `21` - libuv
> #### `1.41.0` or newer
> The libuv package is a multi-platform support library with a focus on asynchronous I/O.

> **Required!** By Cmake in the current stage.
```bash
# Enable `pthread` and generate the configure script.
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

# Install and unset linker flags.
time { make install && unset LDFLAGS; }
```

### `22` - Cmake
> #### `3.20.5` or newer
> The CMake package contains a modern toolset used for generating Makefiles. It is a successor of the auto-generated configure script and aims to be platform- and compiler-independent. A significant user of CMake is KDE since version 4.

> **Required!** To build Clang/LLVM in the next stage (chroot environment).
```bash
# Disable applications that using Cmake from attempting to install files in "/usr/lib64".
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


### `23` - Xz
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

### `24` - Cleaning Up and Changing Ownership
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
