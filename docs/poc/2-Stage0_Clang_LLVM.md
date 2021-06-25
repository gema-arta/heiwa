## `II` Stage-0 Clang/LLVM (ft. GNU) Cross-Compile Toolchains
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

> **Build Notes!**
> 1. Using GNU's `bash` as current shell and symlink it to `sh`.
> 2. Using GNU's `gawk` as `awk` implementation (symlinked).
> 3. Using GNU's `bison` as `yacc` replacements (wrapped).
> 4. Using `flex` as `lex` alternative lexical analyzers (symlinked).
> 
> ```bash
> file $(command -v {sh,awk,yacc,lex})
> ```

### `1` - Linux API Headers
> #### Xanmod-CacULE, `5.10.x` or newer
> Required for runtime C library (musl) to use Linux API.
```bash
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time {
    [[ -n "$HEIWA_ARCH" ]] && \
    make ARCH="$HEIWA_ARCH" headers_check && \
    make ARCH="$HEIWA_ARCH" headers
}

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
[[ -n "$HEIWA_TARGET" ]] && \
mkdir -pv "/clang0-tools/${HEIWA_TARGET}" && \
cp -rv usr/include "/clang0-tools/${HEIWA_TARGET}/."
```

### `2` - Binutils
> #### `2.36.1` or newer
> Required to build GCC.
```bash
# Create a dedicated directory and configure source.
[[ -n "$HEIWA_TARGET" ]] && mkdir -v build && cd build && \
../configure \
    --prefix=/clang0-tools                         \
    --target="$HEIWA_TARGET"                       \
    --with-sysroot="/clang0-tools/${HEIWA_TARGET}" \
    --disable-nls                                  \
    --disable-multilib                             \
    --disable-werror                               \
    --enable-deterministic-archives                \
    --disable-compressed-debug-sections

# Checks the host's environment and makes sure all the necessary tools are available to compile Binutils.
# Then build.
time { make configure-host && make; }

# Install.
time { make install; }
```

### `3` -  GCC (static)
> #### `10.3.1_git20210424` (from Alpine Linux)
> Required to compile required libraries to build Stage-0 Clang/LLVM.
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf ../gmp-6.2.1.tar.xz  && mv -v gmp-6.2.1 gmp
tar xf ../mpfr-4.1.0.tar.xz && mv -v mpfr-4.1.0 mpfr
tar xzf ../mpc-1.2.1.tar.gz && mv -v mpc-1.2.1 mpc

# Create a dedicated directory and configure source.
[[ -n "$HEIWA_HOST" && "$HEIWA_TARGET" && "$HEIWA_CPU" ]] && \
mkdir -v build && cd build && \
CFLAGS="-g0 -O0" CXXFLAGS="-g0 -O0" ../configure    \
    --prefix=/clang0-tools    --build="$HEIWA_HOST" \
    --host="$HEIWA_HOST"   --target="$HEIWA_TARGET" \
    --with-sysroot="/clang0-tools/${HEIWA_TARGET}"  \
    --disable-nls                     --with-newlib \
    --disable-libitm               --disable-libvtv \
    --disable-libssp               --disable-shared \
    --disable-libgomp             --without-headers \
    --disable-threads            --disable-multilib \
    --disable-libatomic         --disable-libstdcxx \
    --enable-languages=c      --disable-libquadmath \
    --disable-libsanitizer --with-arch="$HEIWA_CPU" \
    --disable-decimal-float --enable-clocale=generic

# Build only the minimum.
time { make all-gcc all-target-libgcc; }

# Install.
time { make install-gcc install-target-libgcc; }
```

### `4` - musl
> #### `1.2.2` or newer
> Required for every programs and libraries.
```bash
# Configure source.
[[ -n "$HEIWA_TARGET" ]] && ./configure \
    CROSS_COMPILE="${HEIWA_TARGET}-"    \
    --prefix=/                          \
    --target="$HEIWA_TARGET"

# Build.
time { make; }

# Install.
time { make DESTDIR=/clang0-tools install; }

# GCC will looking for system headers in "/clang0-tools/usr/include/".
# Create the directory, then symlink it.
mkdir -v /clang0-tools/usr && \
ln -sv ../include /clang0-tools/usr/include

# Fix a wrong object symlink.
ln -sfv libc.so /clang0-tools/lib/ld-musl-x86_64.so.1

