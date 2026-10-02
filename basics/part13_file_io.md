# Part 13: File I/O - การอ่านและเขียนไฟล์

## การอ่านไฟล์

```nim
import os, strutils

# อ่านทั้งไฟล์เป็น string
let content = readFile("hello.txt")
echo content

# อ่านทีละบรรทัด
for line in lines("data.txt"):
  echo line

# อ่านด้วย File handle
let f = open("data.txt", fmRead)
defer: f.close()

var line: string
while f.readLine(line):
  echo line

# อ่านเป็น seq[string]
let allLines = readFile("data.txt").splitLines()
for i, line in allLines:
  echo i+1, ": ", line

# อ่าน binary file
let binData = readFile("image.png")
echo "File size: ", binData.len, " bytes"
```

## การเขียนไฟล์

```nim
import os

# เขียนทั้งไฟล์
writeFile("output.txt", "Hello, World!\n")

# เขียนด้วย File handle
let f = open("output.txt", fmWrite)
defer: f.close()

f.writeLine("Line 1")
f.writeLine("Line 2")
f.write("No newline")
f.write("\n")

# เพิ่มต่อท้าย
let f2 = open("output.txt", fmAppend)
defer: f2.close()
f2.writeLine("Appended line")

# เขียน binary
let data = "binary\x00data"
writeFile("binary.bin", data)
```

## File Modes

```nim
# File modes ใน Nim
# fmRead      - read only
# fmWrite     - write (creates/truncates)
# fmAppend    - append
# fmReadWrite - read and write

import os

# สร้างและเขียนไฟล์
var f = open("test.txt", fmWrite)
f.writeLine("Hello")
f.writeLine("World")
f.close()

# อ่านกลับ
f = open("test.txt", fmRead)
echo f.readAll()
f.close()

# ReadWrite mode
f = open("test.txt", fmReadWrite)
let pos = f.getFilePos()  # get current position
f.setFilePos(0)           # seek to beginning
let firstLine = f.readLine()
echo "First line: ", firstLine
f.setFilePos(0, fspEnd)   # seek to end
f.writeLine("New line")
f.close()
```

## Path Operations (os module)

```nim
import os, strutils

# Path manipulation
let path = "/home/user/documents/file.txt"

echo splitPath(path)           # ("/home/user/documents", "file.txt")
echo splitFile(path)           # ("/home/user/documents", "file", ".txt")
echo path.parentDir()          # /home/user/documents
echo path.lastPathPart()       # file.txt
echo path.extractFilename()    # file.txt
echo path.splitFile()[1]       # file (name without ext)
echo path.splitFile()[2]       # .txt (extension)

# Join paths
echo joinPath("/home/user", "documents", "file.txt")
echo "/home/user" / "documents" / "file.txt"  # same

# Normalize
echo normalizePath("./foo/../bar//baz")  # bar/baz

# Check existence
echo fileExists("file.txt")
echo dirExists("/home/user")
echo existsOrCreateDir("newdir")

# File info
if fileExists("file.txt"):
  let info = getFileInfo("file.txt")
  echo "Size: ", info.size
  echo "Created: ", info.creationTime
  echo "Modified: ", info.lastModificationTime

# Current directory
echo getCurrentDir()
setCurrentDir("/tmp")
echo getCurrentDir()

# List directory
for kind, path in walkDir("."):
  case kind
  of pcFile: echo "File: ", path
  of pcDir:  echo "Dir:  ", path
  else: discard

# Recursive directory walk
for path in walkDirRec("."):
  echo path

# Pattern matching
for path in walkFiles("*.nim"):
  echo path

# Create directories
createDir("mydir/subdir")
createDirTree("a/b/c/d")  # create nested (use createDir for newer Nim)

# Delete
removeFile("temp.txt")
removeDir("tempdir")
```

## Streams

