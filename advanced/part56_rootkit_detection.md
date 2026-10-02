# Part 56 - Rootkit Detection Concepts in Nim

## คำเตือนสำคัญ / Important Disclaimer

> เนื้อหานี้มีไว้เพื่อการศึกษา Defensive Security, Kernel research และ Blue Team Operations เท่านั้น
> เน้นการ **ตรวจจับ** rootkits ไม่ใช่การสร้าง

---

## บทนำ Rootkit Detection

Rootkit คือ malware ที่ซ่อนตัวเองในระบบ การตรวจจับต้องใช้เทคนิคพิเศษเพราะ rootkit ปกปิด API calls ปกติ

### ประเภท Rootkits
- **User-mode rootkit** — Hook Win32 API, DLL injection
- **Kernel-mode rootkit** — DKOM (Direct Kernel Object Manipulation)
- **Bootkits** — Infect MBR/UEFI
- **Hypervisor rootkits** — Run below OS

---

## 1. Process Hiding Detection

```nim
# process_hide_detector.nim
# Detect hidden processes by comparing multiple enumeration methods
# Educational/Defensive tool

import std/[strformat, sets, tables, strutils]

when defined(windows):
  import winim/lean

  type
    ProcessEntry* = object
      pid*: DWORD
      name*: string
      method*: string  # How it was found

  # Method 1: Toolhelp32 snapshot (can be hooked)
  proc enumByToolhelp*(): seq[ProcessEntry] =
    result = @[]
    let snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
    if snap == INVALID_HANDLE_VALUE: return
    defer: CloseHandle(snap)
    
    var pe: PROCESSENTRY32
    pe.dwSize = sizeof(PROCESSENTRY32).DWORD
    
    if Process32First(snap, addr pe):
      result.add(ProcessEntry(
        pid: pe.th32ProcessID,
        name: $cast[cstring](addr pe.szExeFile[0]),
        method: "Toolhelp32"
      ))
      while Process32Next(snap, addr pe):
        result.add(ProcessEntry(
          pid: pe.th32ProcessID,
          name: $cast[cstring](addr pe.szExeFile[0]),
          method: "Toolhelp32"
        ))

  # Method 2: EnumProcesses from PSAPI
  proc enumByPsapi*(): seq[ProcessEntry] =
    result = @[]
    var pids: array[1024, DWORD]
    var bytesReturned: DWORD
    
    if not EnumProcesses(addr pids[0], sizeof(pids).DWORD, addr bytesReturned):
      return
    
    let count = bytesReturned div sizeof(DWORD).DWORD
    for i in 0..<count:
      let pid = pids[i]
      var name = ""
      
      let hProc = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, FALSE, pid)
      if hProc != 0:
        var buf: array[MAX_PATH, WCHAR]
        var size = MAX_PATH.DWORD
        if QueryFullProcessImageNameW(hProc, 0, buf, addr size):
          name = $cast[WideCString](addr buf[0])
        CloseHandle(hProc)
      
      result.add(ProcessEntry(pid: pid, name: name, method: "PSAPI"))

  # Method 3: NtQuerySystemInformation (lower-level, harder to hook)
  proc enumByNtQuery*(): seq[ProcessEntry] =
    result = @[]
    
    # NtQuerySystemInformation prototype
    type
      NtQuerySystemInformationFn = proc(
        SystemInformationClass: ULONG,
        SystemInformation: PVOID,
        SystemInformationLength: ULONG,
        ReturnLength: PULONG
      ): NTSTATUS {.stdcall.}
      
      SYSTEM_PROCESS_INFORMATION {.pure.} = object
        NextEntryOffset: ULONG
        NumberOfThreads: ULONG
        Reserved1: array[48, byte]
        ImageName: UNICODE_STRING
        BasePriority: LONG
        UniqueProcessId: HANDLE
        InheritedFromUniqueProcessId: HANDLE
        HandleCount: ULONG
        SessionId: ULONG
    
    let ntdll = GetModuleHandleW("ntdll.dll")
    if ntdll == 0: return
    
    let fn = cast[NtQuerySystemInformationFn](
      GetProcAddress(ntdll, "NtQuerySystemInformation")
    )
    if fn == nil: return
    
    var size: ULONG = 1024 * 1024  # 1MB initial buffer
    var buf = newSeq[byte](size)
    
    while true:
      let status = fn(5, addr buf[0], size, addr size)  # 5 = SystemProcessInformation
      if status == 0: break  # STATUS_SUCCESS
      if status == 0xC0000004:  # STATUS_INFO_LENGTH_MISMATCH
        buf.setLen(size)
      else:
        return
    
    var offset = 0
    while offset < buf.len:
      let spi = cast[ptr SYSTEM_PROCESS_INFORMATION](addr buf[offset])
      
      var name = ""
      if spi.ImageName.Buffer != nil:
        name = $spi.ImageName.Buffer
      
      result.add(ProcessEntry(
        pid: cast[DWORD](spi.UniqueProcessId),
        name: name,
        method: "NtQuery"
      ))
      
      if spi.NextEntryOffset == 0: break
      offset += spi.NextEntryOffset.int

  proc compareEnumerations*(): seq[tuple[pid: DWORD, hiddenFrom: seq[string]]] =
    ## Compare process lists from different methods to find hidden processes
    result = @[]
    
    let byToolhelp = enumByToolhelp()
    let byPsapi = enumByPsapi()
    let byNtQuery = enumByNtQuery()
    
    # Build sets of PIDs
    var toolhelpPids: HashSet[DWORD]
    var psapiPids: HashSet[DWORD]
    var ntqueryPids: HashSet[DWORD]
    
    for p in byToolhelp: toolhelpPids.incl(p.pid)
    for p in byPsapi: psapiPids.incl(p.pid)
    for p in byNtQuery: ntqueryPids.incl(p.pid)
    
    # Find processes visible to NtQuery but hidden from others
    for pid in ntqueryPids:
      var hiddenFrom: seq[string] = @[]
      if pid notin toolhelpPids:
        hiddenFrom.add("Toolhelp32")
      if pid notin psapiPids:
        hiddenFrom.add("PSAPI")
      
      if hiddenFrom.len > 0:
        result.add((pid, hiddenFrom))

when isMainModule:
  when defined(windows):
    echo "Process Hiding Detector"
    echo "========================"
    echo "Comparing process enumeration methods...\n"
    
    let hidden = compareEnumerations()
    
    if hidden.len > 0:
      echo &"[!] Found {hidden.len} potentially hidden processes:"
      for (pid, methods) in hidden:
        echo &"  PID {pid}: hidden from {methods.join(", ")}"
    else:
      echo "[+] All process enumeration methods agree."
      echo "    (Does not guarantee no rootkit - just no simple DKOM)"
  else:
    echo "Windows-specific tool."
```

