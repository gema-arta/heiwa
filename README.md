# <p align="center">`Heiwa/Linux`</p>
<pre><p align="center"><samp>
A hobby and performance-oriented Linux® distribution
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
> |  ✓  | Low-level Standard Libraries and Toolchain         | Clang/LLVM         | Lightweight, Fast, Modern.     |
> |  ✓  | C Dynamic Memory Allocator                         | Microsoft mimalloc | Excellent performance.         |
> |  ✓  | Linux Kernel Patchset                              | Xanmod (CacULE)    | Optimized performance.         |
> |  ✓  | C Runtime Library                                  | musl               | Clean, but not fast as Glibc.  |
> |  ✓  | Build System Tools                                 | GNU                | Most packages depend.          |
> |  ✓  | Native Language Support                            | Gettext-tiny       | Stub of bloated GNU Gettext.   |
> |  ✓  | Secure Socket Layer Library                        | OpenSSL            | Full-featured and Robust.      |
> |  ✓  | Curses (terminal control) Library                  | NetBSD Curses      | Smaller than GNU Ncurses.      |
> |  ✓  | Command Line Interpreter or Shell                  | GNU Bash           | Best implementation.           |
> |  ✓  | Line-editing and History-capabilities Library      | GNU Readline       | Best implementation.           |
> |  ✓  | Deflate or Inflate Algorithm Compression Library   | Zlib-ng            | Next generation.               |
> |  ✓  | Unified Interface for Querying Installed Libraries | Pkgconf            | No circular dependencies.      |
> |  ✓  | Gzip Data Compressor and Decompressor              | Pigz               | Parallel threads support.      |
> |  ✓  | Most Userspace Utility Programs                    | Toybox             | No circular dependencies.      |
> |     | Init and Process Supervision                       | Finit              | F for fast. Fast init.         |
> |  ✓  | Default Text-editor                                | GNU Nano           | I don't use *Vim. :stuck_out_tongue_winking_eye: |

</details>

- [ ] Create a Proof-of-Concept (similiar to LFS books). *Under development, optimizing ..*

<details>
<summary>Details</summary>

<br>

> |  ?  | Stage                                                                                | Status            | Optimized more for         |
> |:---:|--------------------------------------------------------------------------------------|:-----------------:|----------------------------|
> |  ✓  | 1. [Preparation](./docs/poc/1-Preparation.md)                                        | Finished          | -                          |
> |  ✓  | 2. [Stage-0 Clang/LLVM (ft. GNU) Cross-Toolchain](./docs/poc/2-Stage0_Clang_LLVM.md) | Finished          | Build Size and Build Time. |
> |  ✓  | 3. [Stage-1 Clang/LLVM Toolchain](./docs/poc/3-Stage1_Clang_LLVM.md)                 | Finished          | Build Size and Build Time. |
> |     | 4. [Final System](./docs/poc/4-Final_System.md)                                      | Under development | Faster Performance.        |
> |     | 5. [System Configuration](./docs/poc/5-System_Configuration.md)                      | Pending           | -                          |

</details>

- [ ] Create a fast, flexible, and futuristic package manager.
- [ ] Build an auto build systems.
- [ ] Packaging, make it usable but with the aim of not being big.
> Always W.I.P, since this is hobbyist. A radical experiments.

See [docs](./docs). Go [discussions](https://github.com/heiwalinux/heiwa/discussions) for Errata.
