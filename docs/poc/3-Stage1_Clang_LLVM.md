## `III` Stage-1 Clang/LLVM Toolchains
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
> Required for every programs and libraries.
```bash
# Set default compiler to Stage-0 Clang/LLVM.
CC=clang CXX=clang++
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
"${HEIWA_TARGET}-clang" dummy.c -v -Wl,--verbose &> dummy.log
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
> Required for runtime C library (musl) to use Linux API.
```bash
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time {
    [[ -n "$HEIWA_ARCH" && "$HEIWA_TARGET" ]] && \
    make ARCH="$HEIWA_ARCH" LLVM=1 HOSTCC="${HEIWA_TARGET}-clang" headers_check && \
    make ARCH="$HEIWA_ARCH" LLVM=1 HOSTCC="${HEIWA_TARGET}-clang" headers
}

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
cp -rv usr/include /clang1-tools/.
```

### `3` - libunwind from LLVM
> #### `12.0.0`
> Required for programs that needs to enable unwinding.

> No need to re-decompress package.
```bash
# Set default compiler to new symlink from Stage-0 Clang/LLVM.
CC="${HEIWA_TARGET}-clang" CXX="${HEIWA_TARGET}-clang++"
export CC CXX

# Configure source.
pushd "${LLVM_SRC}/projects/libunwind/" && \
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

### `4` - libcxxabi from LLVM
> #### `12.0.0`
> Required as LLVM's C++ ABI standard library.

> No need to re-decompress package.
```bash
# Configure source.
pushd "${LLVM_SRC}/projects/libcxxabi/" && \
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

### `5` - libcxx from LLVM
> #### `12.0.0`
> Required as LLVM's C++ standard library.

> No need to re-decompress package.
```bash
# Disable libatomic detection for Linux, since to get a rid of GCC libraries.

# Configure source.
pushd "${LLVM_SRC}/projects/libcxx/" && \
    cmake -B build  \
        -DCMAKE_INSTALL_PREFIX="/clang1-tools"                 \
        -DLIBCXX_ENABLE_SHARED=ON                              \
        -DLIBCXX_ENABLE_STATIC=ON                              \
        -DLIBCXX_HAS_MUSL_LIBC=ON                              \
        -DLIBCXX_USE_COMPILER_RT=ON                            \
        -DLIBCXX_CXX_ABI=libcxxabi                             \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/clang1-tools/include" \
        -DLIBCXXABI_USE_LLVM_UNWINDER=ON                       \
        -DLIBCXX_CXX_ABI_LIBRARY_PATH="/clang1-tools/lib"      \
        -DLIBCXX_INSTALL_HEADERS=ON                            \
        -DCMAKE_CXX_FLAGS="-isystem /clang1-tools/include"     \
        -DLLVM_PATH="$LLVM_SRC"

# Build.
time { make -C build; }

# Install.
time { make -C build install && rm -rf build && popd; }
```
> #### ^ NOTE!
> Now, you can safely remove ${LLVM_SRC} directory.

### `6` - NetBSD's Curses
> #### `0.3.2` or newer
> Required to build Stage-1 Clang/LLVM that depends on `-ltinfo` or `-lterminfo` ld's flags.
```bash
# Build.
time { make CC="${HEIWA_TARGET}-clang" CFLAGS="$COMMON_FLAGS -Wall -fPIC" all; }

# Install.
time { make PREFIX=/ DESTDIR=/clang1-tools install; }
```

### `7` - libexecinfo
> #### `1.1` or newer
> Required to build Stage-1 Clang/LLVM.
```bash
# Apply patches (from Alpine Linux).
patch -Np1 -i ../../patches/libexecinfo-1.1/10-execinfo.patch
patch -Np1 -i ../../patches/libexecinfo-1.1/20-define-gnu-source.patch
patch -Np1 -i ../../patches/libexecinfo-1.1/30-linux-makefile.patch

# Build.
time { make CC=clang AR=llvm-ar CFLAGS="$COMMON_FLAGS -fno-omit-frame-pointer"; }

# Install.
install -vm755 -t /clang1-tools/include/ execinfo.h stacktraverse.h
install -vm755 -t /clang1-tools/lib/ libexecinfo.a libexecinfo.so.1
ln -sv libexecinfo.so.1 /clang1-tools/lib/libexecinfo.so
```


### `8` - Clang/LLVM
> #### `12.0.0`
> Bootstrapping Stage-1 Clang/LLVM toolchains with `libgcc_s.so*` and `libstdc++.so*` free.
```bash
# Rename the llvm source directory to ${LLVM_SRC}.
popd; mv -v llvm-12.0.0.src "$LLVM_SRC" && pushd "$LLVM_SRC"

