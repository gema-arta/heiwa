## `II` Stage-0 Clang/LLVM (feat. GNU) Cross-Toolchain
The purpose of this stage is to build stage 1 Clang/LLVM toolchain with GCC libraries which will be used to build stage 2 Clang/LLVM toolchain (self-hosted), because if it build directly using host's Clang/LLVM nor GCC, it will fails or breaks. [LLVM reference](https://clang.llvm.org/docs/Toolchain.html) **•** [CLFS reference](http://clfs.org/view/clfs-embedded/x86) **•** [musl faq](https://wiki.musl-libc.org/faq.html).

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
> heiwa@...pkgs/target-package $ make PREFIX="$(pwd)/work/clang1-tools" install
> heiwa@...pkgs/target-package/build $ cmake -DCMAKE_INSTALL_PREFIX="$(pwd)/work/clang1-tools" -P cmake_install.cmake 
>
> < exiting and cleaning up directory >
> ```

### `1` - Linux API Headers
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
time { make mrproper && make ARCH=${TGT_ARCH} headers; }
```
```bash
# Remove unnecessary dotfiles and Makefile.
find usr/include \( -name '.*' -o -name 'Makefile' \) -exec rm -fv {} \;
```
```bash
# Install.
mkdir -v /clang1-tools/${HEI_TRIPLET} && \
cp -rfv usr/include /clang1-tools/${HEI_TRIPLET}/.
```

### `2` - GNU Binutils
> #### `2.37` or newer
> The GNU Binutils package contains a linker, an assembler, and other tools for handling object files.

> **Required!** The musl libc and GCC perform various tests on the linker and assembler to determine which of their own features to enable.
> > **Build time:** ~3m
```bash
# Create a dedicated directory and configure source.
mkdir -v build && cd build &&                            \
../configure --prefix=/clang1-tools                      \
             --target=${HEI_TRIPLET}                     \
             --with-sysroot=/clang1-tools/${HEI_TRIPLET} \
             --without-debuginfod                        \
             --without-stage1-ldflags                    \
             --enable-deterministic-archives             \
             --disable-compressed-debug-sections         \
             --disable-gdb                               \
             --disable-multilib                          \
             --disable-nls                               \
             --disable-libdecnumber                      \
             --disable-lto{,-plugin}                     \
             --disable-readline                          \
             --disable-separate-code                     \
             --disable-sim                               \
             --disable-static                            \
             --disable-werror
```
```bash
# Check host's environment and ensure all necessary tools are available to build Binutils. Then build.
time { make configure-host && make; }
```
```bash
# Install.
time { make install; }
```

### `3` - GCC <kbd>static</kbd>
> #### `10.3.1_git20210921` (from Alpine Linux)
> The GCC package contains the GNU compiler collection, which includes the C and C++ compilers.

> **Required!** This build of GCC is mainly done, so that the musl libc can be built next.
> > **Build time:** <20m
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf  ../gmp-6.2.1.tar.xz && mv -fv gmp{-6.2.1,}
tar xf ../mpfr-4.1.0.tar.xz && mv -fv mpfr{-4.1.0,}
tar xzf ../mpc-1.2.1.tar.gz && mv -fv mpc{-1.2.1,}
```
```bash
# Create a dedicated directory and configure source. Disable optimization.
mkdir -v build && cd build &&                            \
CFLAGS="$(tr Os O0 <<< "$CFLAGS")" CXXFLAGS="$CFLAGS"    \
../configure --prefix=/clang1-tools                      \
             --build=${HST_TRIPLET}                      \
             --host=${HST_TRIPLET}                       \
             --target=${HEI_TRIPLET}                     \
             --with-sysroot=/clang1-tools/${HEI_TRIPLET} \
             --with-arch=${GCC_MCPU}                     \
             --with-newlib                               \
             --without-headers                           \
             --enable-clocale=generic                    \
             --enable-initfini-array                     \
             --enable-languages=c                        \
             --disable-decimal-float                     \
             --disable-gnu-indirect-function             \
             --disable-gnu-unique-object                 \
             --disable-lto{,-plugin}                     \
             --disable-multilib                          \
             --disable-nls                               \
             --disable-shared                            \
             --disable-symvers                           \
             --disable-threads                           \
             --disable-werror                            \
             --disable-libatomic                         \
             --disable-libgomp                           \
             --disable-libitm                            \
             --disable-libmpx                            \
             --disable-libmudflap                        \
             --disable-libquadmath                       \
             --disable-libsanitizer                      \
             --disable-libssp                            \
             --disable-libstdcxx                         \
             --disable-libvtv