---

## 2. SSDT Hook Detector

```nim
# ssdt_hook_detector.nim
# Detect System Service Descriptor Table hooks (conceptual/educational)
# Real detection requires kernel driver; this shows the concepts

import std/[strformat, strutils]

## เกี่ยวกับ SSDT
## System Service Descriptor Table (SSDT) คือตารางที่ Windows kernel ใช้
## map syscall numbers ไปยัง kernel function addresses
## Rootkits สามารถ modify entries ใน SSDT เพื่อ redirect syscalls

type
  SsdtEntry* = object
    index*: int
    name*: string
    expectedAddr*: uint64
    actualAddr*: uint64
    isHooked*: bool
    hookTarget*: string

# Known Windows syscall numbers (Windows 10 1903)
# These change between Windows versions!
const KnownSyscalls* = [
  (0x00, "NtAccessCheck"),
  (0x01, "NtWorkerFactoryWorkerReady"),
  (0x02, "NtAcceptConnectPort"),
  (0x03, "NtMapUserPhysicalPagesScatter"),
  (0x04, "NtWaitForSingleObject"),
  (0x05, "NtCallbackReturn"),
  (0x06, "NtReadFile"),
  (0x07, "NtDeviceIoControlFile"),
  (0x08, "NtWriteFile"),
  (0x09, "NtRemoveIoCompletion"),
  (0x0A, "NtReleaseSemaphore"),
  (0x0B, "NtReplyWaitReceivePort"),
  (0x0C, "NtReplyPort"),
  (0x0D, "NtSetInformationThread"),
  (0x0E, "NtSetEvent"),
  (0x0F, "NtClose"),
  (0x10, "NtQueryObject"),
  (0x11, "NtQueryInformationFile"),
  (0x12, "NtOpenKey"),
  (0x13, "NtEnumerateValueKey"),
  (0x14, "NtFindAtom"),
  (0x15, "NtQueryDefaultLocale"),
  (0x16, "NtQueryKey"),
  (0x17, "NtQueryValueKey"),
  (0x18, "NtAllocateVirtualMemory"),
  (0x19, "NtQueryInformationProcess"),
  (0x1A, "NtWaitForMultipleObjects32"),
  (0x1B, "NtWriteFileGather"),
  (0x1C, "NtSetInformationProcess"),
  (0x1D, "NtCreateKey"),
  (0x1E, "NtFreeVirtualMemory"),
  (0x1F, "NtImpersonateClientOfPort"),
  (0x20, "NtReleaseMutant"),
  (0x21, "NtQueryInformationToken"),
  (0x22, "NtRequestWaitReplyPort"),
  (0x23, "NtQueryVirtualMemory"),
  (0x24, "NtOpenThreadToken"),
  (0x25, "NtQueryInformationThread"),
  (0x26, "NtOpenProcess"),
  (0x27, "NtSetInformationFile"),
  (0x28, "NtMapViewOfSection"),
  (0x29, "NtAccessCheckAndAuditAlarm"),
  (0x2A, "NtUnmapViewOfSection"),
  (0x2B, "NtReplyWaitReceivePortEx"),
  (0x2C, "NtTerminateProcess"),
  (0x2D, "NtSetEventBoostPriority"),
  (0x2E, "NtReadFileScatter"),
  (0x2F, "NtOpenThreadTokenEx"),
  (0x30, "NtOpenProcessTokenEx"),
  (0x31, "NtQueryPerformanceCounter"),
  (0x32, "NtEnumerateKey"),
  (0x33, "NtOpenFile"),
  (0x34, "NtDelayExecution"),
  (0x35, "NtQueryDirectoryFile"),
  (0x36, "NtQuerySystemInformation"),
  (0x37, "NtOpenSection"),
  (0x38, "NtQueryTimer"),
  (0x39, "NtFsControlFile"),
  (0x3A, "NtWriteVirtualMemory"),
  (0x3B, "NtCloseObjectAuditAlarm"),
  (0x3C, "NtDuplicateObject"),
  (0x3D, "NtQueryAttributesFile"),
  (0x3E, "NtClearEvent"),
]

## ใน user-mode เราไม่สามารถ read SSDT โดยตรงได้
## แต่เราสามารถตรวจสอบ inline hooks ใน ntdll.dll ได้

when defined(windows):
  import winim/lean

  proc checkNtdllHooks*(): seq[tuple[funcName: string, hooked: bool, firstBytes: seq[byte]]] =
    ## Check for inline hooks in ntdll.dll syscall stubs
    result = @[]
    
    let ntdll = GetModuleHandleW("ntdll.dll")
    if ntdll == 0: return
    
    for (index, funcName) in KnownSyscalls:
      let fn = GetProcAddress(ntdll, funcName.cstring)
      if fn == nil: continue
      
      # Read first 5 bytes of the function
      var firstBytes = newSeq[byte](10)
      var bytesRead: SIZE_T
      discard ReadProcessMemory(
        GetCurrentProcess(),
        fn,
        addr firstBytes[0],
        10,
        addr bytesRead
      )
      
      # Normal syscall stub looks like:
      # 4C 8B D1    mov r10, rcx
      # B8 XX XX XX XX   mov eax, <syscall_number>
      # F6 04 25 08 03 FE 7F 01   test byte [...]
      # 75 03        jne ...
      # 0F 05        syscall
      
      # A hook usually starts with E9 (JMP) or FF 25 (JMP [RIP+...])
      let isHooked = firstBytes[0] == 0xE9 or  # JMP rel32
                     (firstBytes[0] == 0xFF and firstBytes[1] == 0x25)  # JMP [RIP+...]
      
      # Also check if the syscall number is correct
      var expectedSyscall = false
      if firstBytes[0] == 0x4C and firstBytes[1] == 0x8B and firstBytes[2] == 0xD1:
        if firstBytes[3] == 0xB8:  # MOV EAX, imm32
          let actualSyscallNum = (firstBytes[7].int shl 24) or
                                  (firstBytes[6].int shl 16) or
                                  (firstBytes[5].int shl 8) or
                                  firstBytes[4].int
          expectedSyscall = actualSyscallNum == index
      
      result.add((funcName, isHooked or not expectedSyscall, firstBytes))

  proc reportNtdllHooks*() =
    let hooks = checkNtdllHooks()
    var hookedCount = 0
    
    echo "NTDLL Syscall Hook Detector"
    echo "============================"
    echo &"Checking {hooks.len} syscall stubs...\n"
    
    for (name, hooked, bytes) in hooks:
      if hooked:
        inc hookedCount
        let hexBytes = bytes[0..4].mapIt(&"{it:02X}").join(" ")
        echo &"[HOOKED] {name}"
        echo &"         First bytes: {hexBytes}"
    
    echo ""
    if hookedCount == 0:
      echo "[+] No hooks detected in NTDLL syscall stubs."
    else:
      echo &"[!] {hookedCount} hooked syscalls detected!"
      echo "    This may indicate EDR/AV instrumentation or a rootkit."
      echo "    Note: Security software legitimately hooks these for monitoring."

# --- Linux Rootkit Detection ---

when defined(linux):
  proc checkKernelModules*(): seq[tuple[name: string, suspicious: bool, reason: string]] =
    ## Check loaded kernel modules for suspicious ones
    result = @[]
    
    if not fileExists("/proc/modules"): return
    
    let suspiciousNames = [
      "hide", "rootkit", "rkit", "evil", "stealth",
      "ghost", "phantom", "backdoor"
    ]
    
    for line in lines("/proc/modules"):
      let parts = line.splitWhitespace()
      if parts.len < 2: continue
      let name = parts[0].toLower()
      
      var suspicious = false
      var reason = ""
      
      for s in suspiciousNames:
        if s in name:
          suspicious = true
          reason = &"Name contains suspicious keyword: '{s}'"
          break
      
      # Module with no dependencies and not loaded by another module
      if not suspicious and parts.len >= 4 and parts[2] == "0" and parts[3] == "-":
        # Manual load, no dependencies - worth checking
        discard  # Additional checks could go here
      
      result.add((parts[0], suspicious, reason))

  proc checkSyscallTable*() =
    ## Conceptual - reading syscall table requires kernel module
    echo "Syscall table inspection requires kernel module."
    echo "Use tools like 'rkhunter' or 'chkrootkit' for userspace checks."
    echo ""
    echo "Key files to verify integrity:"
    let criticalFiles = [
      "/bin/ls", "/bin/ps", "/bin/netstat",
      "/bin/ss", "/usr/bin/top", "/sbin/ifconfig"
    ]
    for f in criticalFiles:
      if fileExists(f):
        let info = getFileInfo(f)
        echo &"  {f}: {info.size} bytes, modified {info.lastWriteTime}"

# --- Anti-Forensic Detection ---

proc detectAntiForensics*() =
  ## Detect common anti-forensic techniques
  echo "\n=== Anti-Forensic Technique Detection ==="
  
  when defined(windows):
    import winim/lean
    
    # Check for timestomping indicators
    echo "[*] Checking for timestamp manipulation..."
    echo "    Compare $STANDARD_INFORMATION vs $FILE_NAME timestamps in MFT"
    echo "    (Requires direct MFT access)"
    
    # Check for large time gaps in event logs
    echo "[*] Check Windows Event Log for gaps indicating log clearing"
    echo "    Look for EventID 1102 (Security log cleared) or 104 (System log)"
    
    # Prefetch analysis
    let prefetchDir = "C:\\Windows\\Prefetch"
    if dirExists(prefetchDir):
      var prefetchCount = 0
      for f in walkFiles(prefetchDir & "\\*.pf"):
        inc prefetchCount
      echo &"[*] Prefetch files: {prefetchCount}"
      if prefetchCount == 0:
        echo "    [!] No prefetch files - may indicate cleaning or disabled prefetch"
  
  when defined(linux):
    # Check bash history
    let histFile = getHomeDir() / ".bash_history"
    if fileExists(histFile):
      let content = readFile(histFile)
      if content.len == 0:
        echo "[!] .bash_history is empty - possible deletion"
    else:
      echo "[!] .bash_history missing"
    
    # Check for log clearing
    let logFiles = ["/var/log/auth.log", "/var/log/syslog", "/var/log/messages"]
    for logFile in logFiles:
      if fileExists(logFile):
        let info = getFileInfo(logFile)
        if info.size == 0:
          echo &"[!] Empty log file: {logFile}"
      else:
        echo &"[!] Missing log file: {logFile}"

when isMainModule:
  echo "Rootkit Detection Toolkit (Educational)"
  echo "======================================="
  echo "For defensive use and security research only.\n"
  
  when defined(windows):
    reportNtdllHooks()
  
  when defined(linux):
    echo "Loaded Kernel Modules:"
    let modules = checkKernelModules()
    var suspiciousCount = 0
    for (name, susp, reason) in modules:
      if susp:
        inc suspiciousCount
        echo &"  [!] {name}: {reason}"
    if suspiciousCount == 0:
      echo "  [+] No obviously suspicious modules."
    
    checkSyscallTable()
  
  detectAntiForensics()
```