# Decompress clang, lld, and compiler-rt to correct directories.
pushd "${LLVM_SRC}/projects/" && \
    tar xf ../../pkgs/compiler-rt-12.0.0.src.tar.xz && mv -v compiler-rt-12.0.0.src compiler-rt
popd

pushd "${LLVM_SRC}/tools/" && \
    tar xf ../../pkgs/clang-12.0.0.src.tar.xz && mv -v clang-12.0.0.src clang
    tar xf ../../pkgs/lld-12.0.0.src.tar.xz   && mv -v lld-12.0.0.src lld
popd

# Apply patches (from Void Linux).
../patches/llvm-12/stage1-appatch

# Disable sanitizers for musl, fixing "early build failure".
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Set default compiler to new symlink from Stage-0 Clang/LLVM.
# Sets C and C++ compiler's build flags to reduce debug's symbols.
CC="${HEIWA_TARGET}-clang" CXX="${HEIWA_TARGET}-clang++"
CFLAGS="-g -g1" CXXFLAGS="-g -g1"
export CC CXX CFLAGS CXXFLAGS

# Update host/target triple detection.
cp -v ../files/config.guess cmake/

# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release                                  \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"                      \
    -DLLVM_INSTALL_TOOLCHAIN_ONLY=ON                            \
    -DLLVM_LINK_LLVM_DYLIB=ON                                   \
    -DLLVM_BUILD_LLVM_DYLIB=ON                                  \
    -DLLVM_BUILD_TESTS=OFF                                      \
    -DLLVM_ENABLE_LIBEDIT=OFF                                   \
    -DLLVM_ENABLE_LIBXML2=OFF                                   \
    -DLLVM_ENABLE_LIBCXX=ON                                     \
    -DLLVM_INCLUDE_GO_TESTS=OFF                                 \
    -DLLVM_INCLUDE_TESTS=OFF                                    \
    -DLLVM_INCLUDE_DOCS=OFF                                     \
    -DLLVM_INCLUDE_EXAMPLES=OFF                                 \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                               \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"         \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-musl"                   \
    -DLLVM_TARGET_ARCH="X86"                                    \
    -DLLVM_TARGETS_TO_BUILD="X86"                               \
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
    -DLLVM_ENABLE_LLD=ON                                        \
    -DLLVM_ENABLE_RTTI=ON                                       \
    -DLLVM_ENABLE_ZLIB=OFF                                      \
    -DBacktrace_INCLUDE_DIR="/clang1-tools/include"             \
    -DBacktrace_LIBRARY="/clang1-tools/lib/libexecinfo.so"      \
    -DCMAKE_CXX_COMPILER_AR="/clang0-tools/bin/llvm-ar"         \
    -DCMAKE_C_COMPILER_AR="/clang0-tools/bin/llvm-ar"           \
    -DCMAKE_CXX_COMPILER_RANLIB="/clang0-tools/bin/llvm-ranlib" \
    -DCMAKE_C_COMPILER_RANLIB="/clang0-tools/bin/llvm-ranlib"   \
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
# But some binaries is not installed, make sure to install important binaries.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang1-tools" -P cmake_install.cmake && \
        cp -v bin/llvm-as /clang1-tools/bin/                                && \
        cp -v bin/llvm-readobj /clang1-tools/bin/                           && \
        ln -sv llvm-readobj /clang1-tools/bin/llvm-readelf                  && \
    popd && rm -rf build
}

# Since Binutils won't be used, create a symlink to the LLVM counterparts.
for B in as ar ranlib readelf nm objcopy objdump size strip; do
    ln -sv llvm-${B} /clang1-tools/bin/${B}
done

# Set lld as default toolchain linker.
ln -sv lld /clang1-tools/bin/ld

# Configure Stage-1 Clang to build binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1" instead of "/lib/ld-musl-x86_64.so.1".
ln -sv clang-12 /clang1-tools/bin/x86_64-pc-linux-musl-clang   && \
ln -sv clang-12 /clang1-tools/bin/x86_64-pc-linux-musl-clang++ && \
cat > /clang1-tools/bin/x86_64-pc-linux-musl.cfg << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF

# Unset exported flags and configure new PATH since "/clang0-tools" isn't used anymore.
unset B CFLAGS CXXFLAGS
export PATH="/clang1-tools/bin:/clang1-tools/usr/bin:/bin:/usr/bin"
sed -i "s|PATH=.*|PATH=\"${PATH}\"|" ~/.bashrc && source ~/.bashrc
```