```
```bash
# Only build the minimum.
time { make all-{gcc,target-libgcc}; }
```
```bash
# Install.
time { make install-{gcc,target-libgcc}; }
```

### `4` - musl
> #### `1.2.2` or newer
> The musl package contains the main C library. This library provides the basic routines for allocating memory, searching directories, opening and closing files, reading and writing files, string handling, pattern matching, arithmetic, and so on.

> **Required!** As mentioned in the description above.
> > **Build time:** ~2m
```bash
# Configure source. Use optimization level 3.
CFLAGS="$(tr Os O3 <<< "$CFLAGS")"        \
./configure CROSS_COMPILE=${HEI_TRIPLET}- \
            --prefix=/                    \
            --target=${HEI_TRIPLET}       \
            --with-malloc=mallocng        \
            --disable-gcc-wrapper         \
            --disable-optimize            \
            --disable-static
```
```bash
# Build.
time { make; }
```
```bash
# Install and fix wrong shared object symlink.
time {
    make DESTDIR=/clang1-tools install
    ln -sfv libc.so /clang1-tools/lib/ld-musl-${TGT_ARCH}.so.1
}
```
```bash
# Optionally, create a `ldd` symlink to use to print shared object dependencies.
ln -sfv ../lib/libc.so /clang1-tools/bin/ldd
```
```bash
# Configure path for the dynamic linker.
mkdir -v /clang1-tools/etc && \
cat > /clang1-tools/etc/ld-musl-${TGT_ARCH}.path << EOF
/clang1-tools/lib
/clang1-tools/${HEI_TRIPLET}/lib
/lib64
/usr/lib64
/lib
/usr/lib
EOF
```
```bash
# The final GCC will looking for headers in the "/clang1-tools/usr/include", so create the directory then symlink it.
mkdir -v /clang1-tools/usr && \
ln -sv ../include /clang1-tools/usr/include
```

### `5` - GCC <kbd>final</kbd>
> #### `10.3.1_git20210921` (from Alpine Linux)
> The GCC package contains the GNU compiler collection, which includes the C and C++ compilers.

> **Required!** This second build of GCC will produce the final cross compiler which will use the previously built musl libc.
> > **Build time:** <40m
```bash
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the GCC source tree.
tar xf  ../gmp-6.2.1.tar.xz && mv -fv gmp{-6.2.1,}
tar xf ../mpfr-4.1.0.tar.xz && mv -fv mpfr{-4.1.0,}
tar xzf ../mpc-1.2.1.tar.gz && mv -fv mpc{-1.2.1,}
```
```bash
# Apply patches (from Alpine Linux).
../../syscore/gcc/patches/appatch
```
```bash
# Create a dedicated directory and configure source.
mkdir -v build && cd build &&                \
../configure --prefix=/clang1-tools          \
             --build=${HST_TRIPLET}          \
             --host=${HST_TRIPLET}           \
             --target=${HEI_TRIPLET}         \
             --with-sysroot=/clang1-tools    \
             --enable-clocale=generic        \
             --enable-initfini-array         \
             --enable-languages=c,c++        \
             --enable-shared                 \
             --enable-threads=posix          \
             --disable-gnu-indirect-function \
             --disable-gnu-unique-object     \
             --disable-lto{,-plugin}         \
             --disable-multilib              \
             --disable-nls                   \
             --disable-static                \
             --disable-symvers               \
             --disable-werror                \
             --disable-libmpx                \
             --disable-libmudflap            \
             --disable-libsanitizer          \
             --disable-libssp                \
             --disable-libvtv
```
```bash
# Build.
time { make; }
```
```bash
# Install.
time { make install; }
```
```bash
# Adjust current GCC to produce binaries with "/clang1-tools/lib/ld-musl-${TGT_ARCH}.so.1" by modifying the specs.
if ${HEI_TRIPLET}-gcc -dumpspecs > specs; then
    sed -i "s|/lib/ld-musl-${TGT_ARCH}.so.1|/clang1-tools/lib/ld-musl-${TGT_ARCH}.so.1|g" specs
