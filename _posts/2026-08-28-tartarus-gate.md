---
layout: post
title: "Syscall Series #2 - Tartarus' Gate"
date: 2026-08-28 09:00:00 +0200
tags: [Malware, Windows, Evasion]
author: Miguel Segura
categories: Malware Evasion
excerpt_separator: <!--more-->
---

Tartarus' Gate is a dynamic syscall discovery technique that locates syscall numbers in adjacent functions when the target function is hooked. Rather than searching within the function itself, it searches neighboring functions, exploiting the sequential organization of syscalls in ntdll.

<!--more-->

![Hell Gate](/assets/img/TartarusGate/tartarusGate.png)

## Introduction

- [Syscall Series #1 - Hell's Gate](https://gatesofkernel.com/posts/hells-gate/)
- Syscall Series #2 - Tartarus' Gate
- Syscall Series #3 - HellsHall (coming soon)
- Syscall Series #4 - VEH Syscalls (coming soon)

## Background
Tartarus' Gate fue presentada en Noviembre de 2021 por Trickster (@trickster0), como una evolución directa de Hell's Gate. Se puede encontrar su repositorio en [Tartarus' Gate](https://github.com/trickster0/TartarusGate).

The technique emerged in response to the increasing sophistication of EDRs, which began hooking entire functions instead of just the first few bytes. While Hell's Gate assumed clean bytecode existed within a hooked function, Tartarus' Gate shifted the paradigm: if there's no clean code in the target function, search the neighboring functions instead.

### Differences with Hell's Gate

Hell's Gate works as long as the hook on the target function is a simple one, typically a JMP, because you can step over it by searching inside the function byte by byte (cw++). But if the function contains a more sophisticated hook, as modern EDRs tend to deploy, the entire function can be overwritten and there is no useful bytecode left to walk. That is the problem Tartarus' Gate solves.

## How it works

### Block-based search

Instead of scanning byte by byte within a single function, Tartarus' Gate searches neighboring functions (adjacent in memory) until it finds an unhooked syscall stub. In ntdll, Nt* functions are physically sequential in memory, and their SSNs follow the same order.

![Flowchart](/assets/img/TartarusGate/tartarus_flowchart.svg)

Each function occupies roughly 32 to 64 bytes (two or more functions per cache line). Tartarus uses steps of 32 bytes (DOWN/UP) rather than single-byte increments. That step size corresponds to a typical NTDLL syscall stub including padding aligned to a cache line.

If it finds, for example, NtCreateFile at -32 bytes, its SSN is lower. At +32 bytes, it is higher. If a function is skipped in the search, the SSN changes by approximately 1 to 7.

![Memory Layout](/assets/img/TartarusGate/tartarus_memory_layout.svg)

### Hook detection

Hell's Gate detects only the first hooked byte (linear search). Tartarus detects two specific patterns:

- `0xe9` at offset 0: JMP at the function entry point
- `0xe9` at offset 3: JMP after the first few instructions

### SSN adjustment

Tartarus' Gate calculates the SSN using the formula:

```
SSN = SSN_found - idx   (searching DOWN)
SSN = SSN_found + idx   (searching UP)
```

This can fail if functions are not contiguous or if the gap between them is larger than expected. Offsets can also vary between Windows versions.

Tartarus validates that the found SSN is coherent: if it found NtCreateThreadEx at +32 bytes, it verifies that its SSN is sequentially higher than expected. If it does not match, the result is rejected.

## Limitations and what comes next

Tartarus' Gate assumes that if the target function is hooked, its neighbors are clean. But sophisticated EDRs do not hook functions in isolation, they hook entire regions by risk category. An EDR might hook all memory management functions (NtAllocateVirtualMemory, NtFreeVirtualMemory, NtProtectVirtualMemory) within a range of 100 to 200 bytes. In that scenario, Tartarus searches in 32-byte blocks but keeps finding hooked functions. The search fails because there is no clean neighbor to fall back to.

The assumption that adjacent syscalls have sequential SSNs (+1 to 7) is also not guaranteed across Windows versions. On Windows 7, NtAllocateVirtualMemory has SSN 0x18 and NtFreeVirtualMemory has SSN 0x1F (a gap of 7). On Windows 10 and 11, these may be reorganized or have larger gaps.

Some ntdll functions are too small or compiler-optimized. They can be inlined or jump to shared code (trampolines). If a function is a trampoline that jumps to ntdll+0x5000 where the real syscall lives, Tartarus may find the pattern `0x4c 0x8b 0xd1 0xb8` in inert code or in a different function, extracting the wrong SSN. The byte pattern is real but belongs to a different syscall.

Finally, the fixed 32-byte block search generates a more predictable memory access pattern compared to byte-by-byte scanning, which makes it easier to detect forensically.

## Evolution: Toward Indirect Syscalls

Tartarus' Gate improves significantly on Hell's Gate by handling hooked functions. However, it still assumes you can execute the syscall bytecode (`0x0f 0x05`) you find.

What if instead of executing the syscall you locate, you jumped to a syscall instruction that was already sitting in ntdll unhooked?

That is what Hell's Hall (2023) introduces: indirect syscalls that reuse existing syscall instructions inside ntdll, staying within the legitimate address space of the process. A technique that does not need to discover where to execute syscalls, only where to jump to run them.

## References
- [Tartarus' Gate](https://github.com/trickster0/TartarusGate) by Trickster (@trickster0)
- [MalDev Academy](https://maldevacademy.com/)