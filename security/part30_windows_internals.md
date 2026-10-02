# Part 30: Windows Internals - ความรู้พื้นฐาน Windows

## Windows Architecture

```
User Mode                    Kernel Mode
┌─────────────────────────┐  ┌─────────────────────────────┐
│ User Applications        │  │ Executive (ntoskrnl.exe)    │
│ ┌────┐ ┌────┐ ┌────┐   │  │ ┌────────┐ ┌────────────┐  │
│ │App1│ │App2│ │App3│   │  │ │ Process│ │  Memory    │  │
│ └────┘ └────┘ └────┘   │  │ │ Manager│ │  Manager   │  │
│                          │  │ └────────┘ └────────────┘  │
│ Win32 Subsystem (DLLs)  │  │ ┌────────┐ ┌────────────┐  │
│ kernel32.dll, user32.dll│  │ │  I/O   │ │  Security  │  │
│ ntdll.dll               │  │ │Manager │ │  Reference │  │
└──────────────┬──────────┘  └─────────────────────────────┘
               │                          │
               └─── System Call Interface ┘
               
Windows API Layers:
High-level: Win32 API (CreateProcess, VirtualAlloc, etc.)
Low-level:  NT API (NtCreateProcess, NtAllocateVirtualMemory)
Kernel:     Syscalls (via SSDT)
```

## Win32 API ใน Nim

```nim
# Windows API types
when defined(windows):
  import winlean
  
  # Core types
  type
    HANDLE* = pointer
    DWORD* = uint32
    BOOL* = int32
    WORD* = uint16
    BYTE* = uint8
    LPVOID* = pointer
    LPCSTR* = cstring
    LPCWSTR* = ptr uint16
    SIZE_T* = uint
    NTSTATUS* = int32
    ULONG_PTR* = uint
  
  # Constants
  const
    NULL* = nil
    TRUE* = 1.BOOL
    FALSE* = 0.BOOL
    INVALID_HANDLE_VALUE* = cast[HANDLE](-1)
    
    # Process access rights
    PROCESS_ALL_ACCESS* = 0x1F0FFF.DWORD
    PROCESS_VM_READ* = 0x0010.DWORD
    PROCESS_VM_WRITE* = 0x0020.DWORD
    PROCESS_VM_OPERATION* = 0x0008.DWORD
    PROCESS_CREATE_THREAD* = 0x0002.DWORD
    
    # Memory protection constants
    PAGE_READWRITE* = 0x04.DWORD
    PAGE_EXECUTE_READ* = 0x20.DWORD
    PAGE_EXECUTE_READWRITE* = 0x40.DWORD
    
    # Memory allocation
    MEM_COMMIT* = 0x1000.DWORD
    MEM_RESERVE* = 0x2000.DWORD
    MEM_RELEASE* = 0x8000.DWORD
    
    # TH32CS flags
    TH32CS_SNAPPROCESS* = 0x00000002.DWORD
    TH32CS_SNAPTHREAD* = 0x00000004.DWORD
```

## Process Enumeration

```nim
when defined(windows):
  import winlean, strutils, strformat

  type
    PROCESSENTRY32* {.pure.} = object
      dwSize*: DWORD
      cntUsage*: DWORD
      th32ProcessID*: DWORD
      th32DefaultHeapID*: ULONG_PTR
      th32ModuleID*: DWORD
      cntThreads*: DWORD
      th32ParentProcessID*: DWORD
      pcPriClassBase*: int32
      dwFlags*: DWORD
      szExeFile*: array[260, char]

  proc CreateToolhelp32Snapshot*(dwFlags: DWORD, th32ProcessID: DWORD): HANDLE
    {.importc, dynlib: "kernel32".}
  proc Process32First*(hSnapshot: HANDLE, lppe: ptr PROCESSENTRY32): BOOL
    {.importc, dynlib: "kernel32".}
  proc Process32Next*(hSnapshot: HANDLE, lppe: ptr PROCESSENTRY32): BOOL
    {.importc, dynlib: "kernel32".}
  proc CloseHandle*(hObject: HANDLE): BOOL
    {.importc, dynlib: "kernel32".}
  proc OpenProcess*(dwDesiredAccess: DWORD, bInheritHandle: BOOL, dwProcessId: DWORD): HANDLE
    {.importc, dynlib: "kernel32".}

  proc listProcesses(): seq[(DWORD, string)] =
    result = @[]
    let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
    if snapshot == INVALID_HANDLE_VALUE:
      return
    defer: discard CloseHandle(snapshot)
    var pe: PROCESSENTRY32
    pe.dwSize = sizeof(PROCESSENTRY32).DWORD
    if Process32First(snapshot, addr pe) == TRUE:
      while true:
        let procName = $cast[cstring](addr pe.szExeFile[0])
        result.add((pe.th32ProcessID, procName))
        if Process32Next(snapshot, addr pe) != TRUE:
          break
  
  proc findProcess(name: string): Option[DWORD] =
    import options
    let processes = listProcesses()
    for (pid, procName) in processes:
      if procName.toLowerAscii() == name.toLowerAscii():
        return some(pid)
    none(DWORD)

  echo "Running processes:"
  for (pid, name) in listProcesses():
    echo &"  PID: {pid:5d}  Name: {name}"
```