fi
```
```bash
# Install the specs file if the path is correct.
if grep --color=auto "/clang1-tools/lib/ld-musl-${TGT_ARCH}.so.1" specs; then
    SPECFILE="$(dirname $(${HEI_TRIPLET}-gcc -print-libgcc-file-name))/specs"
    mv -fv specs "$SPECFILE" && unset SPECFILE
fi
```
```bash
# Sanity check for the current GCC.
${HEI_TRIPLET}-gcc -x c++ - ${CFLAGS} ${LDFLAGS} <<< 'int main(){}'
${HEI_TRIPLET}-readelf -l a.out | grep --color=auto "Req.*ter"
```
```bash
# | The output should be:
# |-----------------------
# |      [Requesting program interpreter: /clang1-tools/lib/ld-musl-x86_64.so.1]
```

### `6` - Zstd
> #### `1.5.0` or newer
> The Zstd (Zstandard) package contains real-time compression algorithm, providing high compression ratios. It offers a very wide range of compression / speed trade-offs, while being backed by a very fast decoder.

> **Required!** Only build dynamic libraries and headers for Ccache compression support.
> > **Build time:** <2m
```bash
# Build with verbose. Use optimization level 3.
time { make -C lib CC=${HEI_TRIPLET}-gcc libzstd-release V=1; }
```
```bash
# Install.
time { make -C lib PREFIX=/clang1-tools install-{includes,shared}; }
```

### `7` - Ccache
> #### `4.4.1` or newer
> The Ccache package contains compiler cache. It speeds up recompilation by caching previous compilations and detecting when the same compilation is being done again.

> **Required!** To speeds up Clang/LLVM builds across stage 1 and stage 2, especially when errors occur such as power loss.
> > **Build time:** ~3m
```bash
# Configure source. Use optimization level 3.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev       \
    -DCMAKE_PREFIX_PATH="/clang1-tools"       \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"    \
    -DCMAKE_C_COMPILER="${HEI_TRIPLET}-gcc"   \
    -DCMAKE_CXX_COMPILER="${HEI_TRIPLET}-g++" \
    -DENABLE_DOCUMENTATION=NO                 \
    -DENABLE_TESTING=NO                       \
    -DREDIS_STORAGE_BACKEND=NO
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install.
time { make -C build install; }
```
```bash
# Configure ccache.
cat > /clang1-tools/etc/ccache.conf << "EOF"
umask = 002
compiler_check = none
compression = true
compression_level = 1
EOF
```

### `8` - Clang/LLVM <kbd>stage 1</kbd>
> #### `12.x.x` or newer
> - C language family frontend for LLVM;  
> - C++ runtime stack unwinder from LLVM;  
> - Low level support for a standard C++ library from LLVM;  
> - New implementation of the C++ standard library, targeting C++11 from LLVM.

> **Required!** Build Stage-0 Clang/LLVM toolchain with `libgcc_s.so*` and `libstdc++.so*` dependencies since built with GCC toolchain.
> > **Build time:** ~4h-6h
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
../syscore/llvm/patches/appatch
```
```bash
# Configure the entire source. Use optimization level 3.
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release -Wno-dev         \
    -DCMAKE_INSTALL_PREFIX="/clang1-tools"      \
    -DCMAKE_C_COMPILER="${HEI_TRIPLET}-gcc"     \
    -DCMAKE_CXX_COMPILER="${HEI_TRIPLET}-g++"   \
    -DBUILD_SHARED_LIBS=ON                      \
    -DLLVM_CCACHE_BUILD=ON                      \
    -DLLVM_APPEND_VC_REV=OFF                    \
    -DLLVM_HOST_TRIPLE="$HEI_TRIPLET"           \
    -DLLVM_DEFAULT_TARGET_TRIPLE="$TGT_TRIPLET" \
    -DLLVM_ENABLE_BINDINGS=OFF                  \
    -DLLVM_ENABLE_IDE=OFF                       \
    -DLLVM_ENABLE_LIBCXX=ON                     \
    -DLLVM_ENABLE_BACKTRACES=OFF                \
    -DLLVM_ENABLE_UNWIND_TABLES=OFF             \
    -DLLVM_ENABLE_WARNINGS=OFF                  \
    -DLLVM_ENABLE_LIBEDIT=OFF                   \
    -DLLVM_ENABLE_TERMINFO=OFF                  \
    -DLLVM_ENABLE_LIBXML2=OFF                   \
    -DLLVM_ENABLE_OCAMLDOC=OFF                  \
    -DLLVM_ENABLE_ZLIB=OFF                      \
    -DLLVM_ENABLE_Z3_SOLVER=OFF                 \
    -DLLVM_INCLUDE_BENCHMARKS=OFF               \
    -DLLVM_INCLUDE_EXAMPLES=OFF                 \
    -DLLVM_INCLUDE_TESTS=OFF                    \
    -DLLVM_INCLUDE_GO_TESTS=OFF                 \
    -DLLVM_INCLUDE_DOCS=OFF                     \
    -DLLVM_TARGET_ARCH="$TGT_LLVM"              \
    -DLLVM_TARGETS_TO_BUILD="$TGT_LLVM"         \
    -DLIBUNWIND_ENABLE_ASSERTIONS=OFF           \
    -DLIBUNWIND_ENABLE_STATIC=OFF               \
    -DLIBCXXABI_ENABLE_ASSERTIONS=OFF           \
    -DLIBCXXABI_ENABLE_STATIC=OFF               \
    -DLIBCXX_ENABLE_STATIC=OFF                  \
    -DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF    \
    -DLIBCXX_HAS_MUSL_LIBC=ON                   \
    -DLIBCXX_INCLUDE_BENCHMARKS=OFF             \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF           \
    -DCOMPILER_RT_BUILD_MEMPROF=OFF             \
    -DCOMPILER_RT_BUILD_PROFILE=OFF             \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF          \
    -DCOMPILER_RT_BUILD_XRAY=OFF                \
    -DCLANG_VENDOR="Heiwa/Linux (feat. GNU)"    \
    -DCLANG_ENABLE_ARCMT=OFF                    \
    -DCLANG_ENABLE_STATIC_ANALYZER=OFF          \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++           \
    -DCLANG_DEFAULT_RTLIB=compiler-rt           \
    -DCLANG_DEFAULT_LINKER=lld                  \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind         \
    -DDEFAULT_SYSROOT="/clang1-tools"
```
```bash
# Build.
time { make -C build; }
```
```bash
# Install.
time {
    pushd build/ && \
        cmake -DCMAKE_INSTALL_PREFIX="/clang1-tools" -P cmake_install.cmake && \
    popd
}
```
```bash
# Configure Stage-0 Clang/LLVM with heiwa triplet to produce binaries with "/clang2-tools/lib/ld-musl-${TGT_ARCH}.so.1" later.
ln -sfv clang   /clang1-tools/bin/${HEI_TRIPLET}-clang
ln -sfv clang++ /clang1-tools/bin/${HEI_TRIPLET}-clang++
cat > /clang1-tools/bin/${HEI_TRIPLET}.cfg << EOF
-Wl,-dynamic-linker /clang2-tools/lib/ld-musl-${TGT_ARCH}.so.1
EOF
```
```bash
# Back to "${HEIWA}/sources/pkgs" directory.
popd
```

