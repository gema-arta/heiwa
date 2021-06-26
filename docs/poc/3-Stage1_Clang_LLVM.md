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
/clang1-tools/gnu/lib
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
```sh
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
> Required as LLVM's C++ standard library.
```sh
# Configure source.
pushd "${LLVM_SRC}/projects/libcxxabi/" && \
cmake -B build \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"                           \
    -DLIBCXXABI_ENABLE_STATIC=ON                                     \
    -DLIBCXXABI_USE_COMPILER_RT=ON                                   \
    -DLIBCXXABI_USE_LLVM_UNWINDER=ON                                 \
    -DLIBCXXABI_LIBUNWIND_PATH="/clang1-tools/lib"                   \
    -DLIBCXXABI_LIBCXX_INCLUDES="${LLVMSRC}/projects/libcxx/include" \
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