---

## 3. Network Artifact Detection

```nim
# network_artifact_detector.nim
# Detect suspicious network artifacts for incident response

import std/[strformat, strutils, sets, tables]

when defined(windows):
  import winim/lean

  type
    NetworkConnection* = object
      protocol*: string
      localAddr*: string
      localPort*: int
      remoteAddr*: string
      remotePort*: int
      state*: string
      pid*: DWORD
      processName*: string

  proc getNetworkConnections*(): seq[NetworkConnection] =
    ## Get current TCP connections using GetExtendedTcpTable
    result = @[]
    
    type
      MIB_TCPROW_OWNER_PID {.pure.} = object
        dwState: DWORD
        dwLocalAddr: DWORD
        dwLocalPort: DWORD
        dwRemoteAddr: DWORD
        dwRemotePort: DWORD
        dwOwningPid: DWORD
      
      MIB_TCPTABLE_OWNER_PID {.pure.} = object
        dwNumEntries: DWORD
        table: array[1, MIB_TCPROW_OWNER_PID]
    
    var bufSize: DWORD = 0
    discard GetExtendedTcpTable(nil, addr bufSize, TRUE, AF_INET, TCP_TABLE_OWNER_PID_ALL, 0)
    
    var buf = newSeq[byte](bufSize)
    if GetExtendedTcpTable(addr buf[0], addr bufSize, TRUE, AF_INET, TCP_TABLE_OWNER_PID_ALL, 0) != NO_ERROR:
      return
    
    let table = cast[ptr MIB_TCPTABLE_OWNER_PID](addr buf[0])
    
    for i in 0..<table.dwNumEntries.int:
      let row = table.table[i]
      
      proc addrToStr(a: DWORD): string =
        let b = cast[array[4, byte]](a)
        &"{b[0]}.{b[1]}.{b[2]}.{b[3]}"
      
      let state = case row.dwState
        of 1: "CLOSED"
        of 2: "LISTENING"
        of 3: "SYN_SENT"
        of 4: "SYN_RCVD"
        of 5: "ESTABLISHED"
        of 6: "FIN_WAIT_1"
        of 7: "FIN_WAIT_2"
        of 8: "CLOSE_WAIT"
        of 9: "CLOSING"
        of 10: "LAST_ACK"
        of 11: "TIME_WAIT"
        of 12: "DELETE_TCB"
        else: "UNKNOWN"
      
      result.add(NetworkConnection(
        protocol: "TCP",
        localAddr: addrToStr(row.dwLocalAddr),
        localPort: (row.dwLocalPort shr 8) or (row.dwLocalPort shl 8) and 0xFFFF,
        remoteAddr: addrToStr(row.dwRemoteAddr),
        remotePort: (row.dwRemotePort shr 8) or (row.dwRemotePort shl 8) and 0xFFFF,
        state: state,
        pid: row.dwOwningPid
      ))

  proc detectC2Patterns*(conns: seq[NetworkConnection]): seq[string] =
    ## Detect potential C2 communication patterns
    result = @[]
    
    # Suspicious ports often used by C2
    let suspiciousPorts = {4444, 4445, 1234, 31337, 8888, 9999, 6666, 7777}.toHashSet()
    
    # Count connections per process
    var connsByPid: Table[DWORD, int]
    for c in conns:
      connsByPid[c.pid] = connsByPid.getOrDefault(c.pid, 0) + 1
    
    for c in conns:
      if c.state != "ESTABLISHED": continue
      
      # High port to high port (unusual for legitimate traffic)
      if c.localPort > 49152 and c.remotePort > 49152:
        result.add(&"[MEDIUM] High-high port connection: {c.localAddr}:{c.localPort} -> {c.remoteAddr}:{c.remotePort} (PID {c.pid})")
      
      # Suspicious remote ports
      if c.remotePort in suspiciousPorts:
        result.add(&"[HIGH] Suspicious remote port {c.remotePort}: {c.localAddr} -> {c.remoteAddr} (PID {c.pid})")
      
      # Many connections from single process
      if connsByPid.getOrDefault(c.pid, 0) > 20:
        result.add(&"[MEDIUM] PID {c.pid} has {connsByPid[c.pid]} connections")

when defined(linux):
  proc parseSSOutput*(): seq[tuple[local, remote, state, pid: string]] =
    ## Parse /proc/net/tcp for connection info
    result = @[]
    
    if not fileExists("/proc/net/tcp"): return
    
    for line in lines("/proc/net/tcp"):
      let parts = line.splitWhitespace()
      if parts.len < 10 or parts[0] == "sl": continue
      
      proc hexToAddr(h: string): string =
        let parts = h.split(":")
        if parts.len != 2: return h
        let addrInt = parseHexInt(parts[0]).uint32
        let port = parseHexInt(parts[1])
        let b = cast[array[4, byte]](addrInt)
        &"{b[3]}.{b[2]}.{b[1]}.{b[0]}:{port}"
      
      let state = case parts[3]
        of "01": "ESTABLISHED"
        of "02": "SYN_SENT"
        of "03": "SYN_RECV"
        of "04": "FIN_WAIT1"
        of "05": "FIN_WAIT2"
        of "06": "TIME_WAIT"
        of "07": "CLOSE"
        of "08": "CLOSE_WAIT"
        of "09": "LAST_ACK"
        of "0A": "LISTEN"
        of "0B": "CLOSING"
        else: parts[3]
      
      result.add((
        hexToAddr(parts[1]),
        hexToAddr(parts[2]),
        state,
        parts[7]  # uid
      ))

when isMainModule:
  echo "Network Artifact Detector"
  echo "========================="
  
  when defined(windows):
    let conns = getNetworkConnections()
    echo &"Active TCP connections: {conns.len}\n"
    
    let anomalies = detectC2Patterns(conns)
    if anomalies.len > 0:
      echo "[!] Suspicious patterns found:"
      for a in anomalies:
        echo &"  {a}"
    else:
      echo "[+] No obvious C2 patterns detected."
  
  when defined(linux):
    let conns = parseSSOutput()
    echo &"TCP connections found: {conns.len}"
    for (local, remote, state, uid) in conns:
      if state == "ESTABLISHED":
        echo &"  {local} -> {remote} [{state}] uid={uid}"
```

