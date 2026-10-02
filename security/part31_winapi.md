# Part 31: Windows API Advanced - การใช้งาน Win32 API ขั้นสูง

> **หมายเหตุ**: เนื้อหาส่วนนี้มีไว้เพื่อการศึกษาด้าน Windows internals, security research, pentesting
> และการพัฒนา defensive security tools เท่านั้น ใช้ในขอบเขตที่ได้รับอนุญาตเสมอ

## Process Management

```nim
when defined(windows):
  import winlean, strutils, strformat

  type
    STARTUPINFOA* {.pure.} = object
      cb*: DWORD
      lpReserved*, lpDesktop*, lpTitle*: cstring
      dwX*, dwY*, dwXSize*, dwYSize*: DWORD
      dwXCountChars*, dwYCountChars*: DWORD
      dwFillAttribute*, dwFlags*: DWORD
      wShowWindow*, cbReserved2*: WORD
      lpReserved2*: ptr BYTE
      hStdInput*, hStdOutput*, hStdError*: HANDLE

    PROCESS_INFORMATION* {.pure.} = object
      hProcess*, hThread*: HANDLE
      dwProcessId*, dwThreadId*: DWORD

    SECURITY_ATTRIBUTES* {.pure.} = object
      nLength*: DWORD
      lpSecurityDescriptor*: pointer
      bInheritHandle*: BOOL

  proc CreateProcessA*(
    lpApplicationName: cstring, lpCommandLine: cstring,
    lpProcessAttributes: ptr SECURITY_ATTRIBUTES,
    lpThreadAttributes: ptr SECURITY_ATTRIBUTES,
    bInheritHandles: BOOL, dwCreationFlags: DWORD,
    lpEnvironment: pointer, lpCurrentDirectory: cstring,
    lpStartupInfo: ptr STARTUPINFOA,
    lpProcessInformation: ptr PROCESS_INFORMATION
  ): BOOL {.importc, dynlib: "kernel32".}

  proc WaitForSingleObject*(hHandle: HANDLE, dwMilliseconds: DWORD): DWORD
    {.importc, dynlib: "kernel32".}

  proc launchProcess(cmdLine: string, waitForCompletion: bool = false): (DWORD, HANDLE) =
    var si: STARTUPINFOA
    si.cb = sizeof(STARTUPINFOA).DWORD
    var pi: PROCESS_INFORMATION
    var cmd = cmdLine
    let success = CreateProcessA(
      nil, cmd, nil, nil, FALSE, 0.DWORD, nil, nil, addr si, addr pi
    )
    if success == FALSE:
      raise newException(OSError, "CreateProcess failed")
    if waitForCompletion:
      discard WaitForSingleObject(pi.hProcess, 0xFFFFFFFF.DWORD)
    (pi.dwProcessId, pi.hProcess)

  let (pid, hProc) = launchProcess("notepad.exe", false)
  echo &"Started notepad.exe PID: {pid}"
  discard CloseHandle(hProc)
```

## Registry Operations

