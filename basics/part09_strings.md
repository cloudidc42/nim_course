# Part 09: Strings - การจัดการข้อความ

## String Basics

```nim
# String declaration
var s1: string = "Hello, World!"
var s2 = "สวัสดีชาวโลก"  # UTF-8 supported
var s3 = ""              # empty string

# String literals
let single = "single line"
let multi = """
Multi
Line
String
"""

let raw = r"C:\Users\name\file.txt"  # raw string (no escape)

# String concatenation
let name = "Alice"
let greeting = "Hello, " & name & "!"
echo greeting  # Hello, Alice!

# String length
echo "Hello".len     # 5 (bytes)
echo "สวัสดี".len     # 18 (UTF-8 bytes, not chars)

import unicode
echo "สวัสดี".runeLen  # 6 (actual characters)
```

## String Operations (strutils)

```nim
import strutils

let s = "  Hello, World!  "

# Whitespace
echo s.strip()           # "Hello, World!"
echo s.strip(leading=true, trailing=false)   # "Hello, World!  "

# Case conversion
echo "hello".toUpper()  # HELLO
echo "HELLO".toLower()  # hello
echo "hello world".capitalizeAscii()  # Hello world

# Check methods
echo "Hello123".isAlphaNumeric()  # true
echo "Hello".isAlphaAscii()       # true
echo "12345".isDigit()            # true
echo "   ".isSpace()              # true

echo "Hello".startsWith("He")    # true
echo "Hello".endsWith("lo")      # true
echo "Hello".contains("ell")     # true
echo "ell" in "Hello"            # true

# Find and replace
echo "Hello World".find("World")  # 6 (index)
echo "Hello World".replace("World", "Nim")  # Hello Nim
echo "aababab".replace("ab", "X")  # aXXX

# Split
echo "a,b,c".split(",")          # @["a", "b", "c"]
echo "one two three".split()     # @["one", "two", "three"]
echo "a::b::c".split("::")       # @["a", "b", "c"]

# Join
echo @["a", "b", "c"].join(",")   # a,b,c
echo @["one", "two"].join(" - ")  # one - two

# Repeat
echo "abc".repeat(3)   # abcabcabc
echo "-".repeat(20)    # --------------------

# Count occurrences
echo "banana".count("an")   # 2
echo "hello".count("l")     # 2

# parseInt, parseFloat
let n = parseInt("42")
let f = parseFloat("3.14")
echo n + 1   # 43
echo f + 1.0  # 4.14

# toString
echo $42     # "42"
echo $3.14   # "3.14"
echo $true   # "true"
```

## String Formatting (strformat)

```nim
import strformat

let name = "Alice"
let age = 30
let score = 95.5

# Basic interpolation
echo &"Name: {name}"           # Name: Alice
echo &"Age: {age}"             # Age: 30
echo &"Score: {score}"         # Score: 95.5

# Number formatting
echo &"{score:.2f}"           # 95.50
echo &"{score:.0f}"           # 96
echo &"{255:b}"                # 11111111 (binary)
echo &"{255:o}"                # 377 (octal)
echo &"{255:x}"                # ff (hex lowercase)
echo &"{255:X}"                # FF (hex uppercase)
echo &"{255:#x}"               # 0xff

# Width and alignment
echo &"{'left':<10}|"         # left      |
echo &"{'right':>10}|"        # right     |
echo &"{42:08}"               # 00000042 (pad with zeros)
echo &"{42:+}"                # +42 (show sign)

# Expression in format
let items = @["a", "b", "c"]
echo &"Count: {items.len}"    # Count: 3
echo &"First: {items[0]}"     # First: a

# Multiline format
let info = &"""
Name:  {name}
Age:   {age}
Score: {score:.1f}
"""
echo info
```

## Unicode สำหรับ Strings

