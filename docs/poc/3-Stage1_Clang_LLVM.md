## `III` Stage-1 Clang/LLVM Toolchain
The purpose of this stage is to build a pure Clang/LLVM toolchain that will be used to build the "Final System" afterwards.

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

# Stage-1 Clang/LLVM Environment.
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

CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
LDFLAGS="-Wl,-O2 -Wl,--as-needed"
export CFLAGS CXXFLAGS LDFLAGS
EOF
source ~/.bashrc
```

### `1` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Configure source.
./configure --prefix=/             \
            --disable-static       \
            --with-malloc=mallocng \
            --enable-optimize=speed

# Build.
time { make; }

# Install and fix a wrong shared object symlink, also create a `ldd` symlink to use to print shared object dependencies.
time {
    make DESTDIR=/clang1-tools install
    ln -sfv libc.so /clang1-tools/lib/ld-musl-x86_64.so.1
    mkdir -v /clang1-tools/bin && \
    ln -sfv ../lib/libc.so /clang1-tools/bin/ldd
}
```
```bash
# Set compiler to the new triplet from Stage-0 Clang/LLVM to use current libc.
sed -e "s|\"${CXX}\"|\"${H_TRIPLET}-clang++\"|" \
    -e "s|\"${CC}\"|\"${H_TRIPLET}-clang\"|" -i ~/.bashrc
source                                          ~/.bashrc
```
```bash
# Configure PATH for dynamic linker.
mkdir -v /clang1-tools/etc && \
cat > /clang1-tools/etc/ld-musl-x86_64.path << "EOF"
/clang1-tools/lib
EOF
```
```bash
# Quick test for the new triplet of Stage-0 Clang/LLVM.
echo "int main(){}" > dummy.c
${CC} ${CFLAGS} dummy.c -v -Wl,--verbose &> dummy.log
${READELF} -l a.out | grep --color=auto "Req.*ter"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang1-tools/lib/ld-musl-x86_64.so.1]

grep --color=auto "ld.lld:.*crt[1in].o" dummy.log

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
time { make LLVM=1 LLVM_IAS=1 mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time { make ARCH=${C_ARCH} LLVM=1 LLVM_IAS=1 HOSTCC=${CC} headers; }

# Remove unnecessary files.
find usr/include \( -name '.*' -o -name 'Makefile' \) -exec rm -rfv {} \;

# Install.
cp -rfv usr/include /clang1-tools/.
```
```bash
# I don't know .. why if HOSTCC is defaults since use "LLVM=1" (clang),
# it will fails to compile "scripts/basic/fixdep.c", but successful using `${TARGET_TRIPLET}-compiler`.
```

### `3` - Zlib-ng
> #### `2.0.5` or newer
> The Zlib-ng package contains zlib data compression library for the next generation systems.

> **Required!** By Pigz in the current stage and optionally enabled to build Stage-1 Clang/LLVM.
```bash
# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev    \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools" \
    -DCMAKE_C_FLAGS="-flto=thin $CFLAGS"   \
    -DWITH_NATIVE_INSTRUCTIONS=YES         \
    -DWITH_SANITIZER=ON                    \
    -DZLIB_COMPAT=ON                       \
    -DBUILD_SHARED_LIBS=ON

# Build.
time { make -C build; }

# Install.
time { make -C build install; }
```

### `4` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** To build Stage-1 Clang/LLVM and for the most programs that depends on `-ltinfo` or `-lterminfo` linker's flags.
```bash
# Build.
time { make CFLAGS="-fPIC -flto=thin $CFLAGS" all-dynamic; }

# Install.
time { make PREFIX=/clang1-tools install-dynamic; }
```

### `5` - libexecinfo
> #### `1.1` or newer (from Heiwa/Linux fork)
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** To build Stage-1 Clang/LLVM, since using musl libc.
```bash
# Build.
time { make CFLAGS="-flto=thin $CFLAGS" dynamic; }

# Install.
time { make PREFIX=/clang1-tools install-{header,dynamic}; }
```

### `6` - Clang/LLVM + libunwind, libcxxabi, and libcxx
> #### `12.x.x` or newer
> - C language family frontend for LLVM;  
> - C++ runtime stack unwinder from LLVM;  
> - Low level support for a standard C++ library from LLVM;  
> - New implementation of the C++ standard library, targeting C++11 from LLVM.

