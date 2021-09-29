# <p align="center">`Heiwa/Linux`</p>
<pre><p align="center"><samp>
A hobby and performance-oriented Linux® system
that provides aesthetics and practical functionality.
Without sacrificing <b>build size</b> and <b>build time</b> just
for optimizing <b>the performance</b>. It's all <b>balanced</b>.
</samp></p></pre>

> **Warning!**  
> Everything in this repo is still unstable and still improvising, don't practice it by yourself. The PoC is messed up.

## Roadmap <img alt="" align="right" src="https://badges.pufler.dev/visits/heiwalinux/heiwa?style=flat-square&label=&color=000000&logo=GitHub&logoColor=white&labelColor=373e4d"/>
- [x] Specifies the packages to be used, instead of mainstream GNU/Linux distribution.

<details>
<summary>Details</summary>

<br>

> |  ?  | Kernel and Userspace                               | Packages           | Extended Description            |
> |:---:|----------------------------------------------------|:------------------:|---------------------------------|
> |  ✓  | Low-level standard libraries and toolchain         | Clang/LLVM         | Modular, fast, and modern.      |
> |  ?  | C dynamic memory allocator                         | Microsoft mimalloc | Excellent performance.          |
> |  ✓  | Linux kernel patchset                              | Xanmod (CacULE)    | Optimized performance.          |
> |  ✓  | C runtime library                                  | musl               | Clean, POSIX, and [fast?](https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes) |
> |  ✓  | Build system tools                                 | GNU                | Broad-scale compatibility.      |
> |  ✓  | SSL/TLS implementation                             | OpenSSL            | Full-featured and robust.       |
> |  ✓  | Native language support                            | gettext-tiny       | Stub of bloated GNU gettext.    |
> |  ✓  | Z data compression library                         | Zlib-ng            | Optimized for NGS.              |
> |  ✓  | Curses (terminal control) library                  | NetBSD curses      | Smaller than GNU ncurses.       |
> |  ✓  | Command-line interpreter or shell                  | GNU Bash           | Best implementation.            |
> |  ✓  | Line-editing and history-capabilities library      | GNU Readline       | Best implementation.            |
> |  ✓  | Unified interface for querying installed libraries | Pkgconf            | No circular dependencies.       |
> |  ✓  | Gzip data compressor and decompressor              | Pigz               | Parallel threads support.       |
> |  ✓  | Most userspace utility programs                    | Toybox             | Small, fast, and simple.        |
> |  ✓  | Ninja build system replacement                     | Samurai            | Portable (C99), simple, faster. |
> |     | Init and process supervision                       | Finit              | F for fast. Fast init.          |
> |  ✓  | Manpage suite tools                                | OpenBSD mandoc     | Smaller than GNU man-db.        |
> |  ✓  | Default text-editor                                | GNU nano           | I don't use Neo/Vi/m. :stuck_out_tongue_winking_eye: |
> |  ✓  | Device manager                                     | eudev              | No reason, portable.            |

> **Note:**
> 1. Microsoft mimalloc breaks some packages if build the whole system with it, need more research;
> 2. Samurai is only used in the "Final System".

</details>

- [ ] Create a Proof-of-Concept. Under developments, optimizing ..

<details>
<summary>Details</summary>

<br>

> |  ?  | Build Stages                                                                           | Status             | Optimized more for         |
> |:---:|----------------------------------------------------------------------------------------|:------------------:|----------------------------|
> |  ✓  | 1. [Preparation](./docs/poc/1-Preparation.md)                                          | Finished (unfixed) | -                          |
> |  ✓  | 2. [Stage-0 Clang/LLVM (feat. GNU) Cross-Toolchain](./docs/poc/2-Stage0_Clang_LLVM.md) | Finished (unfixed) | Build time and build size. |
> |  ✓  | 3. [Stage-1 Clang/LLVM Toolchain](./docs/poc/3-Stage1_Clang_LLVM.md)                   | Finished (unfixed) | Build time and build size. |
> |  ?  | 4. [Stage-2 Final System](./docs/poc/4-Stage2_Final_System.md)                         | Under devel        | Faster performance.        |
> |     | 5. [System Configuration](./docs/poc/5-System_Configuration.md)                        | Pending            | -                          |

> Generally there's no **Stage-0** for the toolchain. I lowered the **build stages** value because to match the final toolchain which actually in the **Stage-2** "Final System", there are 3 stages where "stage 1 Clang/LLVM" in the **Stage-0** uses GCC libraries after bootstrapping musl libc and "stage 2 Clang/LLVM" in the **Stage-1** is no more from minimal as **Stage-0** but become native self-hosted. Then, **Stage-1** is used to build **Stage-2** "Final System".
> 
> > Well, the **build stages** and **toolchain stages** are differents.

> The current "inefficient", "painful", and "fake cross-compilation" method as described below:
> | Build Stages | Build | Host  | Target | Action                                                                                            | Status         |
> |:------------:|:-----:|:-----:|:------:|---------------------------------------------------------------------------------------------------|:--------------:|
> | Stage-0      | host  | host  | heiwa  | Build GCC using host's GCC, then build stage 1 Clang/LLVM using second GCC (musl) built.          | Cross          |
> | Stage-1      | host  | heiwa | heiwa  | Build stage 2 Clang/LLVM using previously Clang/LLVM built. Now become self-hosted.               | Crossed Native |
> | Stage-2      | heiwa | heiwa | heiwa  | Build "Final System" using second Clang/LLVM built. This LLVM built has wider registered targets. | Native         |
> 
> > If **build**, **host**, and **target** are all the same, this is called a **native**.
> 
> > If **build** and **host** are the same but **target** is different, this is called a **cross**.
> 
> > If **build**, **host**, and **target** are all different this is called a **canadian** (for obscure reasons dealing with Canada's political party and the background of the person working on the build at that time).
> 
> > If **host** and **target** are the same, but **build** is different, you are using a **cross-compiler** to build a **native** for a **different system**. Some people call this a **host-x-host**, **crossed native**, or **cross-built native**.
> 
> > If **build** and **target** are the same, but **host** is different, you are using a **cross-compiler** to build a **cross-compiler** that produces code for **the machine you're building on**. This is rare, so there is no common way of describing it. There is a proposal to call this a **crossback**.

> This will be long to develop PoC along with the package manager, and the **Stage-2** is like [Gentoo Stage 3 tarball](https://wiki.gentoo.org/wiki/Stage_tarball).

</details>

- [ ] Create a fast, flexible, and futuristic package manager.
- [ ] Create an auto-build system. Work in progress ..
- [ ] Packaging, make it usable but with the aim of not being big. When? Idk.

> Always W.I.P, since this is hobbyist. A radical experiments.

See [docs](./docs). Go [discussions](https://github.com/heiwalinux/heiwa/discussions) for the Errata.

##  
<pre><samp>
<a href="https://github.com/owl4ce">owl4ce</a> © 2021

The registered trademark Linux® is used pursuant to a sublicense from the Linux Foundation,
the exclusive licensee of Linus Torvalds, owner of the mark on a world-wide basis.

</samp></pre>
The Proof-of-Concept is licensed under the [MIT License](./docs/poc/LICENSE). The entire repository is licensed under the [GPL-2.0 License](./LICENSE).
