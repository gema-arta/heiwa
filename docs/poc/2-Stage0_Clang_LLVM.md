## `II` Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain
The purpose of this stage is to build a temporary Clang/LLVM toolchain with GCC libraries which will be used to build Clang/LLVM without relying on GCC because if it build directly using host's Clang/LLVM/GCC it will fail or break. [LLVM Reference](https://clang.llvm.org/docs/Toolchain.html) **•** [CLFS Reference](http://clfs.org/view/clfs-embedded/x86/) **•** [musl faq](https://wiki.musl-libc.org/faq.html)

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

> #### Build Notes
> 0. Using GCC and Binutils as compiler toolchain.
> 1. Using `bash` as current shell and symlink it to `sh`.
> 2. Using `gawk` as `awk` implementation (symlinked).
> 3. Using `bison` as `yacc` replacements (wrapped).
> 4. Using `flex` as `lex` alternative lexical analyzers (symlinked).
> 5. See more at "[LFS Host System Requirements](https://linuxfromscratch.org/lfs/view/stable/chapter02/hostreqs.html)".
> 
> ```bash
> file $(command -v {g{cc,++},sh,awk,yacc,lex})
> ```

### `1` - Linux API Headers
> #### `5.13.x` (CacULE) or newer
> The Linux API Headers expose the kernel's API for use by musl libc.

> **Required!** As mentioned in the description above.
```bash
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target `headers_install` cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in "./usr/", then copied to the needed location.
time { make ARCH=${C_ARCH} headers; }

# Remove unnecessary files.
find usr/include \( -name '.*' -o -name 'Makefile' \) -exec rm -rfv {} \;

# Install.
mkdir -v /clang0-tools/${H_TRIPLET} && \
cp -rfv usr/include /clang0-tools/${H_TRIPLET}/.
```

### `2` - GNU Binutils
> #### `2.36.1` or newer
> The GNU Binutils package contains a linker, an assembler, and other tools for handling object files.

> **Required!** To build Stage-0 Clang/LLVM.
```bash
# Create a dedicated directory and configure source.
mkdir -v build && cd build
CFLAGS="-g0 -Os -pipe" CXXFLAGS="-g0 -Os -pipe" ../configure \
    --prefix=/clang0-tools                                   \
    --target=${H_TRIPLET}                                    \
    --with-sysroot=/clang0-tools/${H_TRIPLET}                \
    --enable-deterministic-archives                          \
    --disable-nls                                            \
    --disable-multilib                                       \
    --disable-werror                                         \
    --disable-compressed-debug-sections

# Checks the host's environment and makes sure all the necessary tools are available to compile Binutils. Then build.
time { make configure-host && make; }

# Install.
time { make install; }
```

### `3` - GCC <kbd>static</kbd>
> #### `10.3.1_git20210424` (from Alpine Linux)
> The GCC package contains the GNU compiler collection, which includes the C and C++ compilers.

> **Required!** This build of GCC is mainly done so that the C library can be built next.
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf ../gmp-6.2.1.tar.xz  && mv -fv gmp-6.2.1 gmp
tar xf ../mpfr-4.1.0.tar.xz && mv -fv mpfr-4.1.0 mpfr
tar xzf ../mpc-1.2.1.tar.gz && mv -fv mpc-1.2.1 mpc

# Create a dedicated directory and configure source.
mkdir -v build && cd build
CFLAGS="-g0 -O0 -pipe" CXXFLAGS="-g0 -O0 -pipe" ../configure \
    --prefix=/clang0-tools                                   \
    --build=${C_TRIPLET}                                     \
    --host=${C_TRIPLET}                                      \
    --target=${H_TRIPLET}                                    \
    --with-sysroot=/clang0-tools/${H_TRIPLET}                \
    --disable-nls                                            \
    --disable-shared                                         \
    --without-headers                                        \
    --with-newlib                                            \
    --disable-decimal-float                                  \
    --disable-libgomp                                        \
    --disable-libssp                                         \
    --disable-libatomic                                      \
    --disable-libquadmath                                    \
    --disable-threads                                        \
    --disable-libitm                                         \
    --disable-libsanitizer                                   \
    --disable-libstdcxx                                      \
    --disable-libvtv                                         \
    --enable-languages=c                                     \
    --enable-clocale=generic                                 \
    --disable-multilib                                       \
    --with-arch=${C_CPU}