> **Required!** Build Stage-1 Clang/LLVM self-hosted toolchain with GCC libraries-free since compiled with Stage-0 Clang/LLVM itself.
```bash
# Exit from the LLVM source directory if already entered after decompressing.
popd
```
```bash
# Rename the LLVM source directory to "$LLVM_SRC", then enter.
mv -fv llvm-12.0.1.src "$LLVM_SRC" && pushd "$LLVM_SRC"

# Decompress `clang`, `lld`, `compiler-rt`, `libunwind`, `libcxxabi`, and `libcxx` to the correct directories.
pushd ${LLVM_SRC}/projects/ && \
    tar xf ../../pkgs/compiler-rt-12.0.1.src.tar.xz && mv -fv compiler-rt-12.0.1.src compiler-rt
    tar xf ../../pkgs/libunwind-12.0.1.src.tar.xz   && mv -fv libunwind-12.0.1.src libunwind
    tar xf ../../pkgs/libcxxabi-12.0.1.src.tar.xz   && mv -fv libcxxabi-12.0.1.src libcxxabi
    tar xf ../../pkgs/libcxx-12.0.1.src.tar.xz      && mv -fv libcxx-12.0.1.src libcxx
popd
pushd ${LLVM_SRC}/tools/ && \
    tar xf ../../pkgs/clang-12.0.1.src.tar.xz && mv -fv clang-12.0.1.src clang
    tar xf ../../pkgs/lld-12.0.1.src.tar.xz   && mv -fv lld-12.0.1.src lld
popd

# Apply patches (from Void Linux).
../extra/llvm/patches/appatch

# Disable sanitizers for musl, it's broken since it duplicates some libc bits.
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Update config.guess for better platform detection.
cp -fv ../extra/llvm/files/config.guess cmake/.
```
```bash
# Configure `libunwind` source.
pushd ${LLVM_SRC}/projects/libunwind/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"             \
        -DCMAKE_C_FLAGS="-fPIC -flto=thin -g0 $CFLAGS"     \
        -DCMAKE_CXX_FLAGS="-fPIC -flto=thin -g0 $CXXFLAGS" \
        -DLLVM_PATH="$LLVM_SRC"                            \
        -DLIBUNWIND_ENABLE_STATIC=OFF                      \
        -DLIBUNWIND_USE_COMPILER_RT=ON

# Build.
time { make -C build; }

# Install, also the headers.
time {
    make -C build install                      && \
    cp -fv include/*.h /clang1-tools/include/. && popd
}
```
```bash
# Configure `libcxxabi` source.
pushd ${LLVM_SRC}/projects/libcxxabi/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"       \
        -DCMAKE_CXX_FLAGS="-flto=thin -g0 $CXXFLAGS" \
        -DLLVM_PATH="$LLVM_SRC"                      \
        -DLIBCXXABI_ENABLE_STATIC=OFF                \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON             \
        -DLIBCXXABI_USE_COMPILER_RT=ON               \
        -DLIBCXXABI_LIBCXX_INCLUDES="${LLVM_SRC}/projects/libcxx/include"

# Build.
time { make -C build; }

# Install, also the headers.
time {
    make -C build install                      && \
    cp -fv include/*.h /clang1-tools/include/. && popd
}
```
```bash
# Configure `libcxx` source.
pushd ${LLVM_SRC}/projects/libcxx/ && \
    cmake -B build \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                                      \
        -DCMAKE_CXX_FLAGS="-isystem /clang1-tools/include -flto=thin -g0 $CXXFLAGS" \
        -DLLVM_PATH="$LLVM_SRC"                                                     \
        -DLIBCXX_ENABLE_STATIC=OFF                                                  \
        -DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF                                    \
        -DLIBCXX_CXX_ABI=libcxxabi                                                  \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/clang1-tools/include"                      \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/clang1-tools/lib"                           \
        -DLIBCXX_HAS_MUSL_LIBC=ON                                                   \
        -DLIBCXX_USE_COMPILER_RT=ON                                                 \
        -DLIBCXX_HAS_ATOMIC_LIB=OFF

# Build.
time { make -C build; }

# Install.
time { make -C build install && popd; }
```
```bash
# Remove `libunwind`, `libcxxabi`, and `libcxx` from the LLVM source.
rm -rf projects/lib{unwind,cxx{abi,}}
```
```bash
# Configure Clang/LLVM source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev       \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"    \
    -DCMAKE_C_FLAGS="-g0 $CFLAGS"             \
    -DCMAKE_CXX_FLAGS="-g0 $CXXFLAGS"         \
    -DLLVM_HOST_TRIPLE="$T_TRIPLET"           \
    -DLLVM_DEFAULT_TARGET_TRIPLE="$T_TRIPLET" \
    -DLLVM_ENABLE_BINDINGS=OFF                \
    -DLLVM_ENABLE_EH=ON                       \
    -DLLVM_ENABLE_IDE=OFF                     \
    -DLLVM_ENABLE_LIBCXX=ON                   \
    -DLLVM_ENABLE_LLD=ON                      \
    -DLLVM_ENABLE_RTTI=ON                     \
    -DLLVM_ENABLE_BACKTRACES=OFF              \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF           \
    -DLLVM_ENABLE_WARNINGS=OFF                \
    -DLLVM_ENABLE_LIBEDIT=OFF                 \
    -DLLVM_ENABLE_LIBXML2=OFF                 \
    -DLLVM_ENABLE_OCAMLDOC=OFF                \
    -DLLVM_ENABLE_Z3_SOLVER=OFF               \
    -DLLVM_ENABLE_LTO=Thin                    \
    -DLLVM_INCLUDE_BENCHMARKS=OFF             \
    -DLLVM_INCLUDE_EXAMPLES=OFF               \
    -DLLVM_INCLUDE_TESTS=OFF                  \
    -DLLVM_INCLUDE_GO_TESTS=OFF               \
    -DLLVM_INCLUDE_DOCS=OFF                   \
    -DLLVM_INSTALL_BINUTILS_SYMLINKS=ON       \
    -DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON        \
    -DLLVM_INSTALL_UTILS=ON                   \
    -DLLVM_LINK_LLVM_DYLIB=ON                 \
    -DLLVM_OPTIMIZED_TABLEGEN=ON              \
    -DLLVM_TARGET_ARCH="$L_TARGET"            \
    -DLLVM_TARGETS_TO_BUILD="$L_TARGET"       \
    -DCLANG_VENDOR="Heiwa/Linux"              \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++         \
    -DCLANG_DEFAULT_RTLIB=compiler-rt         \
    -DCLANG_DEFAULT_LINKER=lld                \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind       \
    -DDEFAULT_SYSROOT="/clang1-tools"

# Build.
time { make -C build; }

# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang1-tools" -P cmake_install.cmake && \
    popd
}
```
```bash
# Configure Stage-1 Clang/LLVM with default triplet (pc) to produce binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1".
ln -sfv clang              /clang1-tools/bin/${T_TRIPLET}-clang
ln -sfv ${T_TRIPLET}-clang /clang1-tools/bin/cc
ln -sfv clang++            /clang1-tools/bin/${T_TRIPLET}-clang++
cat > /clang1-tools/bin/${T_TRIPLET}.cfg << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF

# Set the new PATH since "/clang0-tools" won't be used anymore and the Stage-1 Clang/LLVM default triplet (pc).
sed -e 's|/clang0-tools/usr/bin:/clang0-tools/bin:||' \
    -e "s|\"${CXX}\"|\"${T_TRIPLET}-clang++\"|"       \
    -e "s|\"${CC}\"|\"${T_TRIPLET}-clang\"|" -i ~/.bashrc
source                                          ~/.bashrc
```
```bash
# Back to "${HEIWA}/sources/pkgs" directory.
popd
```

