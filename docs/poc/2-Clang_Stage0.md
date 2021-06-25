## `II` Clang Stage 0
> **Compilation Instruction!**
> ```sh
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
> 3. Using GNU's `bison` as `yacc` replacements (symlinked).
> 4. Using `flex` as `lex` alternative lexical analyzers (symlinked).
> 
> ```sh
> file $(command -v {sh,awk,yacc,lex})
> ```

### `1` - Linux API Headers
> #### Xanmod-CacULE, `5.10.x` or newer
> Required for runtime C library (musl) to use Linux API.
```sh
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# The recommended make target "headers_install" cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in ./usr, then copied to the needed location.
time {
    [[ -n $HEIWA_ARCH ]] && \
    make ARCH=${HEIWA_ARCH} headers_check && \
    make ARCH=${HEIWA_ARCH} headers
}

# Remove unnecessary files.
find usr/include -name '.*' -exec rm -rfv {} \;
rm -fv usr/include/Makefile

# Install.
[[ -n $HEIWA_TARGET ]] && \
mkdir -pv /clang0-tools/${HEIWA_TARGET} && \
cp -rv usr/include /clang0-tools/${HEIWA_TARGET}/.
```

### `2` - Binutils
> #### `2.36.1` or newer
> Required to build GCC.
```sh
# Create a dedicated directory and configure source.
[[ -n $HEIWA_TARGET ]] && mkdir -v build && cd build && \
../configure \
    --prefix=/clang0-tools                       \
    --target=${HEIWA_TARGET}                     \
    --with-sysroot=/clang0-tools/${HEIWA_TARGET} \
    --disable-nls                                \
    --disable-multilib                           \
    --disable-werror                             \
    --enable-deterministic-archives              \
    --disable-compressed-debug-sections

# Checks the host's environment and makes sure all the necessary tools are available to compile Binutils.
# Then build.
time { make configure-host && make; }

# Install.
time { make install; }
```

### `3` -  GCC (static)
> #### `10.3.1_git20210424` (from Alpine Linux)
> Required to compile required libraries to bootstrap clang.
```sh
# GCC requires the GMP, MPFR, and MPC packages to either be present on the host or to be present in source form within the gcc source tree.
tar xf ../gmp-6.2.1.tar.xz  && mv -v gmp-6.2.1 gmp
tar xf ../mpfr-4.1.0.tar.xz && mv -v mpfr-4.1.0 mpfr
tar xzf ../mpc-1.2.1.tar.gz && mv -v mpc-1.2.1 mpc

# Create a dedicated directory and configure source.
[[ -n "$HEIWA_HOST" && "$HEIWA_TARGET" && "$HEIWA_CPU" ]] && \
mkdir -v build && cd build && \
CFLAGS="-g0 -O0" CXXFLAGS="-g0 -O0" && ../configure  \
    --prefix=/clang0-tools     --build=${HEIWA_HOST} \
    --host=${HEIWA_HOST}    --target=${HEIWA_TARGET} \
    --with-sysroot=/clang0-tools/${HEIWA_TARGET}     \
    --disable-nls                      --with-newlib \
    --disable-libitm                --disable-libvtv \
    --disable-libssp                --disable-shared \
    --disable-libgomp              --without-headers \
    --disable-threads             --disable-multilib \
    --disable-libatomic          --disable-libstdcxx \
    --enable-languages=c       --disable-libquadmath \
    --disable-libsanitizer  --with-arch=${HEIWA_CPU} \
    --disable-decimal-float --enable-clocale=generic

# Build only the minimum.
time { make all-gcc all-target-libgcc; }

# Install.
time { make install-gcc install-target-libgcc; }
```