### `9` - Cleaning Up and Stripping Unneeded Symbols
> #### This section is recommended!

> Remove the documentation and manpages.
```bash
rm -rf /clang1-tools/share/{info,man}
```
> The libtool ".la" files are only useful when linking with static libraries. They are unneeded and potentially harmful when using dynamic shared libraries, specially when using non-autotools build systems. So, remove those files.
```bash
find /clang1-tools/{libexec,{,${HEI_TRIPLET}/}lib}/ -name '*.la' -exec rm -fv {} \;
```
> Strip off all unneeded symbols from binaries using `llvm-strip`. A large number of files will be reported "The file was not recognized as a valid object file". These warnings can be safely ignored. These warnings indicate that those files are scripts instead of binaries.
```bash
find /clang1-tools/{,${HEI_TRIPLET}/}lib/ -type f \( -name '*.a' -o -name '*.so*' \) -exec llvm-strip --strip-unneeded {} \;
```
```bash
if cp -v $(command -v llvm-strip) ./; then
    find /clang1-tools/{libexec,{,${HEI_TRIPLET}/}bin}/ -type f -exec ./llvm-strip --strip-unneeded {} \;
fi && rm -v ./llvm-strip
```

<h2></h2>

Continue to [Stage-1 Clang/LLVM Toolchain](./3-Stage1_Clang_LLVM.md).