### `7` - Xz
> #### `5.2.5` or newer
> The Xz package contains programs for compressing and decompressing files. It provides capabilities for the lzma and the newer xz compression formats. Compressing text files with xz yields a better compression percentage than with the traditional gzip or bzip2 commands.

> **Required!** As default ".xz" and ".lzma" files de/compressor for the current and next stage (chroot environment).
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}    \
            --disable-doc          \
            --disable-static

# Build.
time { make; }

# Install.
time { make install; }
```

### `8` - Pigz
> #### `2.6` or newer
> The Pigz package contains parallel implementation of gzip, is a fully functional replacement for GNU zip that exploits multiple processors and multiple cores to the hilt when compressing data.

> **Required!** As default ".gz" files de/compressor for the current and next stage (chroot environment).
```bash
# Make sure to use symlink instead of hardlink for `unpigz`.
sed -i 's|ln -f|ln -sf|' Makefile

# Build.
time { make CC=${CC} CFLAGS="-flto=thin $CFLAGS"; }

# Install and create symlinks as `gzip` tools.
ln -sfv pigz gzip; ln -sfv unpigz gunzip
install -vm755 -t /clang1-tools/bin/ pigz unpigz gzip gunzip
```

### `9` - Gettext-tiny
> #### `0.3.2` or newer
> The Gettext-tiny package contains utilities for internationalization and localization. These allow programs to be compiled with NLS (Native Language Support), enabling them to output messages in the user's native language. A lightweight replacements for tools typically used from the GNU gettext suite, which is incredibly bloated and takes a lot of time to build (in the order of an hour on slow devices).

> **Required!** To allow programs compiled with NLS (Native Language Support) in the next stage (chroot environment).
```bash
# Build.
time { make LIBINTL=MUSL CFLAGS="-flto=thin $CFLAGS" prefix=/clang1-tools; }

