# <p align="center">`Heiwa/Linux`</p>
<p align="center">A hobby and performance-oriented Linux® distribution that provides aesthetics and practical functionality.</p>

<br>

## Roadmap <img alt="" align="right" src="https://badges.pufler.dev/visits/heiwalinux/heiwa?style=flat-square&label=&color=000000&logo=GitHub&logoColor=white&labelColor=373e4d"/>
- [x] Specifies the packages to be used, instead of mainstream GNU/Linux distribution.

<details>
<summary>Details</summary>

<br>

> |  ?  | Kernel and Userspace                               | Packages                  | Extended Description           |
> |:---:|----------------------------------------------------|:-------------------------:|--------------------------------|
> |  ✓  | Low-level Standard Libraries and Toolchain         | Clang/LLVM                | Pure, Fast, and Modern.        |
> |  ✓  | Linux Kernel Patchset                              | Xanmod                    | and .. CacULE CPU scheduler.   |
> |  ✓  | C Runtime Library                                  | musl                      | Clean, but not fast as Glibc.  |
> |  ✓  | Build System Tools                                 | GNU                       | Most packages depend.          |
> |  ✓  | Native Language Support                            | Gettext-tiny              | Stub of bloated GNU Gettext.   |
> |  ✓  | Secure Socket Layer Library                        | OpenSSL                   | Full-featured and Robust.      |
> |  ✓  | Curses (terminal control) Library                  | NetBSD Curses             | Smaller than GNU Ncurses.      |
> |  ✓  | Command Line Interpreter or Shell                  | GNU Bash                  | Best implementation.           |
> |  ✓  | Line-editing and History-capabilities Library      | GNU Readline              | Best implementation.           |
> |  ✓  | Deflate or Inflate Algorithm Compression Library   | Zlib-ng                   | Next generation.               |
> |  ✓  | Unified Interface for Querying Installed Libraries | Pkgconf                   | No circular dependencies.      |
> |  ✓  | Gzip Data Compressor and Decompressor              | Pigz                      | Parallel threads support.      |
> |  ✓  | Most Userspace Utility Programs                    | Toybox                    | No circular dependencies.      |
> |  ✓  | Init and Service Manager                           | OpenRC                    | Sophisticated version of SysV. |
> |  ✓  | Default Text-editor                                | GNU Nano                  | I don't use *Vim. :stuck_out_tongue_winking_eye: |

> Maybe init switches to [finit](https://github.com/troglobit/finit) if that fits.
> 
> [See the reference about alternate packages.](https://wiki.musl-libc.org/alternatives.html)

</details>

- [ ] Create a Proof-of-Concept (similiar to LFS books). *Under development, optimizing ..*
- [ ] Create a fast, flexible, and futuristic package manager.
- [ ] Build an auto build systems.
- [ ] Packaging, make it usable but with the aim of not being big.
> Always W.I.P, since this is hobbyist. A radical experiments.

See [docs](./docs).