## Memory Operations

```nim
when defined(windows):
  proc VirtualAllocEx*(
    hProcess: HANDLE, lpAddress: LPVOID, 
    dwSize: SIZE_T, flAllocationType, flProtect: DWORD
  ): LPVOID {.importc, dynlib: "kernel32".}
  
  proc WriteProcessMemory*(
    hProcess: HANDLE, lpBaseAddress: LPVOID,
    lpBuffer: pointer, nSize: SIZE_T,
    lpNumberOfBytesWritten: ptr SIZE_T
  ): BOOL {.importc, dynlib: "kernel32".}
  
  proc ReadProcessMemory*(
    hProcess: HANDLE, lpBaseAddress: LPVOID,
    lpBuffer: pointer, nSize: SIZE_T,
    lpNumberOfBytesRead: ptr SIZE_T
  ): BOOL {.importc, dynlib: "kernel32".}
  
  # Read memory from another process
  proc readMemory(hProc: HANDLE, address: pointer, size: int): seq[byte] =
    result = newSeq[byte](size)
    var bytesRead: SIZE_T
    discard ReadProcessMemory(hProc, address, addr result[0], size.SIZE_T, addr bytesRead)
    result.setLen(bytesRead.int)
  
  # Write memory to another process
  proc writeMemory(hProc: HANDLE, address: pointer, data: seq[byte]): bool =
    var bytesWritten: SIZE_T
    WriteProcessMemory(hProc, address, unsafeAddr data[0], 
                       data.len.SIZE_T, addr bytesWritten) == TRUE
```

## PE Format (Portable Executable)

```nim
import os, strutils, strformat

type
  DOSHeader = object
    e_magic: uint16   # MZ
    e_cblp: uint16
    e_cp: uint16
    e_crlc: uint16
    e_cparhdr: uint16
    e_minalloc: uint16
    e_maxalloc: uint16
    e_ss: uint16
    e_sp: uint16
    e_csum: uint16
    e_ip: uint16
    e_cs: uint16
    e_lfarlc: uint16
    e_ovno: uint16
    e_res: array[4, uint16]
    e_oemid: uint16
    e_oeminfo: uint16
    e_res2: array[10, uint16]
    e_lfanew: int32   # Offset to PE header

  FileHeader = object
    machine: uint16
    numberOfSections: uint16
    timeDateStamp: uint32
    pointerToSymbolTable: uint32
    numberOfSymbols: uint32
    sizeOfOptionalHeader: uint16
    characteristics: uint16

  SectionHeader = object
    name: array[8, char]
    virtualSize: uint32
    virtualAddress: uint32
    sizeOfRawData: uint32
    pointerToRawData: uint32
    pointerToRelocations: uint32
    pointerToLinenumbers: uint32
    numberOfRelocations: uint16
    numberOfLinenumbers: uint16
    characteristics: uint32

proc parsePE(filename: string) =
  let data = readFile(filename)
  let dosHdr = cast[ptr DOSHeader](unsafeAddr data[0])
  if dosHdr.e_magic != 0x5A4D:  # MZ
    echo "Not a valid PE file"
    return
  echo "=== PE Analysis: ", filename
  echo "DOS signature: MZ (0x5A4D)"
  echo "PE offset: 0x", dosHdr.e_lfanew.toHex(8)
  echo "Analysis complete"

# parsePE("C:\\Windows\\System32\\notepad.exe")
```

## สรุป Part 30

ในบทนี้เราได้เรียนรู้:
- ✅ Windows Architecture (User/Kernel mode)
- ✅ Win32 API types และ constants
- ✅ Process enumeration
- ✅ Memory operations (read/write)
- ✅ PE file format analysis

---

**Previous**: [Part 29 - Auth](../web/part29_auth.md)
**Next**: [Part 31 - Windows API Advanced](part31_winapi.md)