```nim
import streams

# String Stream
var ss = newStringStream("Hello, World!")
echo ss.readLine()    # Hello, World!
ss.setPosition(0)
echo ss.readChar()    # H
echo ss.readChar()    # e
ss.write("XYZ")
ss.setPosition(0)
echo ss.readAll()     # HXYZo, World!

# File Stream
var fs = newFileStream("data.txt", fmWrite)
fs.writeLine("Line 1")
fs.writeLine("Line 2")
fs.close()

fs = newFileStream("data.txt", fmRead)
while not fs.atEnd():
  echo fs.readLine()
fs.close()

# Binary Stream
var bs = newFileStream("binary.dat", fmWrite)
bs.write(42'i32)         # write int32
bs.write(3.14'f32)       # write float32
bs.write("hello\0")      # write C string
bs.close()

bs = newFileStream("binary.dat", fmRead)
let n = bs.readInt32()
let f = bs.readFloat32()
echo "n=", n, " f=", f
bs.close()
```

## CSV Processing

```nim
import strutils, sequtils

type
  CsvRecord = seq[string]
  CsvTable  = seq[CsvRecord]

proc readCsv(filename: string, separator: char = ','): CsvTable =
  result = @[]
  for line in lines(filename):
    if line.len == 0: continue
    result.add(line.split(separator))

proc writeCsv(filename: string, table: CsvTable, separator: char = ',') =
  var f = open(filename, fmWrite)
  defer: f.close()
  for row in table:
    f.writeLine(row.join($separator))

proc printTable(table: CsvTable) =
  if table.len == 0: return
  let headers = table[0]
  
  # Calculate column widths
  var maxWidths = headers.mapIt(it.len)
  for row in table[1..^1]:
    for i, cell in row:
      if i < maxWidths.len:
        maxWidths[i] = max(maxWidths[i], cell.len)
  
  # Print
  let sep = "+" & maxWidths.mapIt("-".repeat(it+2)).join("+") & "+"
  echo sep
  for rowIdx, row in table:
    var line = "|"
    for i, cell in row:
      if i < maxWidths.len:
        line &= " " & cell.alignLeft(maxWidths[i]) & " |"
    echo line
    if rowIdx == 0: echo sep
  echo sep

# Test
let csvContent = """Name,Age,City,Score
Alice,30,Bangkok,95.5
Bob,25,Chiang Mai,87.0
Carol,28,Phuket,92.3"""

writeFile("students.csv", csvContent)
let data = readCsv("students.csv")
printTable(data)

# Process data
var totalScore = 0.0
for row in data[1..^1]:  # skip header
  totalScore += parseFloat(row[3])
import strformat
echo &"\nAverage score: {totalScore/(data.len-1).float:.2f}"
```

## JSON File Operations

```nim
import json, os

# Write JSON
let jsonData = %* {
  "name": "Alice",
  "age": 30,
  "skills": ["Nim", "Python", "Rust"],
  "address": {
    "city": "Bangkok",
    "country": "Thailand"
  }
}

writeFile("user.json", jsonData.pretty())

# Read JSON
let rawJson = readFile("user.json")
let parsed = parseJson(rawJson)

echo parsed["name"].getStr()        # Alice
echo parsed["age"].getInt()         # 30
echo parsed["skills"][0].getStr()   # Nim
echo parsed["address"]["city"].getStr()  # Bangkok

# Work with JSON arrays
for skill in parsed["skills"]:
  echo "Skill: ", skill.getStr()

# Check fields
if parsed.hasKey("email"):
  echo "Email: ", parsed["email"].getStr()
else:
  echo "No email field"
```

## Practical: Configuration File Handler

