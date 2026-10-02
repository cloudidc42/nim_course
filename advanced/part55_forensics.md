# Part 55 - Memory Forensics Tools in Nim

## คำเตือนสำคัญ / Important Disclaimer

> เนื้อหานี้มีไว้เพื่อการศึกษา, การวิจัยด้านความปลอดภัย, CTF competitions และ Digital Forensics เท่านั้น
> ห้ามนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตโดยเด็ดขาด

---

## บทนำ Memory Forensics

Memory forensics คือการวิเคราะห์ RAM dump เพื่อค้นหาหลักฐานทางดิจิทัล, malware artifacts, และ process information

### เครื่องมือ Memory Forensics ที่นิยม
- **Volatility** — Python-based framework
- **Rekall** — Memory analysis framework
- **WinPmem** — Memory acquisition
- **LiME** — Linux Memory Extractor

---

## 1. Memory Dump Parser

```nim
# memory_parser.nim
# Parse raw memory dumps for forensic analysis

import std/[streams, endians, strformat, tables, sets, sequtils, algorithm]

type
  MemoryRegion* = object
    startAddr*: uint64
    endAddr*: uint64
    size*: uint64
    permissions*: string
    mappedFile*: string

  ProcessInfo* = object
    pid*: uint32
    ppid*: uint32
    name*: string
    cmdline*: string
    baseAddr*: uint64

  MemoryDump* = object
    data*: seq[byte]
    baseAddress*: uint64
    size*: uint64
    arch*: string  # "x86", "x64"

proc loadDump*(path: string): MemoryDump =
  ## Load a raw memory dump from file
  let fs = newFileStream(path, fmRead)
  if fs == nil:
    raise newException(IOError, "Cannot open dump: " & path)
  defer: fs.close()
  
  result.data = @[]
  let fileSize = fs.getPosition()
  fs.setPosition(0)
  
  var buf: array[4096, byte]
  while not fs.atEnd():
    let n = fs.readData(addr buf[0], buf.len)
    result.data.add(buf[0..<n])
  
  result.size = result.data.len.uint64
  result.baseAddress = 0
  result.arch = "x64"

proc readU32*(dump: MemoryDump, offset: uint64): uint32 =
  ## Read 4-byte little-endian value
  if offset + 4 > dump.size:
    raise newException(RangeDefect, &"Read out of bounds: {offset:#x}")
  let p = addr dump.data[offset]
  littleEndian32(addr result, p)

proc readU64*(dump: MemoryDump, offset: uint64): uint64 =
  ## Read 8-byte little-endian value
  if offset + 8 > dump.size:
    raise newException(RangeDefect, &"Read out of bounds: {offset:#x}")
  let p = addr dump.data[offset]
  littleEndian64(addr result, p)

proc readString*(dump: MemoryDump, offset: uint64, maxLen: int = 256): string =
  ## Read null-terminated ASCII string
  result = ""
  var i = offset
  while i < dump.size and result.len < maxLen:
    let c = dump.data[i]
    if c == 0: break
    result.add(chr(c))
    inc i

proc readWideString*(dump: MemoryDump, offset: uint64, maxChars: int = 256): string =
  ## Read null-terminated wide (UTF-16LE) string
  result = ""
  var i = offset
  while i + 1 < dump.size and result.len < maxChars:
    let lo = dump.data[i]
    let hi = dump.data[i + 1]
    if lo == 0 and hi == 0: break
    if hi == 0:  # Basic ASCII range
      result.add(chr(lo))
    i += 2

# --- Pattern Searching ---

proc searchPattern*(dump: MemoryDump, pattern: seq[byte], mask: seq[byte] = @[]): seq[uint64] =
  ## Search for byte pattern with optional mask (0xFF = must match, 0x00 = wildcard)
  result = @[]
  let patLen = pattern.len
  if patLen == 0: return
  
  for i in 0'u64..<(dump.size - patLen.uint64):
    var match = true
    for j in 0..<patLen:
      let m = if mask.len > j: mask[j] else: 0xFF.byte
      if (dump.data[i + j.uint64] and m) != (pattern[j] and m):
        match = false
        break
    if match:
      result.add(i)

proc searchString*(dump: MemoryDump, s: string, caseSensitive = true): seq[uint64] =
  ## Search for ASCII string in dump
  var pat = newSeq[byte](s.len)
  for i, c in s:
    pat[i] = if caseSensitive: c.byte else: c.toLowerAscii.byte
  result = dump.searchPattern(pat)

# --- Windows-specific Structures ---

const
  EPROCESS_SIGNATURE* = [0x03'u8, 0x00, 0x1B, 0x00]  # Pool tag simplified
  MZ_MAGIC* = [0x4D'u8, 0x5A]  # MZ header

type
  PEHeader* = object
    magic*: uint16        # MZ
    peOffset*: uint32
    machine*: uint16
    numSections*: uint16
    timestamp*: uint32
    entryPoint*: uint32
    imageBase*: uint64
    imageSize*: uint32
    sectionNames*: seq[string]

proc parsePEHeader*(dump: MemoryDump, offset: uint64): PEHeader =
  ## Parse PE header at given offset
  result.magic = dump.readU32(offset).uint16
  if result.magic != 0x5A4D:  # 'MZ'
    raise newException(ValueError, "Not a valid PE file")
  
  result.peOffset = dump.readU32(offset + 0x3C)
  let peBase = offset + result.peOffset
  
  # PE signature check
  let peSig = dump.readU32(peBase)
  if peSig != 0x00004550:  # 'PE\0\0'
    raise newException(ValueError, "Invalid PE signature")
  
  result.machine = dump.readU32(peBase + 4).uint16
  result.numSections = dump.readU32(peBase + 6).uint16
  result.timestamp = dump.readU32(peBase + 8)
  
  # Optional header
  let optHdrOffset = peBase + 24
  let magic = dump.readU32(optHdrOffset).uint16
  
  if magic == 0x020B:  # PE32+
    result.entryPoint = dump.readU32(optHdrOffset + 16)
    result.imageBase = dump.readU64(optHdrOffset + 24)
    result.imageSize = dump.readU32(optHdrOffset + 56)
  elif magic == 0x010B:  # PE32
    result.entryPoint = dump.readU32(optHdrOffset + 16)
    result.imageBase = dump.readU32(optHdrOffset + 28).uint64
    result.imageSize = dump.readU32(optHdrOffset + 56)

proc findPEHeaders*(dump: MemoryDump): seq[tuple[offset: uint64, pe: PEHeader]] =
  ## Find all PE headers in memory dump
  result = @[]
  let mzOffsets = dump.searchPattern(@[0x4D'u8, 0x5A])
  
  for offset in mzOffsets:
    try:
      let pe = dump.parsePEHeader(offset)
      result.add((offset, pe))
    except:
      discard  # Not a valid PE

# --- String Extraction ---

type
  ExtractedString* = object
    offset*: uint64
    value*: string
    encoding*: string  # "ascii", "unicode"
    isPrintable*: bool

proc isPrintableAscii*(c: byte): bool =
  c >= 0x20 and c <= 0x7E

proc extractStrings*(dump: MemoryDump, minLen: int = 4): seq[ExtractedString] =
  ## Extract printable ASCII and Unicode strings
  result = @[]
  
  # ASCII strings
  var current = ""
  var startOffset = 0'u64
  
  for i in 0'u64..<dump.size:
    let c = dump.data[i]
    if isPrintableAscii(c):
      if current.len == 0:
        startOffset = i
      current.add(chr(c))
    else:
      if current.len >= minLen:
        result.add(ExtractedString(
          offset: startOffset,
          value: current,
          encoding: "ascii",
          isPrintable: true
        ))
      current = ""
  
  # Unicode strings (UTF-16LE)
  current = ""
  startOffset = 0
  var i = 0'u64
  while i + 1 < dump.size:
    let lo = dump.data[i]
    let hi = dump.data[i + 1]
    if isPrintableAscii(lo) and hi == 0:
      if current.len == 0:
        startOffset = i
      current.add(chr(lo))
      i += 2
    else:
      if current.len >= minLen:
        result.add(ExtractedString(
          offset: startOffset,
          value: current,
          encoding: "unicode",
          isPrintable: true
        ))
      current = ""
      inc i

# --- Heuristic Analysis ---

type
  ForensicFinding* = object
    category*: string
    description*: string
    offset*: uint64
    severity*: string  # "low", "medium", "high"
    evidence*: string

proc detectSuspiciousPatterns*(dump: MemoryDump): seq[ForensicFinding] =
  result = @[]
  
  # Look for common malware indicators
  let indicators = [
    ("cmd.exe", "Shell command execution", "high"),
    ("powershell", "PowerShell execution", "high"),
    ("WScript", "Script engine usage", "medium"),
    ("PAYLOAD", "Payload string found", "high"),
    ("inject", "Injection-related string", "medium"),
    ("VirtualAlloc", "Memory allocation API", "low"),
    ("WriteProcessMemory", "Remote process write", "high"),
    ("CreateRemoteThread", "Remote thread creation", "high"),
    ("LoadLibrary", "Dynamic library loading", "low"),
    ("GetProcAddress", "Dynamic function resolution", "medium"),
  ]
  
  for (pattern, desc, severity) in indicators:
    let offsets = dump.searchString(pattern, caseSensitive = false)
    for offset in offsets:
      result.add(ForensicFinding(
        category: "API/String Indicator",
        description: desc,
        offset: offset,
        severity: severity,
        evidence: pattern
      ))
  
  # High entropy regions (possible encrypted/packed content)
  let chunkSize = 4096
  var offset = 0'u64
  while offset + chunkSize.uint64 < dump.size:
    var freq: array[256, int]
    for i in 0..<chunkSize:
      inc freq[dump.data[offset + i.uint64]]
    
    var entropy = 0.0
    for count in freq:
      if count > 0:
        let p = count.float / chunkSize.float
        entropy -= p * (p.ln / 2.0.ln)
    
    if entropy > 7.5:  # Very high entropy
      result.add(ForensicFinding(
        category: "High Entropy",
        description: &"High entropy region (entropy={entropy:.2f}), possible packed/encrypted data",
        offset: offset,
        severity: "medium",
        evidence: &"{chunkSize} bytes at offset {offset:#x}"
      ))
    
    offset += chunkSize.uint64

# --- Report Generation ---

proc generateReport*(dump: MemoryDump, outputPath: string) =
  ## Generate a forensic analysis report
  var report = ""
  
  report.add("# Memory Forensics Report\n\n")
  report.add(&"Dump size: {dump.size} bytes ({dump.size div 1024 div 1024} MB)\n")
  report.add(&"Architecture: {dump.arch}\n\n")
  
  # PE Headers found
  report.add("## Embedded Executables\n\n")
  let peHeaders = dump.findPEHeaders()
  report.add(&"Found {peHeaders.len} PE headers\n\n")
  for (offset, pe) in peHeaders:
    report.add(&"- Offset: {offset:#x}\n")
    report.add(&"  Machine: {pe.machine:#x}\n")
    report.add(&"  Image Base: {pe.imageBase:#x}\n")
    report.add(&"  Timestamp: {pe.timestamp}\n\n")
  
  # Suspicious patterns
  report.add("## Suspicious Patterns\n\n")
  let findings = dump.detectSuspiciousPatterns()
  let highFindings = findings.filterIt(it.severity == "high")
  report.add(&"Total findings: {findings.len} (High: {highFindings.len})\n\n")
  
  for f in findings.sortedByIt(it.severity):
    report.add(&"- [{f.severity.toUpperAscii}] {f.description}\n")
    report.add(&"  Offset: {f.offset:#x} | Evidence: {f.evidence}\n")
  
  # Interesting strings
  report.add("\n## Interesting Strings (sample)\n\n")
  let strings = dump.extractStrings(minLen = 8)
  let interesting = strings.filterIt(
    it.value.contains("http") or
    it.value.contains("cmd") or
    it.value.contains(".exe") or
    it.value.contains("password")
  )
  
  for s in interesting[0..min(49, interesting.len-1)]:
    report.add(&"- [{s.encoding}] @ {s.offset:#x}: {s.value}\n")
  
  writeFile(outputPath, report)
  echo &"Report saved to: {outputPath}"

when isMainModule:
  echo "Memory Forensics Tool"
  echo "Usage: memory_parser <dump_file> [output_report]"
  echo ""
  echo "This tool analyzes memory dumps for forensic evidence."
  echo "Supports: PE header extraction, string analysis, pattern detection"
```