# Build only the minimum.
time { make all-gcc all-target-libgcc; }

# Install.
time { make install-gcc install-target-libgcc; }
```

### `4` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
```bash
# Configure source.
CFLAGS="-Os -pipe" CXXFLAGS="-Os -pipe" ./configure \
    CROSS_COMPILE=${H_TRIPLET}-                     \
    --prefix=/                                      \
    --target=${H_TRIPLET}                           \
    --disable-static                                \
    --with-malloc=mallocng                          \
    --enable-optimize=speed

# Build.
time { make; }

# Install and fix a wrong shared object symlink, also create a `ldd` symlink to use to print shared object dependencies.
time {
    make DESTDIR=/clang0-tools install
    ln -sfv libc.so /clang0-tools/lib/ld-musl-x86_64.so.1
    ln -sfv ../lib/libc.so /clang0-tools/bin/ldd
}
```
```bash
# Configure PATH for dynamic linker.
mkdir -v /clang0-tools/etc && \
cat > /clang0-tools/etc/ld-musl-x86_64.path << EOF
/clang0-tools/lib
/clang0-tools/${H_TRIPLET}/lib
/usr/lib
/lib
EOF
```
```bash
# GCC will looking for system headers in "/clang0-tools/usr/include/*".
# Create the directory, then symlink it.
mkdir -v /clang0-tools/usr && \
ln -sv ../include /clang0-tools/usr/include
```

### `5` - GCC <kbd>final</kbd>
> #### `10.3.1_git20210424` (from Alpine Linux)
> The GCC package contains the GNU compiler collection, which includes the C and C++ compilers.

> **Required!** This second build of GCC will produce the final cross compiler which will use the previously built C library.
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf ../gmp-6.2.1.tar.xz  && mv -fv gmp-6.2.1 gmp
tar xf ../mpfr-4.1.0.tar.xz && mv -fv mpfr-4.1.0 mpfr
tar xzf ../mpc-1.2.1.tar.gz && mv -fv mpc-1.2.1 mpc

# Apply patches (from Alpine Linux).
../../extra/gcc/patches/appatch

# On x86_64 hosts, set the default directory name for 64-bit libraries to "lib".
case $(uname -m) in
    x86_64) sed -e '/m64=/s/lib64/lib/' -i.orig gcc/config/i386/t-linux64
    ;;
esac

# Create a dedicated directory and configure source.
mkdir -v build && cd build
CFLAGS="-g0 -Os -pipe" CXXFLAGS="-g0 -Os -pipe"     \
LDFLAGS="-Wl,-rpath,/clang0-tools/lib" ../configure \
    --prefix=/clang0-tools                          \
    --build=${C_TRIPLET}                            \
    --host=${C_TRIPLET}                             \
    --target=${H_TRIPLET}                           \
    --with-sysroot=/clang0-tools                    \
    --disable-nls                                   \
    --enable-languages=c,c++                        \
    --enable-clocale=generic                        \
    --enable-libstdcxx-time                         \
    --enable-threads=posix                          \
    --enable-fully-dynamic-string                   \
    --enable-shared                                 \
    --disable-multilib                              \
    --disable-libsanitizer                          \
    --disable-symvers                               \
    --disable-lto-plugin                            \
    --disable-libssp

# Build.
time { make AS_FOR_TARGET=${H_TRIPLET}-as LD_FOR_TARGET=${H_TRIPLET}-ld; }

# Install.
time { make install; }
```
```bash
# Adjust current GCC to produce binaries with "/clang0-tools/lib/ld-musl-x86_64.so.1".
export SPECFILE="$(dirname $(${H_TRIPLET}-gcc -print-libgcc-file-name))/specs"
${H_TRIPLET}-gcc -dumpspecs > specs
sed -i 's|/lib/ld-musl-x86_64.so.1|/clang0-tools/lib/ld-musl-x86_64.so.1|g' specs

# Check specs file.
grep --color=auto "/clang0-tools/lib/ld-musl-x86_64.so.1" specs

# Install specs file (if correct).
mv -fv specs "$SPECFILE"; unset SPECFILE

# Quick test.
echo "int main(){}" > dummy.c
${H_TRIPLET}-gcc dummy.c
readelf -l a.out | grep "Requesting"

# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang0-tools/lib/ld-musl-x86_64.so.1]
```