```nim
when defined(windows):
  type
    HKEY* = pointer
    REGSAM* = DWORD
    LSTATUS* = int32

  const
    KEY_READ* = 0x20019.DWORD
    KEY_WRITE* = 0x20006.DWORD
    REG_SZ* = 1.DWORD
    REG_DWORD* = 4.DWORD
    ERROR_SUCCESS* = 0.LSTATUS

  proc RegOpenKeyExA*(
    hKey: HKEY, lpSubKey: cstring, ulOptions: DWORD,
    samDesired: REGSAM, phkResult: ptr HKEY
  ): LSTATUS {.importc, dynlib: "advapi32".}
  
  proc RegQueryValueExA*(
    hKey: HKEY, lpValueName: cstring, lpReserved: ptr DWORD,
    lpType: ptr DWORD, lpData: pointer, lpcbData: ptr DWORD
  ): LSTATUS {.importc, dynlib: "advapi32".}
  
  proc RegSetValueExA*(
    hKey: HKEY, lpValueName: cstring, Reserved: DWORD,
    dwType: DWORD, lpData: pointer, cbData: DWORD
  ): LSTATUS {.importc, dynlib: "advapi32".}
  
  proc RegCloseKey*(hKey: HKEY): LSTATUS
    {.importc, dynlib: "advapi32".}
  
  proc RegCreateKeyExA*(
    hKey: HKEY, lpSubKey: cstring, Reserved: DWORD, lpClass: cstring,
    dwOptions: DWORD, samDesired: REGSAM,
    lpSecurityAttributes: ptr SECURITY_ATTRIBUTES,
    phkResult: ptr HKEY, lpdwDisposition: ptr DWORD
  ): LSTATUS {.importc, dynlib: "advapi32".}

  proc readRegistryString(hive: HKEY, path, name: string): Option[string] =
    import options
    var hKey: HKEY
    if RegOpenKeyExA(hive, path, 0, KEY_READ, addr hKey) != ERROR_SUCCESS:
      return none(string)
    defer: discard RegCloseKey(hKey)
    var dataType: DWORD
    var dataSize: DWORD = 256
    var data = newString(256)
    if RegQueryValueExA(hKey, name, nil, addr dataType, 
                        addr data[0], addr dataSize) != ERROR_SUCCESS:
      return none(string)
    data.setLen(dataSize.int - 1)
    some(data)

  proc writeRegistryString(hive: HKEY, path, name, value: string): bool =
    var hKey: HKEY
    var disposition: DWORD
    if RegCreateKeyExA(hive, path, 0, nil, 0, KEY_WRITE, 
                       nil, addr hKey, addr disposition) != ERROR_SUCCESS:
      return false
    defer: discard RegCloseKey(hKey)
    RegSetValueExA(hKey, name, 0, REG_SZ, 
                   unsafeAddr value[0], (value.len + 1).DWORD) == ERROR_SUCCESS
```

## Thread Operations

```nim
when defined(windows):
  type
    LPTHREAD_START_ROUTINE* = proc(param: pointer): DWORD {.stdcall.}

  proc CreateThread*(
    lpThreadAttributes: ptr SECURITY_ATTRIBUTES, dwStackSize: SIZE_T,
    lpStartAddress: LPTHREAD_START_ROUTINE, lpParameter: pointer,
    dwCreationFlags: DWORD, lpThreadId: ptr DWORD
  ): HANDLE {.importc, dynlib: "kernel32".}
  
  proc SuspendThread*(hThread: HANDLE): DWORD
    {.importc, dynlib: "kernel32".}
  
  proc ResumeThread*(hThread: HANDLE): DWORD
    {.importc, dynlib: "kernel32".}

  proc threadProc(param: pointer): DWORD {.stdcall.} =
    echo "Thread running! Param: ", cast[int](param)
    return 0

  var threadId: DWORD
  let hThread = CreateThread(nil, 0, threadProc, cast[pointer](42), 0, addr threadId)
  if hThread != nil:
    echo "Thread ID: ", threadId
    discard WaitForSingleObject(hThread, 0xFFFFFFFF.DWORD)
    discard CloseHandle(hThread)
```

## File System Operations (Win32)

