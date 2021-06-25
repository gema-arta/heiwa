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
> **Verify:**
> ```sh
> file $(command -v {sh,awk,yacc,lex})
> ```

### `1` - Linux API Headers
> Recommended to use LTS.
```sh
# Make sure there are no stale files embedded in the package.
time { make mrproper; }

# Now extract the user-visible kernel headers from the source.
# The recommended make target "headers_install" cannot be used, because it requires rsync, which may not be available.
# The headers are first placed in ./usr, then copied to the needed location.
time {
    [ ! -z $HEIWA_ARCH ] && \
    make ARCH=${HEIWA_ARCH} headers_check && \
    make ARCH=${HEIWA_ARCH} headers
}

find usr/include -name '.*' -exec rm -rfv {} \;
rm -v usr/include/Makefile

[ ! -z $HEIWA_TARGET ] && \
mkdir -pv /clang0-tools/${HEIWA_TARGET} && \
cp -rv usr/include /clang0-tools/${HEIWA_TARGET}/.
```
