# <p align="center">`Heiwa/Linux`</p>
<pre><p align="center"><samp>
A hobby and performance-oriented Linux® system
that provides aesthetics and practical functionality.
Without sacrificing <b>build size</b> and <b>build time</b> just
for optimizing <b>the performance</b>. It's all <b>balanced</b>.
</samp></p></pre>

<br>

## Roadmap <img alt="" align="right" src="https://badges.pufler.dev/visits/heiwalinux/heiwa?style=flat-square&label=&color=000000&logo=GitHub&logoColor=white&labelColor=373e4d"/>
- [x] Specifies the packages to be used, instead of mainstream GNU/Linux distribution.

<details>
<summary>Details</summary>

<br>

> |  ?  | Kernel and Userspace                               | Packages           | Extended Description           |
> |:---:|----------------------------------------------------|:------------------:|--------------------------------|
> |  ✓  | Low-level standard libraries and toolchain         | Clang/LLVM         | Modular, fast, and modern.     |
> |  ?  | C dynamic memory allocator                         | Microsoft mimalloc | Excellent performance.         |
> |  ✓  | Linux kernel patchset                              | Xanmod (CacULE)    | Optimized performance.         |
> |  ✓  | C runtime library                                  | musl               | Clean, POSIX, and [fast?](https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes) |
> |  ✓  | Build system tools                                 | GNU                | Broad-scale compatibility.     |
> |  ✓  | SSL/TLS implementation                             | OpenSSL            | Full-featured and robust.      |
> |  ✓  | Native language support                            | Gettext-tiny       | Stub of bloated GNU Gettext.   |
> |  ✓  | Curses (terminal control) library                  | NetBSD Curses      | Smaller than GNU Ncurses.      |
> |  ✓  | Command-line interpreter or shell                  | GNU Bash           | Best implementation.           |
> |  ✓  | Line-editing and history-capabilities library      | GNU Readline       | Best implementation.           |
> |  ✓  | Deflate or inflate algorithm compression library   | Zlib-ng            | Optimized for NGS.             |
> |  ✓  | Unified interface for querying installed libraries | Pkgconf            | No circular dependencies.      |
> |  ✓  | Gzip data compressor and decompressor              | Pigz               | Parallel threads support.      |
> |  ✓  | Most userspace utility programs                    | Toybox             | Small, fast, and simple.       |
> |     | Init and process supervision                       | Finit              | F for fast. Fast init.         |
> |  ✓  | Default text-editor                                | GNU Nano           | I don't use Neo/Vi/m. :stuck_out_tongue_winking_eye: |
> |  ✓  | Device manager                                     | Eudev              | No reason, portable.           |

> I think Microsoft mimalloc breaks some packages if build whole system with it, need more research.

</details>

- [ ] Create a Proof-of-Concept. *Under development, optimizing ..*

<details>
<summary>Details</summary>

<br>

> |  ?  | Stage                                                                                | Status            | Optimized more for         |
> |:---:|--------------------------------------------------------------------------------------|:-----------------:|----------------------------|
> |  ✓  | 1. [Preparation](./docs/poc/1-Preparation.md)                                        | Finished          | -                          |
> |  ✓  | 2. [Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain](./docs/poc/2-Stage0_Clang_LLVM.md) | Finished          | Build size and build time. |
> |  ✓  | 3. [Stage-1 Clang/LLVM Toolchain](./docs/poc/3-Stage1_Clang_LLVM.md)                 | Finished          | Build size and build time. |
> |     | 4. [Final System](./docs/poc/4-Final_System.md)                                      | Under development | Faster performance.        |
> |     | 5. [System Configuration](./docs/poc/5-System_Configuration.md)                      | Pending           | -                          |

</details>

- [ ] Create a fast, flexible, and futuristic package manager.
- [ ] Build an auto build systems.
- [ ] Packaging, make it usable but with the aim of not being big.

> Always W.I.P, since this is hobbyist. A radical experiments.

See [docs](./docs). Go [discussions](https://github.com/heiwalinux/heiwa/discussions) for the Errata.

##  
<pre><samp>
owl4ce © 2021

The registered trademark Linux® is used pursuant to a sublicense from the
Linux Foundation, the exclusive licensee of Linus Torvalds, owner of the mark
on a world­wide basis.

</samp></pre>
The Proof-of-Concept is licensed under the [MIT License](./docs/poc/LICENSE). The entire repository is licensed under the [GPL-2.0 License](./LICENSE).
