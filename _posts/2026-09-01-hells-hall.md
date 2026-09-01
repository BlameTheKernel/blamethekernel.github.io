---
layout: post
title: "Syscall Series #3 - Hell's Hall"
date: 2026-09-01 09:00:00 +0200
tags: [Malware, Windows, Evasion]
author: Miguel Segura
categories: Malware Evasion
excerpt_separator: <!--more-->
---

Hell's Hall is an indirect syscall technique that executes syscall instructions from system DLLs other than ntdll.dll, typically kernel32.dll. By making the return address appear to originate from a trusted DLL, it evades kernel instrumentation callbacks that would flag syscalls executing from ntdll.dll directly.

<!--more-->

![Hells Hall](/assets/img/HellsHall/hells_hall.png)

## Introduction

- [Syscall Series #1 - Hell's Gate](https://gatesofkernel.com/posts/hells-gate/)
- [Syscall Series #2 - Tartarus' Gate](https://gatesofkernel.com/posts/tartarus-gate/)
- Syscall Series #3 - Hell's Hall
- Syscall Series #4 - VEH Syscalls (coming soon)

The main problem with Hell's Gate and Tartarus' Gate is that direct syscalls are detectable by EDRs through a Windows mechanism called Nirvana, introduced in Windows Vista (2006).

Nirvana is an internal Windows mechanism that monitors user-mode execution through kernel callbacks, allowing detection of manual syscalls and other evasive behaviors. It is the defense that makes techniques like Hell's Gate detectable if the return address does not originate from a legitimate DLL.

For a deeper look at Nirvana, this article is worth reading: [Detecting Manual Syscalls from User Mode](https://winternl.com/detecting-manual-syscalls-from-user-mode/)

### How does Nirvana work?

Every time the kernel returns to user-mode, it checks whether the `KPROCESS!InstrumentationCallback` member is not NULL. If it points to valid memory, the kernel replaces the RIP in the trap frame with the value stored in the callback.

Any syscall that does not return to a known and valid location is considered evasive. Direct syscalls from ntdll.dll are detectable precisely because the return address lands in a known legitimate DLL. To bypass Nirvana, indirect syscall techniques become necessary.

## Indirect Syscalls

Indirect syscalls work similarly to direct syscalls: the assembly stub must be manually crafted, but the syscall instruction is not included in it. Instead, execution is redirected to a syscall instruction that already exists in ntdll.dll's memory space.

![Indirect syscalls diagram](/assets/img/HellsHall/indirect_syscalls.png)

When security solutions activate the instrumentation callback, they see the syscall as originating from within ntdll.dll and classify it as legitimate, even though the attacker's code initiated it.

## Hell's Hall: Technical Implementation

To implement Hell's Hall, the NT_SYSCALL structure is extended to store not only the syscall address in ntdll.dll but also the address of a syscall instruction located in a different DLL, typically kernel32.dll. For more detail in code here is the [Github of MalDev Academy](https://github.com/Maldev-Academy/HellHall)

### Structure Definition

```c
typedef struct {
    DWORD SSN;                      // Service Number extracted from ntdll
    PVOID pNtdllSyscallAddress;     // Hell's Gate: syscall instruction in ntdll
    PVOID pSyscallInstAddress;      // Hell's Hall: syscall instruction in kernel32
} NT_SYSCALL;
```

The key addition is `pSyscallInstAddress`, which holds the address of a syscall instruction found elsewhere in the process memory space.

Relevant bytecode:

```
0x0F 0x05 = syscall instruction
0x90      = nop (padding, commonly found before syscall for alignment)
```

### The FetchNtSyscall Function

The modified `FetchNtSyscall` function performs two operations:

- Hell's Gate phase: extracts the SSN from ntdll.dll as before.
- Hell's Hall phase: searches for a syscall instruction in kernel32.dll and caches its address.

```c
NT_SYSCALL FetchNtSyscall(LPCSTR lpFunctionName) {
    NT_SYSCALL syscall = {0};

    // Phase 1: Hell's Gate - extract SSN from ntdll
    syscall.SSN = ExtractSSNFromNtdll(lpFunctionName);
    syscall.pNtdllSyscallAddress = (PVOID)&<inline syscall in ntdll>;

    // Phase 2: Hell's Hall - search for syscall instruction in kernel32
    HMODULE hKernel32 = GetModuleHandleA("kernel32.dll");
    if (hKernel32) {
        syscall.pSyscallInstAddress = FindSyscallInstruction(hKernel32);
    }

    return syscall;
}
```

### Finding the Syscall Instruction

`FindSyscallInstruction` scans the executable sections (typically `.text`) of a target DLL, searching for the raw bytecode `0x0F 0x05`, the machine code for the syscall instruction on x64.

```c
PVOID FindSyscallInstruction(HMODULE hDll) {
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)hDll;
    PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)((PBYTE)hDll + pDosHeader->e_lfanew);
    PIMAGE_SECTION_HEADER pSectionHeader = (PIMAGE_SECTION_HEADER)((PBYTE)pNtHeaders + sizeof(IMAGE_NT_HEADERS));

    for (int i = 0; i < pNtHeaders->FileHeader.NumberOfSections; i++) {
        if (!strcmp((char*)pSectionHeader[i].Name, ".text")) {
            PBYTE pSectionBase = (PBYTE)hDll + pSectionHeader[i].VirtualAddress;
            DWORD dwSectionSize = pSectionHeader[i].Misc.VirtualSize;

            for (DWORD j = 0; j < dwSectionSize - 1; j++) {
                if (pSectionBase[j] == 0x0F && pSectionBase[j+1] == 0x05) {
                    return (PVOID)&pSectionBase[j];
                }
            }
        }
    }

    return NULL;
}
```

How it works:

- Parse the PE header to locate section information.
- Iterate through all sections in the module.
- Focus on the `.text` section, which contains executable code.
- Scan byte by byte searching for `0x0F 0x05`.
- Return the first occurrence found.

By caching this address, when the syscall is executed using the extracted SSN, execution jumps to `pSyscallInstAddress` (kernel32's syscall) instead of ntdll's. The kernel's instrumentation callback sees the return address pointing to kernel32.dll and considers it legitimate, even though an arbitrary syscall with a manually crafted SSN is being executed.

## Evasion Mechanism and Current Defenses

Hell's Gate and Hell's Hall both extract the SSN from ntdll.dll but differ in execution.

Hell's Gate's flaw is that the syscall instruction is also executed from within ntdll.dll. When the kernel returns to user-mode, the InstrumentationCallback sees RIP pointing back into ntdll.dll, the same DLL responsible for the syscall. A manually crafted SSN breaking the expected return address pattern is a red flag.

Hell's Hall's workaround exploits the fact that the kernel does not validate which DLL executes the syscall, only that the return address appears to originate from a known legitimate DLL. By jumping to a syscall instruction in kernel32.dll instead of ntdll.dll, the InstrumentationCallback check passes.

Modern defenses have evolved to counter this:

- Comprehensive DLL instrumentation: if an EDR instruments all system DLLs, not just ntdll.dll, arbitrary syscalls become detectable regardless of return address origin.
- Behavioral analysis: security tools can flag anomalies such as kernel32.dll executing NtCreateThreadEx at an unusual code offset, which legitimate code does not do.
- Return address validation: more sophisticated callbacks validate not just which DLL but which function within that DLL is making the syscall.

## The Evolution Continues: Toward VEH Syscalls

Hell's Hall represents a significant step in the evasion arms race, but by 2024 even more advanced techniques emerged. VEH Syscalls abandon the traditional model entirely.

Instead of executing syscalls from a known DLL, VEH syscalls leverage Windows exception handling mechanisms to trigger syscalls in ways that bypass kernel instrumentation callbacks altogether. Rather than finding a legitimate-looking place to execute, the technique avoids the detection infrastructure entirely.

That is the final post in this series: Syscall Series #4 - VEH Syscalls.

## References

- [Hell's Hall code](https://github.com/Maldev-Academy/HellHall)
- [MalDev Academy](https://maldevacademy.com/)