# Only install `msgfmt`, `msgmerge` and `xgettext`.
install -vm755 -t /clang1-tools/bin/ msg{fmt,merge} xgettext 
```

### `10` - Toybox (Coreutils, File, Findutils, Grep, Sed, Tar)
> #### `0.8.5`
> The Toybox package contains "portable" utilities for showing and setting the basic system characteristics.

> **Required!** For the current and next stage (chroot environment).
```bash
# Copy the Toybox .config file.
cp -v ../../extra/toybox/files/.config.toolchain.nolibcrypto .config

# Make sure to enable `libz`.
grep --color=auto "LIBZ" .config

# Export commands as TOYBOX variable that will be use to verify Toybox .config.
read -rd '' TOYBOX << "EOF"
base64 base32 basename cat chgrp chmod chown chroot cksum comm cp cut date
dd df dirname du echo env expand expr factor false fmt fold groups head hostid
id install link ln logname ls md5sum mkdir mkfifo mknod mktemp mv nice nl nohup
nproc od paste printenv printf pwd readlink realpath rm rmdir seq sha1sum shred
sleep sort split stat stty sync tac tail tee test timeout touch tr true truncate
tty uname uniq unlink wc who whoami yes file find xargs egrep grep fgrep sed tar
EOF

# Verify 87 commands, and make sure is enabled (=y).
# Pipe to ` | wc -l` at the right of `done` to checks total of commands.
for X in ${TOYBOX}; do
    grep -v '#' .config | grep -i --color=auto "_${X}=" \
    || echo "* ${X} not CONFIGURED"
done

# Build with verbose. Toybox will use `cc` that break the builds, so need to specify the CC and HOSTCC variable.
time { make CC=${CC} HOSTCC=${CC} CFLAGS="-flto=thin $CFLAGS" V=1; }

# Verify compiled 87 commands.
./toybox | tr ' ' '\n'i \
| grep -xE --color=auto $(echo ${TOYBOX} | tr ' ' '|'i) | wc -l

# Verify commands that not configured but compiled.
# `[` (coreutils)
./toybox | tr ' ' '\n'i \
| grep -vxE --color=auto $(echo ${TOYBOX} | tr ' ' '|'i)

# So, totally is 88 commands.
./toybox | wc -w

# Install and unset exported commands in TOYBOX variable.
time { make CC=${CC} HOSTCC=${CC} PREFIX=/clang1-tools install; unset X TOYBOX; }
```

### `11` - GNU AWK
> #### `5.1.0` or newer
> The GNU AWK (gawk) package contains programs for manipulating text files.

> **Required!** For the current and next stage (chroot environment) that most build systems depends on GNU implementation style.
```bash
# Ensure some unneeded files are not installed.
sed -i 's|extras||' Makefile.in

# Configure source.
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}

# Build.
time { make; }

# Install.
time { make install; }
```

### `12` - GNU Diffutils
> #### `3.7` or newer
> The GNU Diffutils package contains programs that show the differences between files or directories.

> **Required!** For the current and next stage (chroot environment) that most build systems depends on GNU implementation style.
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}    \
            --with-packager="Heiwa/Linux"

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
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}    \
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
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}

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
CFLAGS="-flto=thin $CFLAGS"                   \
./configure --prefix=/clang1-tools            \
            --build=${T_TRIPLET}              \
            --host=${T_TRIPLET}               \
            --disable-perl-xs                 \
            --without-external-libintl-perl   \
            --without-external-Text-Unidecode \
            --without-external-Unicode-EastAsianWidth

# Build.
time { make; }

# Install.
time { make install; }
```

### `16` - GNU Bash
> #### `5.1` (with patch level 8) or newer
> The GNU Bash package contains the Bourne-Again SHell.

> **Required!** As default shell for the next stage (chroot environment).
```bash
# Configure source.
CFLAGS="-flto=thin $CFLAGS"        \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}    \
            --disable-profiling    \
            --without-bash-malloc  \
            --enable-net-redirections

# Build.
time { make; }

# Install.
time { make install; }
```

### `17` - Perl (+ cross)
> #### `5.34.0` (and `1.3.6` for cross)
> The Perl package contains the Practical Extraction and Report Language.