---

## 2. Process Memory Scanner

```nim
# process_scanner.nim
# Scan running processes for forensic artifacts (Windows)
# ⚠️ Requires elevated privileges for some operations

import std/[os, strformat, times, tables]

when defined(windows):
  import winim/lean

  type
    ProcessSnapshot* = object
      pid*: DWORD
      name*: string
      parentPid*: DWORD
      threads*: int
      modules*: seq[string]
      regions*: seq[MemoryRegionInfo]

    MemoryRegionInfo* = object
      baseAddr*: uint64
      size*: uint64
      state*: DWORD
      protect*: DWORD
      mType*: DWORD
      mappedFile*: string

  proc protectToString*(protect: DWORD): string =
    case protect
    of PAGE_EXECUTE: "X"
    of PAGE_EXECUTE_READ: "RX"
    of PAGE_EXECUTE_READWRITE: "RWX"
    of PAGE_EXECUTE_WRITECOPY: "RWX(copy)"
    of PAGE_READONLY: "R"
    of PAGE_READWRITE: "RW"
    of PAGE_WRITECOPY: "W(copy)"
    else: &"{protect:#x}"

  proc listProcesses*(): seq[tuple[pid: DWORD, name: string]] =
    result = @[]
    let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
    if snapshot == INVALID_HANDLE_VALUE: return
    defer: CloseHandle(snapshot)
    
    var pe32: PROCESSENTRY32
    pe32.dwSize = sizeof(PROCESSENTRY32).DWORD
    
    if Process32First(snapshot, addr pe32):
      result.add((pe32.th32ProcessID, $cast[cstring](addr pe32.szExeFile[0])))
      while Process32Next(snapshot, addr pe32):
        result.add((pe32.th32ProcessID, $cast[cstring](addr pe32.szExeFile[0])))

  proc getProcessModules*(pid: DWORD): seq[string] =
    result = @[]
    let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE or TH32CS_SNAPMODULE32, pid)
    if snapshot == INVALID_HANDLE_VALUE: return
    defer: CloseHandle(snapshot)
    
    var me32: MODULEENTRY32
    me32.dwSize = sizeof(MODULEENTRY32).DWORD
    
    if Module32First(snapshot, addr me32):
      result.add($cast[cstring](addr me32.szExePath[0]))
      while Module32Next(snapshot, addr me32):
        result.add($cast[cstring](addr me32.szExePath[0]))

  proc scanProcessMemory*(pid: DWORD): seq[MemoryRegionInfo] =
    result = @[]
    let hProc = OpenProcess(PROCESS_QUERY_INFORMATION or PROCESS_VM_READ, FALSE, pid)
    if hProc == 0: return
    defer: CloseHandle(hProc)
    
    var mbi: MEMORY_BASIC_INFORMATION
    var addr64: uint64 = 0
    
    while VirtualQueryEx(hProc, cast[LPCVOID](addr64), addr mbi, sizeof(mbi).SIZE_T) > 0:
      var info = MemoryRegionInfo(
        baseAddr: cast[uint64](mbi.BaseAddress),
        size: cast[uint64](mbi.RegionSize),
        state: mbi.State,
        protect: mbi.Protect,
        mType: mbi.Type
      )
      result.add(info)
      addr64 = cast[uint64](mbi.BaseAddress) + cast[uint64](mbi.RegionSize)
      if addr64 == 0: break  # Wraparound

  proc detectSuspiciousRegions*(regions: seq[MemoryRegionInfo]): seq[string] =
    ## Find suspicious memory regions (RWX, private executable, etc.)
    result = @[]
    
    for r in regions:
      if r.state != MEM_COMMIT: continue
      
      # Executable + Writable (classic shellcode indicator)
      if (r.protect and PAGE_EXECUTE_READWRITE) != 0:
        result.add(&"[HIGH] RWX region at {r.baseAddr:#x} size={r.size}")
      
      # Private executable memory (no mapped file)
      if (r.protect and PAGE_EXECUTE_READ) != 0 and 
         r.mType == MEM_PRIVATE and r.mappedFile == "":
        result.add(&"[MEDIUM] Private RX region at {r.baseAddr:#x} size={r.size}")
      
      # Very large committed regions
      if r.size > 100 * 1024 * 1024:  # > 100MB
        result.add(&"[LOW] Large region {r.size div 1024 div 1024}MB at {r.baseAddr:#x}")

when isMainModule:
  when defined(windows):
    echo "Process Memory Scanner"
    echo "======================="
    
    let procs = listProcesses()
    echo &"Found {procs.len} running processes\n"
    
    # Show suspicious processes
    for (pid, name) in procs:
      let regions = scanProcessMemory(pid)
      let suspicious = detectSuspiciousRegions(regions)
      if suspicious.len > 0:
        echo &"\n[!] {name} (PID: {pid})"
        for s in suspicious:
          echo &"    {s}"
  else:
    echo "This tool is Windows-specific."
    echo "On Linux, use /proc/<pid>/maps for memory analysis."
```