---

## 4. Integrity Checker

```nim
# integrity_checker.nim
# Monitor file system integrity for rootkit/tampering detection

import std/[os, times, strformat, tables, hashes, md5]

type
  FileBaseline* = object
    path*: string
    size*: int64
    modTime*: Time
    md5Hash*: string

  IntegrityReport* = object
    newFiles*: seq[string]
    deletedFiles*: seq[string]
    modifiedFiles*: seq[tuple[path: string, reason: string]]
    checkedAt*: DateTime

proc computeMD5*(path: string): string =
  try:
    let data = readFile(path)
    result = getMD5(data)
  except:
    result = "ERROR"

proc buildBaseline*(directories: seq[string]): Table[string, FileBaseline] =
  result = initTable[string, FileBaseline]()
  
  for dir in directories:
    for filePath in walkDirRec(dir):
      try:
        let info = getFileInfo(filePath)
        result[filePath] = FileBaseline(
          path: filePath,
          size: info.size,
          modTime: info.lastWriteTime,
          md5Hash: computeMD5(filePath)
        )
      except:
        discard

proc saveBaseline*(baseline: Table[string, FileBaseline], outputPath: string) =
  var content = ""
  for path, bl in baseline:
    content.add(&"{path}|{bl.size}|{bl.modTime.toUnix()}|{bl.md5Hash}\n")
  writeFile(outputPath, content)

proc loadBaseline*(inputPath: string): Table[string, FileBaseline] =
  result = initTable[string, FileBaseline]()
  
  for line in lines(inputPath):
    let parts = line.split("|")
    if parts.len < 4: continue
    
    result[parts[0]] = FileBaseline(
      path: parts[0],
      size: parseInt(parts[1]),
      modTime: fromUnix(parseInt(parts[2])),
      md5Hash: parts[3]
    )

proc compareBaseline*(old: Table[string, FileBaseline],
                      current: Table[string, FileBaseline]): IntegrityReport =
  result.checkedAt = now()
  result.newFiles = @[]
  result.deletedFiles = @[]
  result.modifiedFiles = @[]
  
  # Check for deleted and modified
  for path, oldInfo in old:
    if path notin current:
      result.deletedFiles.add(path)
    else:
      let newInfo = current[path]
      if newInfo.md5Hash != oldInfo.md5Hash:
        result.modifiedFiles.add((path, "MD5 hash changed"))
      elif newInfo.size != oldInfo.size:
        result.modifiedFiles.add((path, "Size changed"))
      elif newInfo.modTime != oldInfo.modTime:
        result.modifiedFiles.add((path, "Modification time changed"))
  
  # Check for new files
  for path in current.keys:
    if path notin old:
      result.newFiles.add(path)

proc printReport*(report: IntegrityReport) =
  echo &"\n=== Integrity Check Report ==="
  echo &"Checked at: {report.checkedAt.format(\"yyyy-MM-dd HH:mm:ss\")}"
  echo ""
  
  if report.newFiles.len > 0:
    echo &"[NEW FILES] {report.newFiles.len}:"
    for f in report.newFiles:
      echo &"  + {f}"
  
  if report.deletedFiles.len > 0:
    echo &"\n[DELETED FILES] {report.deletedFiles.len}:"
    for f in report.deletedFiles:
      echo &"  - {f}"
  
  if report.modifiedFiles.len > 0:
    echo &"\n[MODIFIED FILES] {report.modifiedFiles.len}:"
    for (f, reason) in report.modifiedFiles:
      echo &"  ~ {f}: {reason}"
  
  if report.newFiles.len == 0 and report.deletedFiles.len == 0 and report.modifiedFiles.len == 0:
    echo "[+] No integrity violations detected."

when isMainModule:
  import std/parseopt
  
  var mode = ""
  var dirs: seq[string] = @[]
  var baselineFile = "baseline.db"
  
  for kind, key, val in getopt():
    case kind
    of cmdLongOption, cmdShortOption:
      case key
      of "create", "c": mode = "create"
      of "check", "v": mode = "check"
      of "baseline", "b": baselineFile = val
    of cmdArgument:
      dirs.add(key)
    else: discard
  
  if dirs.len == 0:
    dirs = @["/etc", "/bin", "/usr/bin"]
  
  case mode
  of "create":
    echo &"Creating baseline for: {dirs.join(", ")}"
    let baseline = buildBaseline(dirs)
    saveBaseline(baseline, baselineFile)
    echo &"Baseline saved: {baseline.len} files -> {baselineFile}"
  of "check":
    echo &"Checking integrity against: {baselineFile}"
    let oldBaseline = loadBaseline(baselineFile)
    let currentBaseline = buildBaseline(dirs)
    let report = compareBaseline(oldBaseline, currentBaseline)
    printReport(report)
  else:
    echo "File Integrity Monitor"
    echo "Usage:"
    echo "  --create   Build initial baseline"
    echo "  --check    Check against existing baseline"
    echo "  --baseline <file>  Baseline file path"
```

---

## สรุป Part 56

| เครื่องมือ | จุดประสงค์ |
|-----------|----------|
| `process_hide_detector.nim` | ตรวจจับ hidden processes ด้วย cross-method comparison |
| `ssdt_hook_detector.nim` | ตรวจสอบ syscall hooks ใน ntdll.dll |
| `network_artifact_detector.nim` | ค้นหา suspicious network connections |
| `integrity_checker.nim` | Monitor file system integrity changes |

### Defensive Takeaways

1. **Cross-method comparison** — Rootkits ไม่สามารถ hook ทุก API พร้อมกัน
2. **Kernel-level visibility** — User-mode detection มีข้อจำกัด
3. **Baseline + comparison** — วิธีที่ดีที่สุดสำหรับ integrity monitoring
4. **Multiple data sources** — Correlate filesystem, network, process data

**Next**: [Part 57 - gRPC in Nim](../web/part57_grpc.md)