### `6` - NetBSD Curses
> #### `0.3.2` or newer
> The NetBSD Curses package contains libraries for terminal-independent handling of character screens.

> **Required!** To build Stage-0 Clang/LLVM.
```bash
# Build.
time { make CC=${H_TRIPLET}-gcc CFLAGS="-fPIC -Os -pipe" all-dynamic; }

# Install.
time { make PREFIX=/clang0-tools install-dynamic; }
```

### `7` - libexecinfo
> #### `1.1` or newer (from Heiwa/Linux fork)
> The libexecinfo package contains backtrace facility that usually found in GNU libc (glibc).

> **Required!** To build Stage-0 Clang/LLVM, since using musl libc.
```bash
# Build.
time { make CC=${H_TRIPLET}-gcc CFLAGS="-Os -pipe" dynamic; }

# Install.
time { make PREFIX=/clang0-tools install-{header,dynamic}; }
```

### `8` - Clang/LLVM
> #### `12.x.x` or newer
> C language family frontend for LLVM.

> **Required!** Build Stage-0 Clang/LLVM toolchain with `libgcc_s.so*` and `libstdc++.so*` dependencies since built with GCC toolchain.
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
# Configure source.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev                                                     \
    -DCMAKE_INSTALL_PREFIX="/clang0-tools"                                                  \
    -DCMAKE_C_COMPILER="${H_TRIPLET}-gcc"                                                   \
    -DCMAKE_CXX_COMPILER="${H_TRIPLET}-g++"                                                 \
    -DCMAKE_C_FLAGS="-g0 -Os -pipe"                                                         \
    -DCMAKE_CXX_FLAGS="-g0 -Os -pipe"                                                       \
    -DCMAKE_EXE_LINKER_FLAGS="-Wl,-dynamic-linker /clang0-tools/lib/ld-musl-x86_64.so.1"    \
    -DCMAKE_SHARED_LINKER_FLAGS="-Wl,-dynamic-linker /clang0-tools/lib/ld-musl-x86_64.so.1" \
    -DLLVM_HOST_TRIPLE="$T_TRIPLET"                                                         \
    -DLLVM_DEFAULT_TARGET_TRIPLE="$T_TRIPLET"                                               \
    -DLLVM_ENABLE_BINDINGS=OFF                                                              \
    -DLLVM_ENABLE_IDE=OFF                                                                   \
    -DLLVM_ENABLE_LIBCXX=ON                                                                 \
    -DLLVM_ENABLE_BACKTRACES=OFF                                                            \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF                                                         \
    -DLLVM_ENABLE_WARNINGS=OFF                                                              \
    -DLLVM_ENABLE_LIBEDIT=OFF                                                               \
    -DLLVM_ENABLE_LIBXML2=OFF                                                               \
    -DLLVM_ENABLE_OCAMLDOC=OFF                                                              \
    -DLLVM_ENABLE_ZLIB=OFF                                                                  \
    -DLLVM_ENABLE_Z3_SOLVER=OFF                                                             \
    -DLLVM_INCLUDE_BENCHMARKS=OFF                                                           \
    -DLLVM_INCLUDE_EXAMPLES=OFF                                                             \
    -DLLVM_INCLUDE_TESTS=OFF                                                                \
    -DLLVM_INCLUDE_GO_TESTS=OFF                                                             \
    -DLLVM_INCLUDE_DOCS=OFF                                                                 \
    -DLLVM_OPTIMIZED_TABLEGEN=ON                                                            \
    -DLLVM_TARGET_ARCH="$L_TARGET"                                                          \
    -DLLVM_TARGETS_TO_BUILD="$L_TARGET"                                                     \
    -DLIBUNWIND_ENABLE_STATIC=OFF                                                           \
    -DLIBCXXABI_ENABLE_STATIC=OFF                                                           \
    -DLIBCXX_ENABLE_STATIC=OFF                                                              \
    -DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF                                                \
    -DLIBCXX_HAS_MUSL_LIBC=ON                                                               \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                                                       \
    -DCOMPILER_RT_BUILD_MEMPROF=OFF                                                         \
    -DCOMPILER_RT_BUILD_PROFILE=OFF                                                         \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF                                                      \
    -DCOMPILER_RT_BUILD_XRAY=OFF                                                            \
    -DCLANG_VENDOR="Heiwa/Linux (ft. GNU)"                                                  \
    -DCLANG_ENABLE_ARCMT=OFF                                                                \
    -DCLANG_ENABLE_STATIC_ANALYZER=OFF                                                      \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++                                                       \
    -DCLANG_DEFAULT_RTLIB=compiler-rt                                                       \
    -DCLANG_DEFAULT_LINKER=lld                                                              \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind                                                     \
    -DDEFAULT_SYSROOT="/clang0-tools"