---

## 3. Linux /proc Memory Analyzer

```nim
# linux_proc_analyzer.nim
# Analyze running processes via /proc filesystem (Linux)

import std/[os, strformat, strutils, tables, sets]

type
  ProcMapEntry* = object
    startAddr*: uint64
    endAddr*: uint64
    permissions*: string
    offset*: uint64
    device*: string
    inode*: uint64
    pathname*: string

  ProcInfo* = object
    pid*: int
    name*: string
    cmdline*: string
    state*: char
    ppid*: int
    maps*: seq[ProcMapEntry]
    openFiles*: seq[string]
    networkConns*: seq[string]

proc parseProcMaps*(pid: int): seq[ProcMapEntry] =
  result = @[]
  let mapsPath = &"/proc/{pid}/maps"
  
  if not fileExists(mapsPath): return
  
  for line in lines(mapsPath):
    let parts = line.splitWhitespace()
    if parts.len < 5: continue
    
    let addrParts = parts[0].split("-")
    if addrParts.len != 2: continue
    
    var entry: ProcMapEntry
    entry.startAddr = parseHexInt(addrParts[0]).uint64
    entry.endAddr = parseHexInt(addrParts[1]).uint64
    entry.permissions = parts[1]
    entry.offset = parseHexInt(parts[2]).uint64
    entry.device = parts[3]
    entry.inode = parseInt(parts[4]).uint64
    if parts.len > 5:
      entry.pathname = parts[5]
    
    result.add(entry)

proc getProcInfo*(pid: int): ProcInfo =
  result.pid = pid
  
  # Name from /proc/pid/comm
  let commPath = &"/proc/{pid}/comm"
  if fileExists(commPath):
    result.name = readFile(commPath).strip()
  
  # Cmdline
  let cmdlinePath = &"/proc/{pid}/cmdline"
  if fileExists(cmdlinePath):
    result.cmdline = readFile(cmdlinePath).replace("\0", " ").strip()
  
  # Status for PPID
  let statusPath = &"/proc/{pid}/status"
  if fileExists(statusPath):
    for line in lines(statusPath):
      if line.startsWith("PPid:"):
        result.ppid = parseInt(line.splitWhitespace()[1])
      elif line.startsWith("State:"):
        result.state = line[7]
  
  result.maps = parseProcMaps(pid)
  
  # Open file descriptors
  let fdDir = &"/proc/{pid}/fd"
  if dirExists(fdDir):
    for entry in walkDir(fdDir):
      try:
        let target = expandSymlink(entry.path)
        result.openFiles.add(target)
      except: discard

proc listRunningProcesses*(): seq[int] =
  result = @[]
  for kind, name in walkDir("/proc"):
    if kind == pcDir:
      let base = name.extractFilename()
      try:
        result.add(parseInt(base))
      except: discard

proc findSuspiciousProcesses*(): seq[tuple[pid: int, reason: string]] =
  result = @[]
  
  for pid in listRunningProcesses():
    let info = getProcInfo(pid)
    
    # Process with deleted executable
    for mapEntry in info.maps:
      if mapEntry.pathname.contains("(deleted)"):
        result.add((pid, &"Deleted executable: {mapEntry.pathname}"))
        break
    
    # Process with anonymous executable mappings
    var anonExec = 0
    for mapEntry in info.maps:
      if "x" in mapEntry.permissions and mapEntry.pathname == "" and mapEntry.inode == 0:
        inc anonExec
    if anonExec > 2:
      result.add((pid, &"Anonymous executable mappings: {anonExec}"))
    
    # Suspicious cmdline patterns
    let suspiciousCmds = ["base64", "curl | sh", "wget | sh", "python -c", "perl -e"]
    for pattern in suspiciousCmds:
      if pattern in info.cmdline.toLower():
        result.add((pid, &"Suspicious cmdline pattern: {pattern}"))
        break

proc analyzeProcess*(pid: int) =
  let info = getProcInfo(pid)
  
  echo &"\n=== Process Analysis: PID {pid} ==="
  echo &"Name: {info.name}"
  echo &"Command: {info.cmdline}"
  echo &"State: {info.state}"
  echo &"Parent PID: {info.ppid}"
  
  echo &"\nMemory Regions: {info.maps.len}"
  var execRegions = 0
  var anonExec = 0
  
  for m in info.maps:
    if "x" in m.permissions:
      inc execRegions
      if m.pathname == "":
        inc anonExec
        echo &"  [!] Anonymous executable: {m.startAddr:#x}-{m.endAddr:#x} ({m.permissions})"
  
  echo &"Total executable regions: {execRegions} (anonymous: {anonExec})"
  
  echo &"\nOpen Files: {info.openFiles.len}"
  for f in info.openFiles[0..min(9, info.openFiles.len-1)]:
    echo &"  {f}"

when isMainModule:
  when defined(linux):
    echo "Linux Process Forensic Analyzer"
    echo "================================"
    
    let suspicious = findSuspiciousProcesses()
    if suspicious.len > 0:
      echo &"\n[!] Found {suspicious.len} suspicious processes:"
      for (pid, reason) in suspicious:
        echo &"  PID {pid}: {reason}"
    else:
      echo "\n[+] No obviously suspicious processes found."
    
    # Analyze specific PID from args
    if paramCount() > 0:
      try:
        let targetPid = parseInt(paramStr(1))
        analyzeProcess(targetPid)
      except:
        echo &"Invalid PID: {paramStr(1)}"
  else:
    echo "Linux-specific tool. Use process_scanner.nim on Windows."
```