```nim
when defined(windows):
  const
    GENERIC_READ* = 0x80000000.DWORD
    GENERIC_WRITE* = 0x40000000.DWORD
    FILE_SHARE_READ* = 0x00000001.DWORD
    OPEN_EXISTING* = 3.DWORD
    CREATE_ALWAYS* = 2.DWORD
    FILE_ATTRIBUTE_NORMAL* = 0x80.DWORD

  proc CreateFileA*(
    lpFileName: cstring, dwDesiredAccess: DWORD, dwShareMode: DWORD,
    lpSecurityAttributes: ptr SECURITY_ATTRIBUTES,
    dwCreationDisposition: DWORD, dwFlagsAndAttributes: DWORD,
    hTemplateFile: HANDLE
  ): HANDLE {.importc, dynlib: "kernel32".}
  
  proc ReadFile*(
    hFile: HANDLE, lpBuffer: pointer, nNumberOfBytesToRead: DWORD,
    lpNumberOfBytesRead: ptr DWORD, lpOverlapped: pointer
  ): BOOL {.importc, dynlib: "kernel32".}
  
  proc WriteFile*(
    hFile: HANDLE, lpBuffer: pointer, nNumberOfBytesToWrite: DWORD,
    lpNumberOfBytesWritten: ptr DWORD, lpOverlapped: pointer
  ): BOOL {.importc, dynlib: "kernel32".}
  
  proc GetFileSize*(hFile: HANDLE, lpFileSizeHigh: ptr DWORD): DWORD
    {.importc, dynlib: "kernel32".}

  proc readFileWin32(path: string): seq[byte] =
    let hFile = CreateFileA(path, GENERIC_READ, FILE_SHARE_READ, nil,
                            OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nil)
    if hFile == INVALID_HANDLE_VALUE:
      raise newException(IOError, "Cannot open file: " & path)
    defer: discard CloseHandle(hFile)
    let fileSize = GetFileSize(hFile, nil)
    result = newSeq[byte](fileSize)
    var bytesRead: DWORD
    discard ReadFile(hFile, addr result[0], fileSize, addr bytesRead, nil)
    result.setLen(bytesRead.int)
```

## Pipe และ IPC (Command Execution)

```nim
when defined(windows):
  proc CreatePipe*(
    hReadPipe, hWritePipe: ptr HANDLE,
    lpPipeAttributes: ptr SECURITY_ATTRIBUTES, nSize: DWORD
  ): BOOL {.importc, dynlib: "kernel32".}

  proc execWithOutput(cmd: string): string =
    var hReadPipe, hWritePipe: HANDLE
    var sa: SECURITY_ATTRIBUTES
    sa.nLength = sizeof(SECURITY_ATTRIBUTES).DWORD
    sa.bInheritHandle = TRUE
    if CreatePipe(addr hReadPipe, addr hWritePipe, addr sa, 0) == FALSE:
      raise newException(OSError, "CreatePipe failed")
    defer:
      discard CloseHandle(hReadPipe)
      discard CloseHandle(hWritePipe)
    var si: STARTUPINFOA
    si.cb = sizeof(STARTUPINFOA).DWORD
    si.dwFlags = 0x00000001.DWORD  # STARTF_USESTDHANDLES
    si.hStdOutput = hWritePipe
    si.hStdError = hWritePipe
    var pi: PROCESS_INFORMATION
    var cmdBuf = "cmd.exe /c " & cmd
    if CreateProcessA(nil, cmdBuf, nil, nil, TRUE, 0, nil, nil, addr si, addr pi) == FALSE:
      raise newException(OSError, "CreateProcess failed")
    discard CloseHandle(hWritePipe)
    discard WaitForSingleObject(pi.hProcess, 0xFFFFFFFF.DWORD)
    discard CloseHandle(pi.hProcess)
    discard CloseHandle(pi.hThread)
    result = ""
    var buf = newString(4096)
    var bytesRead: DWORD
    while ReadFile(hReadPipe, addr buf[0], 4096, addr bytesRead, nil) == TRUE and bytesRead > 0:
      result &= buf[0..<bytesRead.int]
  
  let output = execWithOutput("whoami")
  echo "Current user: ", output.strip()
```

## สรุป Part 31

ในบทนี้เราได้เรียนรู้:
- ✅ Process creation (CreateProcess)
- ✅ Registry read/write
- ✅ Thread creation และ enumeration
- ✅ File system operations (Win32 API level)
- ✅ Pipe และ IPC (command execution with output capture)

---

**Previous**: [Part 30 - Windows Internals](part30_windows_internals.md)
**Next**: [Part 32 - Shellcode Basics](part32_shellcode.md)