# Build.
time { make -C build; }

# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang0-tools" -P cmake_install.cmake && \
    popd
}
```
```bash
# Configure Stage-0 Clang with new triplet to produce binaries with "/clang1-tools/lib/ld-musl-x86_64.so.1" later.
ln -sv clang   /clang0-tools/bin/${H_TRIPLET}-clang
ln -sv clang++ /clang0-tools/bin/${H_TRIPLET}-clang++
cat > /clang0-tools/bin/${H_TRIPLET}.cfg << "EOF"
-Wl,-dynamic-linker /clang1-tools/lib/ld-musl-x86_64.so.1
EOF
```
```bash
# Back to "${HEIWA}/sources/pkgs" directory.
popd
```

### `9` - Cleaning Up
> #### This section is optional!

> If the intended user is not a programmer and does not plan to do any debugging on the system software, the system size can be decreased by removing the debugging symbols from binaries and libraries. This causes no inconvenience other than not being able to debug the software fully anymore.
```bash
# The libtool .la files are only useful when linking with static libraries.
# They are unneeded, and potentially harmful, when using dynamic shared libraries, specially when using non-autotools build systems.
# Remove those files.
find /clang0-tools/{{,${H_TRIPLET}/}lib{,64},libexec}/ -name '*.la' -exec rm -rfv {} \;

# Remove the documentation.
rm -rf /clang0-tools/share/{info,man,doc}/*

# Strip off debugging symbols from binaries using `llvm-strip`.
# A large number of files will be reported "The file was not recognized as a valid object file".
# These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
find /clang0-tools/{,${H_TRIPLET}/}lib{,64}/ -type f \( -name '*.a' -o -name '*.so*' \) -exec llvm-strip --strip-debug {} \;
find /clang0-tools/libexec/gcc/${H_TRIPLET}/10.3.1/ -type f -exec llvm-strip --strip-unneeded {} \;
find /clang0-tools/{,${H_TRIPLET}/}bin/ -maxdepth 1 -type f -exec /usr/bin/strip --strip-unneeded {} \;
```

<h2></h2>

Continue to [Stage-1 Clang/LLVM Toolchain](./3-Stage1_Clang_LLVM.md).