---

## 4. YARA-like Pattern Matching

```nim
# yara_matcher.nim
# Simplified YARA-like rule engine for memory/file analysis

import std/[re, strformat, strutils, tables, parseutils]

type
  RuleCondition* = enum
    rcAny, rcAll, rcNOf

  StringPattern* = object
    name*: string
    value*: string
    isHex*: bool
    isRegex*: bool
    isWide*: bool
    isNocase*: bool

  YaraRule* = object
    name*: string
    tags*: seq[string]
    meta*: Table[string, string]
    strings*: seq[StringPattern]
    condition*: RuleCondition
    conditionN*: int  # For N-of condition

  MatchResult* = object
    ruleName*: string
    matched*: bool
    matchedStrings*: seq[tuple[name: string, offsets: seq[int]]]

proc parseHexPattern*(hex: string): seq[byte] =
  ## Parse YARA hex pattern like { 4D 5A ?? 00 }
  result = @[]
  let cleaned = hex.replace(" ", "").replace("{", "").replace("}", "")
  var i = 0
  while i < cleaned.len:
    if cleaned[i] == '?':
      result.add(0x00)  # Wildcard placeholder
      i += 2
    else:
      let hexByte = cleaned[i..i+1]
      result.add(parseHexInt(hexByte).byte)
      i += 2

proc matchesPattern*(data: seq[byte], pattern: StringPattern): seq[int] =
  ## Find all offsets where pattern matches in data
  result = @[]
  
  if pattern.isRegex:
    let dataStr = cast[string](data)
    let rex = re(pattern.value)
    var pos = 0
    while pos < dataStr.len:
      let bounds = dataStr.findBounds(rex, pos)
      if bounds.first < 0: break
      result.add(bounds.first)
      pos = bounds.last + 1
    return
  
  if pattern.isHex:
    let hexBytes = parseHexPattern(pattern.value)
    for i in 0..data.len - hexBytes.len:
      var match = true
      for j, b in hexBytes:
        if pattern.value.replace(" ", "")[(j*2)..(j*2+1)] == "??":
          continue  # Wildcard
        if data[i + j] != b:
          match = false
          break
      if match:
        result.add(i)
    return
  
  # Regular string
  var searchStr = pattern.value
  var dataStr = cast[string](data)
  
  if pattern.isNocase:
    searchStr = searchStr.toLower()
    dataStr = dataStr.toLower()
  
  if pattern.isWide:
    # Create wide version
    var wide = ""
    for c in searchStr:
      wide.add(c)
      wide.add('\0')
    searchStr = wide
  
  var pos = 0
  while pos <= dataStr.len - searchStr.len:
    if dataStr[pos..pos + searchStr.len - 1] == searchStr:
      result.add(pos)
    inc pos

proc evaluate*(rule: YaraRule, data: seq[byte]): MatchResult =
  result.ruleName = rule.name
  result.matched = false
  result.matchedStrings = @[]
  
  var matchCount = 0
  
  for sp in rule.strings:
    let offsets = data.matchesPattern(sp)
    if offsets.len > 0:
      result.matchedStrings.add((sp.name, offsets))
      inc matchCount
  
  case rule.condition
  of rcAny:
    result.matched = matchCount > 0
  of rcAll:
    result.matched = matchCount == rule.strings.len
  of rcNOf:
    result.matched = matchCount >= rule.conditionN

# Example rules
proc buildMimeTypeRules*(): seq[YaraRule] =
  result = @[
    YaraRule(
      name: "PDF_File",
      tags: @["document", "pdf"],
      meta: {"description": "PDF document identifier"}.toTable,
      strings: @[
        StringPattern(name: "$header", value: "%PDF-", isHex: false)
      ],
      condition: rcAll
    ),
    YaraRule(
      name: "Windows_Executable",
      tags: @["executable", "windows"],
      meta: {"description": "Windows PE executable"}.toTable,
      strings: @[
        StringPattern(name: "$mz", value: "{ 4D 5A }", isHex: true),
        StringPattern(name: "$pe", value: "{ 50 45 00 00 }", isHex: true)
      ],
      condition: rcAll
    ),
    YaraRule(
      name: "Possible_Shellcode",
      tags: @["shellcode", "suspicious"],
      meta: {"description": "Common shellcode patterns"}.toTable,
      strings: @[
        StringPattern(name: "$nop_sled", value: "{ 90 90 90 90 90 90 90 90 }", isHex: true),
        StringPattern(name: "$int3_bp", value: "{ CC CC CC CC }", isHex: true)
      ],
      condition: rcAny
    ),
  ]

proc runYaraScan*(filePath: string, rules: seq[YaraRule]) =
  echo &"\nScanning: {filePath}"
  let data = cast[seq[byte]](readFile(filePath))
  
  var matchCount = 0
  for rule in rules:
    let result = rule.evaluate(data)
    if result.matched:
      inc matchCount
      echo &"  [MATCH] Rule: {rule.name} (tags: {rule.tags.join(", ")})"
      for (name, offsets) in result.matchedStrings:
        echo &"    String {name} found at: {offsets[0..min(4, offsets.len-1)]}"
  
  if matchCount == 0:
    echo "  [INFO] No rules matched."
  else:
    echo &"  Total: {matchCount} rules matched."

when isMainModule:
  let rules = buildMimeTypeRules()
  echo &"Loaded {rules.len} detection rules"
  
  # Scan files from command args
  if paramCount() > 0:
    for i in 1..paramCount():
      if fileExists(paramStr(i)):
        runYaraScan(paramStr(i), rules)
  else:
    echo "Usage: yara_matcher <file1> [file2] ..."
    echo "\nAvailable rules:"
    for r in rules:
      echo &"  - {r.name}: {r.meta.getOrDefault(\"description\", \"\")}"
```