# Create a ldd symlink to use to print shared object dependencies.
ln -sv ../lib/ld-musl-x86_64.so.1 /clang0-tools/bin/ldd

# Configure PATH for dynamic linker.
mkdir -v /clang0-tools/etc && \
cat > /clang0-tools/etc/ld-musl-x86_64.path << "EOF"
/clang0-tools/lib
/clang0-tools/x86_64-heiwa-linux-musl/lib
/usr/lib
/lib
EOF
```

### `5` -  GCC (final)
> #### `10.3.1_git20210424` (from Alpine Linux)
> Required to compile required libraries to build Stage-0 Clang/LLVM.
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf ../gmp-6.2.1.tar.xz  && mv -v gmp-6.2.1 gmp
tar xf ../mpfr-4.1.0.tar.xz && mv -v mpfr-4.1.0 mpfr
tar xzf ../mpc-1.2.1.tar.gz && mv -v mpc-1.2.1 mpc

# Apply patches (from Alpine Linux).
../../patches/gcc-10.3.1_git20210424/appatch

# On x86_64 hosts, set the default directory name for 64-bit libraries to "lib".
case $(uname -m) in
    x86_64) sed -e '/m64=/s/lib64/lib/' \
            -i.orig gcc/config/i386/t-linux64
    ;;
esac

# Create a dedicated directory and configure source.
[[ -n "$HEIWA_HOST" && "$HEIWA_TARGET" ]] && \
mkdir -v build && cd build && \
AR=ar LDFLAGS="-Wl,-rpath,/clang0-tools/lib" \
../configure \
    --prefix=/clang0-tools        \
    --build="$HEIWA_HOST"         \
    --host="$HEIWA_HOST"          \
    --target="$HEIWA_TARGET"      \
    --disable-multilib            \
    --with-sysroot=/clang0-tools  \
    --disable-nls                 \
    --enable-shared               \
    --enable-languages=c,c++      \
    --enable-threads=posix        \
    --enable-clocale=generic      \
    --enable-libstdcxx-time       \
    --enable-fully-dynamic-string \
    --disable-symvers             \
    --disable-libsanitizer        \
    --disable-lto-plugin          \
    --disable-libssp

# Build.
time {
    [[ -n "$HEIWA_TARGET" ]] && \
    make AS_FOR_TARGET="${HEIWA_TARGET}-as" LD_FOR_TARGET="${HEIWA_TARGET}-ld"
}

# Install.
time { make install; }

# Adjust GCC to produce programs and libraries that will use musl libc in "/clang0-tools/".
[[ -n "$HEIWA_TARGET" ]] && \
export SPECFILE="$(dirname $(${HEIWA_TARGET}-gcc -print-libgcc-file-name))/specs"
"${HEIWA_TARGET}-gcc" -dumpspecs > specs
sed -i 's|/lib/ld-musl-x86_64.so.1|/clang0-tools/lib/ld-musl-x86_64.so.1|g' specs

# Check specs file.
grep --color=auto "/clang0-tools/lib/ld-musl-x86_64.so.1" specs

# Install specs file.
mv -v specs "$SPECFILE" && unset SPECFILE

# Quick test.
echo "int main(){}" > dummy.c
"${HEIWA_TARGET}-gcc" dummy.c
readelf -l a.out | grep Requesting

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang0-tools/lib/ld-musl-x86_64.so.1]
```

### `6` - Toybox's File
> #### `0.8.5`
> Optional?
```sh
# Copy toybox's .config file.
cp -v ../../files/toybox-0.8.5/.config_file_no_libz_no_ssl .config

# Build.
time {
    [[ -n "$HEIWA_TARGET" ]] && make \
    CC="${HEIWA_TARGET}-gcc" CFLAGS="$COMMON_FLAGS"
}

# Install.
time { make PREFIX=/clang0-tools install; }
```

### `7` - libexecinfo
> #### `1.1` or newer
> Required to build Stage-0 Clang/LLVM.
```bash
# Apply patches (from Alpine Linux).
patch -Np1 -i ../../patches/libexecinfo-1.1/10-execinfo.patch
patch -Np1 -i ../../patches/libexecinfo-1.1/20-define-gnu-source.patch
patch -Np1 -i ../../patches/libexecinfo-1.1/30-linux-makefile.patch

# Build.
time {
    make CC="${HEIWA_TARGET}-gcc" AR="${HEIWA_TARGET}-ar" \
    CFLAGS="$COMMON_FLAGS -fno-omit-frame-pointer"
}

# Install.
install -vm755 -t /clang0-tools/include/ execinfo.h stacktraverse.h
install -vm755 -t /clang0-tools/lib/ libexecinfo.a libexecinfo.so.1
ln -sv libexecinfo.so.1 /clang0-tools/lib/libexecinfo.so
```