```nim
# config.nim - Simple config file handler

import tables, strutils, os, strformat

type
  Config = object
    data: Table[string, Table[string, string]]
    filename: string

proc newConfig(): Config =
  Config(data: initTable[string, Table[string, string]]())

proc load(cfg: var Config, filename: string): bool =
  cfg.filename = filename
  if not fileExists(filename):
    return false
  
  var currentSection = "default"
  
  for line in lines(filename):
    let trimmed = line.strip()
    
    # Skip empty lines and comments
    if trimmed.len == 0 or trimmed.startsWith('#') or trimmed.startsWith(';'):
      continue
    
    # Section header
    if trimmed.startsWith('[') and trimmed.endsWith(']'):
      currentSection = trimmed[1..^2].strip()
      if currentSection notin cfg.data:
        cfg.data[currentSection] = initTable[string, string]()
      continue
    
    # Key-value pair
    let pos = trimmed.find('=')
    if pos > 0:
      let key = trimmed[0..<pos].strip()
      let val = trimmed[pos+1..^1].strip()
      
      if currentSection notin cfg.data:
        cfg.data[currentSection] = initTable[string, string]()
      
      cfg.data[currentSection][key] = val
  
  true

proc save(cfg: Config) =
  var f = open(cfg.filename, fmWrite)
  defer: f.close()
  
  for section, kvPairs in cfg.data:
    f.writeLine(&"[{section}]")
    for key, val in kvPairs:
      f.writeLine(&"{key} = {val}")
    f.writeLine("")

proc get(cfg: Config, section, key: string, default: string = ""): string =
  if section in cfg.data and key in cfg.data[section]:
    cfg.data[section][key]
  else:
    default

proc set(cfg: var Config, section, key, value: string) =
  if section notin cfg.data:
    cfg.data[section] = initTable[string, string]()
  cfg.data[section][key] = value

proc `$`(cfg: Config): string =
  result = ""
  for section, kvPairs in cfg.data:
    result &= &"[{section}]\n"
    for k, v in kvPairs:
      result &= &"  {k} = {v}\n"

# Test
let configContent = """
# Application Configuration
[database]
host = localhost
port = 5432
name = mydb
user = admin

[server]
host = 0.0.0.0
port = 8080
debug = false

[logging]
level = INFO
file = app.log
"""

writeFile("app.cfg", configContent)

var cfg = newConfig()
discard cfg.load("app.cfg")

echo "DB Host: ", cfg.get("database", "host")
echo "DB Port: ", cfg.get("database", "port")
echo "Server Port: ", cfg.get("server", "port")
echo "Log Level: ", cfg.get("logging", "level")

# Modify and save
cfg.set("server", "port", "9090")
cfg.set("app", "version", "1.0.0")
cfg.save()

echo "\nUpdated config:"
echo cfg
```

## แบบฝึกหัด Part 13

### แบบฝึกหัดที่ 1: File Statistics
```nim
import os, strutils, strformat

proc analyzeFile(path: string) =
  if not fileExists(path):
    echo "File not found: ", path
    return
  
  let content = readFile(path)
  let lines = content.splitLines()
  let words = content.split()
  let chars = content.len
  
  echo &"File: {path}"
  echo &"Lines: {lines.len}"
  echo &"Words: {words.len}"
  echo &"Chars: {chars}"
  echo &"Size:  {getFileInfo(path).size} bytes"

analyzeFile("hello.txt")
```

### แบบฝึกหัดที่ 2: Directory Tree
```nim
import os, strutils

proc printTree(dir: string, indent: string = "") =
  for kind, path in walkDir(dir):
    let name = path.lastPathPart()
    case kind
    of pcFile:
      echo indent & "├── " & name
    of pcDir:
      echo indent & "├── " & name & "/"
      printTree(path, indent & "│   ")
    else: discard

echo "."
printTree(".")
```

## สรุป Part 13

ในบทนี้เราได้เรียนรู้:
- ✅ อ่านไฟล์: readFile, lines, File handle
- ✅ เขียนไฟล์: writeFile, writeLine, write
- ✅ File modes: fmRead, fmWrite, fmAppend
- ✅ Path operations: splitPath, joinPath, walkDir
- ✅ Streams: StringStream, FileStream
- ✅ CSV processing
- ✅ JSON file operations
- ✅ Config file handler

---

**Previous**: [Part 12 - Enums](part12_enums.md)
**Next**: [Part 14 - Error Handling](part14_error_handling.md)