> **Required!** To build required packages in the next stage (chroot environment). 
```bash
# Decompress, then copy `perl-cross` over the source.
tar xzf ../perl-cross-1.3.6.tar.gz && \
cp -af ./perl-cross-1.3.6/* .

# Configure source.
CFLAGS="-DNO_POSIX_2008_LOCALE -D_GNU_SOURCE $CFLAGS" \
HOSTLDFLAGS="-pthread" HOSTCFLAGS="-D_GNU_SOURCE"     \
LDFLAGS="-Wl,-z,stack-size=2097152 -pthread $LDFLAGS" \
./configure --prefix=/clang1-tools                    \
            --build=${T_TRIPLET}                      \
            --target=${T_TRIPLET}

# Build. Fails with LTO. This will display a lot of compiler warnings.
time { make; }

# Install.
time { make install; }
```

### `18` - Python3
> #### `3.9.6` or newer
> The Python3 package contains the Python development environment. It is useful for object-oriented programming, writing scripts, prototyping large programs, or developing entire applications.

> **Required!** To build Clang/LLVM and other required packages in the next stage (chroot environment).
```bash
# Apply patch (from Void Linux) to allow compile under musl libc.
patch -Np1 -i ../../extra/python3/patches/musl-find_library.patch

# Prevent main script that uses hard-coded paths to the host "/usr/include" and "/usr/lib" directories.
sed -i '/def add_multiarch_paths/a \        return' setup.py

# Use ThinLTO rather than Full LTO to speedup build.
sed -i 's|-flto|-flto=thin|' configure

# Configure source using provided libraries (built-in).
ax_cv_c_float_words_bigendian=no   \
./configure --prefix=/clang1-tools \
            --build=${T_TRIPLET}   \
            --host=${T_TRIPLET}    \
            --without-ensurepip    \
            --enable-shared --with-lto

# Build. -> Ignore all issues at the end! <-
time { make; }

# Install.
time { make install; }
```

### `19` - Cmake
> #### `3.20.5` or newer
> The CMake package contains a modern toolset used for generating Makefiles. It is a successor of the auto-generated configure script and aims to be platform- and compiler-independent. A significant user of CMake is KDE since version 4.

> **Required!** To build Clang/LLVM in the next stage (chroot environment).
```bash
# Disable applications that using Cmake from attempting to install files in "/usr/lib64".
sed -i '/"lib64"/s/64//' Modules/GNUInstallDirs.cmake

# Configure source using provided libraries (built-in).
CFLAGS="-flto=thin $CFLAGS"                  \
CXXFLAGS="-flto=thin $CXXFLAGS"              \
./bootstrap --prefix=/clang1-tools           \
            --mandir=/share/man              \
            --parallel=$(nproc)              \
            --docdir=/share/doc/cmake-3.20.5 \
            -- -DCMAKE_BUILD_TYPE=Release    \
            -Wno-dev -DCMAKE_USE_OPENSSL=OFF

# Build.
time { make; }

# Install.
time { make install; }
```

### `20` - Cleaning Up and Changing Ownership
> **This section is optional!**

> If the intended user is not a programmer and does not plan to do any debugging on the system software, the system size can be decreased by removing the debugging symbols from binaries and libraries. This causes no inconvenience other than not being able to debug the software fully anymore.
```bash
# The libtool .la files are only useful when linking with static libraries.
# They are unneeded, and potentially harmful, when using dynamic shared libraries, specially when using non-autotools build systems.
# Remove those files.
find /clang1-tools/{lib,libexec}/ -name '*.la' -exec rm -rfv {} \;

# Remove the documentation and manpages.
rm -rf /clang1-tools/share/{info,man,doc}/*

# Strip off debugging symbols from binaries using `llvm-strip`.
# A large number of files will be reported "The file was not recognized as a valid object file".
# These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
find /clang1-tools/lib/ -type f \( -name '*.a' -o -name '*.so*' \) -exec llvm-strip --strip-debug {} \;
find /clang1-tools/{{,usr/}{,s}bin,libexec/awk}/ -maxdepth 1 -type f -exec /clang0-tools/bin/llvm-strip --strip-unneeded {} \;
```
```bash
# Exit from privileged user.
exit
```
> #### * End of as privileged user!

> #### * Beginning of as root!
```bash
# Change the ownership of the "${HEIWA}/clang1-tools" directory to root by running the following command.
# Warning! This is danger, so check its variables before `chown`.
# echo ${HEIWA}/clang1-tools
[[ -n "$HEIWA" ]] && chown -R root:root ${HEIWA}/clang1-tools
```
> #### * End of as root!

<h2></h2>

~Continue to [Final System](./4-Final_System.md).~ (under developments)