### `8` -  Clang/LLVM
> #### `12.0.0`
> Required to bootstrap Stage-1 Clang/LLVM toolchains without depends on `libgcc_s.so*` later.
```sh
# Rename the llvm source directory to ${LLVM_SRC}.
popd; mv -v llvm-12.0.0.src "$LLVM_SRC" && pushd "$LLVM_SRC"

# Decompress clang, lld, compiler-rt, libcxx, libcxxabi, and libunwind to correct directories.
pushd "${LLVM_SRC}/projects/" && \
    tar xf ../../pkgs/compiler-rt-12.0.0.src.tar.xz && mv -v compiler-rt-12.0.0.src compiler-rt
    tar xf ../../pkgs/libcxx-12.0.0.src.tar.xz      && mv -v libcxx-12.0.0.src libcxx
    tar xf ../../pkgs/libcxxabi-12.0.0.src.tar.xz   && mv -v libcxxabi-12.0.0.src libcxxabi
    tar xf ../../pkgs/libunwind-12.0.0.src.tar.xz   && mv -v libunwind-12.0.0.src libunwind
popd

pushd "${LLVM_SRC}/tools/" && \
    tar xf ../../pkgs/clang-12.0.0.src.tar.xz && mv -v clang-12.0.0.src clang
    tar xf ../../pkgs/lld-12.0.0.src.tar.xz   && mv -v lld-12.0.0.src lld
popd

# Apply patches (from Void Linux).
pushd "${LLVM_SRC}/projects/" && \
    for P in \
        compiler-rt-aarch64-ucontext.patch \
        compiler-rt-sanitizer-ppc64-musl.patch \
        compiler-rt-size_t.patch \
        compiler-rt-xray-ppc64-musl.patch
    do patch -Np1 -i ../../patches/llvm-12/${P}
    done; unset P
    for P in \
        libcxx-musl.patch \
        libcxx-ppc.patch \
        libcxx-ssp-nonshared.patch
    do patch -Np1 -i ../../patches/llvm-12/${P}
    done; unset P
    patch -Np1 -i ../../patches/llvm-12/libunwind-ppc32.patch
popd

pushd "${LLVM_SRC}/tools/" && \
    for P in \
        clang-001-fix-unwind-chain-inclusion.patch \
        clang-002-add-musl-triples.patch \
        clang-003-ppc64-dynamic-linker-path.patch \
        clang-004-ppc64-musl-elfv2.patch
    do patch -Np1 -i ../../patches/llvm-12/${P}
    done; unset P
popd

pushd "${LLVM_SRC}/../" && \
    for P in \
        llvm-001-musl.patch \
        llvm-002-musl-ppc64-elfv2.patch \
        llvm-003-ppc-secureplt.patch \
        llvm-004-override-opt.patch \
        llvm-005-ppc-bigpic.patch \
        llvm-006-aarch64-mf_exec.patch
    do patch  -Np1 -i ./patches/llvm-12/${P}
    done; unset P
popd

# Disable sanitizers for musl, fixing "early build failure".
sed -i 's|set(COMPILER_RT_HAS_SANITIZER_COMMON TRUE)|set(COMPILER_RT_HAS_SANITIZER_COMMON FALSE)|' \
projects/compiler-rt/cmake/config-ix.cmake

# Fix missing header for lld, (https://bugs.llvm.org/show_bug.cgi?id=49228).
mkdir -pv tools/lld/include/mach-o && \
cp -v projects/libunwind/include/mach-o/compact_unwind_encoding.h tools/lld/include/mach-o

# Sets C and C++ compiler's build flags to reduce debug's symbols.
CFLAGS="-g -g1" CXXFLAGS="-g -g1"
export CFLAGS CXXFLAGS

# Update host/target triple detection.
cp -v ../files/config.guess cmake/

# Configure source.
cmake -B build  \
    -DCMAKE_BUILD_TYPE=Release                                                              \
    -DCMAKE_INSTALL_PREFIX="/clang0-tools"                                                  \
    -DCMAKE_C_COMPILER="${HEIWA_TARGET}-gcc"                                                \
    -DCMAKE_CXX_COMPILER="${HEIWA_TARGET}-g++"                                              \
    -DLLVM_BUILD_TESTS=OFF                                                                  \
    -DLLVM_ENABLE_LIBEDIT=OFF                                                               \
    -DLLVM_ENABLE_LIBXML2=OFF                                                               \
    -DLLVM_INCLUDE_GO_TESTS=OFF                                                             \
    -DLLVM_INCLUDE_TESTS=OFF                                                                \
    -DLLVM_INCLUDE_DOCS=OFF                                                                 \
    -DLLVM_INCLUDE_EXAMPLES=OFF                                                             \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                                                           \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"                                     \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-musl"                                               \
    -DLLVM_TARGET_ARCH="X86"                                                                \
    -DLLVM_TARGETS_TO_BUILD="X86"                                                           \
    -DCOMPILER_RT_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl"                              \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF                                                      \
    -DCOMPILER_RT_BUILD_XRAY=OFF                                                            \
    -DCOMPILER_RT_BUILD_PROFILE=OFF                                                         \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                                                       \
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON                                                   \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                                                       \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                                                     \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                                                       \
    -DICONV_LIBRARY_PATH="/clang0-tools/lib/libc.so"                                        \
    -DDEFAULT_SYSROOT="/clang0-tools"                                                       \
    -DLIBCXX_HAS_MUSL_LIBC=ON                                                               \
    -DLLVM_ENABLE_LIBCXX=ON                                                                 \
    -DCLANG_DEFAULT_LINKER="/clang0-tools/bin/ld.lld"                                       \
    -DBacktrace_HEADER="/clang0-tools/include/execinfo.h"                                   \
    -DCMAKE_EXE_LINKER_FLAGS="-Wl,-dynamic-linker /clang0-tools/lib/ld-musl-x86_64.so.1"    \
    -DCMAKE_SHARED_LINKER_FLAGS="-Wl,-dynamic-linker /clang0-tools/lib/ld-musl-x86_64.so.1" \
    -DBacktrace_LIBRARY="/clang0-tools/lib/libexecinfo.so.1"

# Build.
time { make -C build; }

# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang0-tools" -P cmake_install.cmake && \
    popd && rm -rf build
}

# Set lld as default toolchain linker.
ln -sv lld /clang0-tools/bin/ld

# Since Clang/LLVM still have GCC dependencies, add the GCC libraries in the search paths of the toolchain's dynamic linker.
echo "/clang0-tools/${HEIWA_TARGET}/lib" >> /clang0-tools/etc/ld-musl-x86_64.path

# Configure Clang to build binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1" instead of "/lib/ld-musl-x86_64.so.1".
ln -sv clang-12 "/clang0-tools/bin/${HEIWA_TARGET}-clang"
ln -sv clang-12 "/clang0-tools/bin/${HEIWA_TARGET}-clang++"
cat > "/clang0-tools/bin/${HEIWA_TARGET}.cfg" << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF

# Configure cross GCC of clang0-tools to match the same output as Clang.
export SPECFILE="$(dirname $(${HEIWA_TARGET}-gcc -print-libgcc-file-name))/specs"
"${HEIWA_TARGET}-gcc" -dumpspecs > specs 
sed -i 's|/lib/ld-musl-x86_64.so.1|/clang1-tools\/lib\/ld-musl-x86_64.so.1|g' specs

# Check specs file.
grep --color=auto "/clang1-tools/lib/ld-musl-x86_64.so.1" specs

# Install specs file.
mv -v specs "$SPECFILE" && unset SPECFILE

# Quick test.
"${HEIWA_TARGET}-gcc" dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ": /clang1-tools"

# The output should be:
#       [Requesting program interpreter: /clang1-tools/lib/ld-musl-x86_64.so.1]

grep  "lib.*/crt[1in].*succeeded" dummy.log | cut -d ' ' -f 4-5 | cut -b 5-

# The output should be:
# /heiwa/clang0-tools/bin/../../clang0-tools/lib/../lib/crt1.o succeeded
# /heiwa/clang0-tools/bin/../../clang0-tools/lib/../lib/crti.o succeeded
# /heiwa/clang0-tools/bin/../../clang0-tools/lib/../lib/crtn.o succeeded

# Clean up after testing.
rm -v dummy.log dummy.c a.out
```

<h2></h2>

Continue to [Stage-1 Clang/LLVM Toolchains](./3-Stage1_Clang_LLVM.md).