```nim
import unicode

# Unicode string operations
let thai = "สวัสดีชาวโลก"
echo thai.len         # byte length
echo thai.runeLen     # character count (12)

# Iterate over Unicode characters (Runes)
for rune in thai.runes:
  stdout.write(rune)
stdout.write("\n")

# Convert Rune to string
let r = "ก".runeAt(0)
echo r           # 3585 (code point)
echo $r          # ก

# String slicing with Unicode (by rune)
proc runeSlice(s: string, start, stop: int): string =
  var idx = 0
  result = ""
  for i, r in s.runelenPairs:
    if i >= start and i < stop:
      result &= $r

echo runeSlice(thai, 0, 3)  # สวัส

# isAlpha, isDigit สำหรับ Unicode
echo unicode.isAlpha("ก".runeAt(0))  # true
echo unicode.isAlpha("1".runeAt(0))  # false
```

## Regular Expressions

```nim
import re

# Basic matching
let pattern = re"(\d+)-(\d+)-(\d+)"
let date = "2024-01-15"

# Find all
let text = "Dates: 2024-01-15 and 2023-12-31"
for m in text.findAll(pattern):
  echo "Found: ", m

# Replace
echo "foo bar baz".replace(re"\s+", "_")  # foo_bar_baz

# Split with regex
echo "one  two   three".split(re"\s+")  # @["one", "two", "three"]

# Named groups
let emailPattern = re"(?P<user>[^@]+)@(?P<domain>[^@]+)"
let email = "alice@example.com"
let em = email.find(emailPattern)
if em.isSome:
  let match = em.get()
  echo "User: ", match.captures["user"]     # alice
  echo "Domain: ", match.captures["domain"] # example.com
```

## String Builder Pattern

```nim
# String builder สำหรับ efficient string construction
import std/strutils

# ดีมาก (ใช้ string buffer)
proc buildWithBuffer(items: seq[string]): string =
  var buf = newStringOfCap(items.len * 10)  # pre-allocate
  for i, item in items:
    if i > 0: buf.add(", ")
    buf.add(item)
  buf

echo buildWithBuffer(@["apple", "banana", "cherry"])

# StringStream
import std/streams

proc buildWithStream(items: seq[string]): string =
  var ss = newStringStream()
  for i, item in items:
    if i > 0: ss.write(", ")
    ss.write(item)
  ss.data

echo buildWithStream(@["one", "two", "three"])
```

## Common String Algorithms

```nim
import strutils

# Palindrome check
proc isPalindrome(s: string): bool =
  let cleaned = s.toLower().filter(c => c.isAlphaNumeric())
  cleaned == cleaned.reversed()

echo isPalindrome("racecar")     # true
echo isPalindrome("A man a plan a canal Panama")  # true
echo isPalindrome("hello")       # false

# Edit Distance (Levenshtein)
proc editDistance(a, b: string): int =
  let m = a.len
  let n = b.len
  var dp = newSeqWith(m+1, newSeq[int](n+1))
  
  for i in 0..m: dp[i][0] = i
  for j in 0..n: dp[0][j] = j
  
  for i in 1..m:
    for j in 1..n:
      if a[i-1] == b[j-1]:
        dp[i][j] = dp[i-1][j-1]
      else:
        dp[i][j] = 1 + min(dp[i-1][j], min(dp[i][j-1], dp[i-1][j-1]))
  
  dp[m][n]

echo editDistance("kitten", "sitting")  # 3
echo editDistance("hello", "hello")     # 0
```

## String Templates

```nim
# Simple template engine
import strutils, tables

proc renderTemplate(tmpl: string, vars: Table[string, string]): string =
  result = tmpl
  for key, value in vars:
    result = result.replace("{{" & key & "}}", value)

let tmpl = """
Dear {{name}},

Thank you for registering on {{site}}.
Your account is now active.

Best regards,
{{team}}
"""

let vars = {
  "name": "Alice",
  "site": "NimLand",
  "team": "The Nim Team"
}.toTable()

echo renderTemplate(tmpl, vars)
```

## Practical: Log Parser