---

## 5. Timeline Analysis

```nim
# timeline_analyzer.nim
# Build forensic timelines from file system and log artifacts

import std/[os, times, strformat, algorithm, tables, strutils]

type
  TimelineEvent* = object
    timestamp*: DateTime
    eventType*: string  # "created", "modified", "accessed", "log_entry"
    source*: string
    description*: string
    severity*: string

proc collectFileTimeline*(path: string, recursive = true): seq[TimelineEvent] =
  result = @[]
  
  let walkMode = if recursive: {pcFile, pcDir} else: {pcFile}
  
  for kind, filePath in walkDirRec(path):
    if kind == pcFile:
      try:
        let info = getFileInfo(filePath)
        let modTime = info.lastWriteTime.utc()
        
        result.add(TimelineEvent(
          timestamp: modTime,
          eventType: "modified",
          source: "filesystem",
          description: filePath,
          severity: "info"
        ))
      except:
        discard

proc filterByTimeRange*(events: seq[TimelineEvent],
                        startTime, endTime: DateTime): seq[TimelineEvent] =
  events.filterIt(it.timestamp >= startTime and it.timestamp <= endTime)

proc sortByTime*(events: seq[TimelineEvent]): seq[TimelineEvent] =
  result = events
  result.sort do (a, b: TimelineEvent) -> int:
    cmp(a.timestamp, b.timestamp)

proc exportTimeline*(events: seq[TimelineEvent], outputPath: string) =
  var csv = "Timestamp,EventType,Source,Description,Severity\n"
  
  for e in events.sortByTime():
    let ts = e.timestamp.format("yyyy-MM-dd HH:mm:ss")
    csv.add(&"{ts},{e.eventType},{e.source},\"{e.description}\",{e.severity}\n")
  
  writeFile(outputPath, csv)
  echo &"Timeline exported to: {outputPath} ({events.len} events)"

when isMainModule:
  echo "Forensic Timeline Analyzer"
  echo "==========================="
  
  if paramCount() < 1:
    echo "Usage: timeline_analyzer <directory> [output.csv]"
    quit(1)
  
  let targetDir = paramStr(1)
  let outputFile = if paramCount() >= 2: paramStr(2) else: "timeline.csv"
  
  echo &"Collecting timeline from: {targetDir}"
  let events = collectFileTimeline(targetDir)
  echo &"Found {events.len} events"
  
  exportTimeline(events, outputFile)
```

---

## สรุป Part 55

ในส่วนนี้เราได้เรียนรู้:

| เครื่องมือ | ฟังก์ชัน |
|-----------|----------|
| `memory_parser.nim` | Parse raw memory dumps, extract PE headers, strings |
| `process_scanner.nim` | Scan Windows process memory for suspicious regions |
| `linux_proc_analyzer.nim` | Analyze Linux /proc for suspicious processes |
| `yara_matcher.nim` | Pattern matching engine for malware detection |
| `timeline_analyzer.nim` | Build forensic timelines from file system |

### Key Forensic Techniques

1. **Memory carving** — Extract embedded files from raw dumps
2. **String extraction** — Find ASCII/Unicode indicators
3. **Entropy analysis** — Detect packed/encrypted regions
4. **Pattern matching** — YARA-like rule-based scanning
5. **Timeline analysis** — Correlate events across sources

**Next**: [Part 56 - Rootkit Detection Concepts](part56_rootkit_detection.md)