```nim
# log_parser.nim - Parse log files

import strutils, strformat, sequtils

type
  LogLevel = enum
    DEBUG, INFO, WARN, ERROR, FATAL

  LogEntry = object
    timestamp: string
    level: LogLevel
    message: string
    source: string

proc parseLevel(s: string): LogLevel =
  case s.toUpper()
  of "DEBUG": DEBUG
  of "INFO":  INFO
  of "WARN":  WARN
  of "ERROR": ERROR
  of "FATAL": FATAL
  else:       INFO

proc parseLogLine(line: string): LogEntry =
  # Format: [2024-01-15 10:30:45] [ERROR] [main.nim:42] Message here
  let parts = line.split("] [")
  if parts.len >= 4:
    LogEntry(
      timestamp: parts[0][1..^1],  # remove leading [
      level: parseLevel(parts[1]),
      source: parts[2],
      message: parts[3][0..^2]  # remove trailing ]
    )
  else:
    LogEntry(timestamp: "", level: INFO, message: line, source: "unknown")

proc filterByLevel(entries: seq[LogEntry], minLevel: LogLevel): seq[LogEntry] =
  entries.filterIt(it.level >= minLevel)

# Test
let logData = """
[2024-01-15 10:30:45] [DEBUG] [app.nim:10] Starting application
[2024-01-15 10:30:46] [INFO] [app.nim:25] Server started on port 8080
[2024-01-15 10:31:00] [WARN] [db.nim:55] Connection pool running low
[2024-01-15 10:31:15] [ERROR] [db.nim:78] Database connection failed
[2024-01-15 10:31:20] [FATAL] [db.nim:90] Max retries exceeded
"""

var entries: seq[LogEntry] = @[]
for line in logData.strip().splitLines():
  entries.add(parseLogLine(line))

echo "=== Errors and above ==="
for e in filterByLevel(entries, ERROR):
  echo &"[{e.timestamp}] [{e.level}] {e.message}"

echo "\n=== Statistics ==="
import tables
var counts = initCountTable[LogLevel]()
for e in entries:
  counts.inc(e.level)
for level, count in counts:
  echo &"  {level}: {count}"
```

## แบบฝึกหัด Part 9

### แบบฝึกหัดที่ 1: Password Validator
```nim
import strutils, re

proc validatePassword(password: string): (bool, seq[string]) =
  var errors: seq[string] = @[]
  
  if password.len < 8:
    errors.add("Must be at least 8 characters")
  
  if not password.contains(re"[A-Z]"):
    errors.add("Must contain uppercase letter")
  
  if not password.contains(re"[a-z]"):
    errors.add("Must contain lowercase letter")
  
  if not password.contains(re"[0-9]"):
    errors.add("Must contain a digit")
  
  if not password.contains(re"[!@#$%^&*]"):
    errors.add("Must contain special character (!@#$%^&*)")
  
  (errors.len == 0, errors)

let passwords = ["hello", "Hello123", "Hello123!", "MyP@ss123"]
for pwd in passwords:
  let (valid, errors) = validatePassword(pwd)
  if valid:
    echo pwd, ": Valid"
  else:
    echo pwd, ": Invalid"
    for err in errors:
      echo "  - ", err
```

### แบบฝึกหัดที่ 2: Word Frequency Counter
```nim
import strutils, tables, algorithm, sequtils

proc wordFrequency(text: string): Table[string, int] =
  result = initTable[string, int]()
  for word in text.toLower().splitWhitespace():
    let cleaned = word.strip({'!', '.', ',', '?', ';', ':'})
    if cleaned.len > 0:
      result.inc(cleaned)

let text = "To be or not to be that is the question whether tis nobler in the mind"
let freq = wordFrequency(text)
var sorted = toSeq(freq.pairs)
sorted.sort(proc(a, b: (string, int)): int = b[1] - a[1])

echo "Top 10 words:"
for (word, count) in sorted[0..<min(10, sorted.len)]:
  echo &"  {word}: {count}"
```

## สรุป Part 9

ในบทนี้เราได้เรียนรู้:
- ✅ String literals: single, multi-line, raw
- ✅ strutils: strip, upper, lower, split, join, replace
- ✅ strformat: string interpolation
- ✅ Unicode string handling
- ✅ Regular expressions
- ✅ String builder patterns
- ✅ Common string algorithms
- ✅ Practical: Log parser

---

**Previous**: [Part 08 - Arrays และ Sequences](part08_arrays_sequences.md)
**Next**: [Part 10 - Tables และ Sets](part10_tables_sets.md)
