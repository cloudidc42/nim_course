# Part 81 - CLI Tools & TUI Development

## บทนำ

Nim เหมาะมากสำหรับสร้าง CLI tools เพราะ:
- Compile เป็น binary ขนาดเล็ก
- ไม่ต้องการ runtime
- Cross-platform

---

## 1. Argument Parsing

```nim
# cli_args.nim
# Argument parsing ครบครัน

import std/[os, strutils, strformat, tables, sequtils, options]

type
  FlagKind* = enum
    fkBool, fkString, fkInt, fkFloat, fkStringList

  Flag* = object
    long*: string        # --flag
    short*: char         # -f
    help*: string
    required*: bool
    case kind*: FlagKind
    of fkBool: boolDefault*: bool
    of fkString: strDefault*: string
    of fkInt: intDefault*: int
    of fkFloat: floatDefault*: float
    of fkStringList: listDefault*: seq[string]

  ParsedArgs* = object
    flags*: Table[string, string]
    args*: seq[string]    # Positional args
    command*: string      # Subcommand

  CLI* = ref object
    name*: string
    version*: string
    description*: string
    flags*: seq[Flag]
    subcommands*: Table[string, CLI]
    action*: proc(args: ParsedArgs)

proc newCLI*(name, version, description: string): CLI =
  CLI(
    name: name,
    version: version,
    description: description,
    flags: @[],
    subcommands: initTable[string, CLI]()
  )

proc flag*(cli: CLI, long: string, short = '\0', help = "",
    required = false, default = ""): CLI =
  cli.flags.add(Flag(kind: fkString, long: long, short: short,
    help: help, required: required, strDefault: default))
  cli

proc boolFlag*(cli: CLI, long: string, short = '\0', help = ""): CLI =
  cli.flags.add(Flag(kind: fkBool, long: long, short: short,
    help: help, boolDefault: false))
  cli

proc intFlag*(cli: CLI, long: string, short = '\0', help = "",
    default = 0): CLI =
  cli.flags.add(Flag(kind: fkInt, long: long, short: short,
    help: help, intDefault: default))
  cli

proc subcommand*(cli: CLI, sub: CLI): CLI =
  cli.subcommands[sub.name] = sub
  cli

proc on*(cli: CLI, action: proc(args: ParsedArgs)): CLI =
  cli.action = action
  cli

proc printHelp*(cli: CLI) =
  echo &"{cli.name} {cli.version}"
  echo cli.description
  echo ""
  echo "Usage:"
  echo &"  {cli.name} [flags] [args]"
  
  if cli.subcommands.len > 0:
    echo ""
    echo "Commands:"
    for name, sub in cli.subcommands:
      echo &"  {name:<15} {sub.description}"
  
  if cli.flags.len > 0:
    echo ""
    echo "Flags:"
    for f in cli.flags:
      var flagStr = &"  --{f.long}"
      if f.short != '\0': flagStr &= &", -{f.short}"
      echo &"{flagStr:<25} {f.help}"
  
  echo ""
  echo &"  --help, -h           Show this help"
  echo &"  --version, -v        Show version"

proc parse*(cli: CLI, argv: seq[string] = commandLineParams()): ParsedArgs =
  result.flags = initTable[string, string]()
  
  # Set defaults
  for f in cli.flags:
    case f.kind
    of fkBool: result.flags[f.long] = "false"
    of fkString: result.flags[f.long] = f.strDefault
    of fkInt: result.flags[f.long] = $f.intDefault
    of fkFloat: result.flags[f.long] = $f.floatDefault
    of fkStringList: result.flags[f.long] = ""
  
  var i = 0
  while i < argv.len:
    let arg = argv[i]
    
    if arg == "--help" or arg == "-h":
      cli.printHelp()
      quit(0)
    
    if arg == "--version" or arg == "-v":
      echo &"{cli.name} {cli.version}"
      quit(0)
    
    if arg.startsWith("--"):
      let key = arg[2..^1]
      var found = false
      for f in cli.flags:
        if f.long == key:
          found = true
          if f.kind == fkBool:
            result.flags[key] = "true"
          else:
            inc i
            if i < argv.len:
              result.flags[key] = argv[i]
          break
      if not found:
        echo &"Unknown flag: {arg}"
        quit(1)
    
    elif arg.startsWith("-") and arg.len == 2:
      let short = arg[1]
      var found = false
      for f in cli.flags:
        if f.short == short:
          found = true
          if f.kind == fkBool:
            result.flags[f.long] = "true"
          else:
            inc i
            if i < argv.len:
              result.flags[f.long] = argv[i]
          break
      if not found:
        echo &"Unknown flag: {arg}"
        quit(1)
    
    else:
      # Check subcommand
      if result.command.len == 0 and arg in cli.subcommands:
        result.command = arg
        # Parse remaining with subcommand
        let sub = cli.subcommands[arg]
        let subResult = sub.parse(argv[i+1..^1])
        if not sub.action.isNil:
          sub.action(subResult)
        return
      else:
        result.args.add(arg)
    
    inc i
  
  # Check required flags
  for f in cli.flags:
    if f.required and result.flags.getOrDefault(f.long, "") == "":
      echo &"Required flag missing: --{f.long}"
      quit(1)
  
  if not cli.action.isNil:
    cli.action(result)

# Convenience getters
proc getString*(args: ParsedArgs, key: string, default = ""): string =
  args.flags.getOrDefault(key, default)

proc getInt*(args: ParsedArgs, key: string, default = 0): int =
  let s = args.flags.getOrDefault(key, "")
  if s.len == 0: return default
  try: parseInt(s) except ValueError: default

proc getBool*(args: ParsedArgs, key: string): bool =
  args.flags.getOrDefault(key, "false") == "true"

# Example tool
when isMainModule:
  let cli = newCLI("mytool", "1.0.0", "An example CLI tool")
    .flag("output", 'o', "Output file", required = false, default = "stdout")
    .flag("format", 'f', "Output format (json|text)", default = "text")
    .intFlag("count", 'n', "Number of items", default = 10)
    .boolFlag("verbose", 'v', "Verbose output")
    .on(proc(args: ParsedArgs) =
      echo &"Output: {args.getString(\"output\")}"
      echo &"Format: {args.getString(\"format\")}"
      echo &"Count: {args.getInt(\"count\")}"
      echo &"Verbose: {args.getBool(\"verbose\")}"
      echo &"Args: {args.args}"
    )
  
  cli.parse()
```

---

## 2. Terminal Colors & Formatting

```nim
# terminal_ui.nim
# ANSI colors, progress bars, tables

import std/[strformat, strutils, terminal, times, math]

# ANSI color codes
type
  Color* = enum
    cReset = "\e[0m"
    cBold = "\e[1m"
    cDim = "\e[2m"
    cItalic = "\e[3m"
    cUnderline = "\e[4m"
    cBlack = "\e[30m"
    cRed = "\e[31m"
    cGreen = "\e[32m"
    cYellow = "\e[33m"
    cBlue = "\e[34m"
    cMagenta = "\e[35m"
    cCyan = "\e[36m"
    cWhite = "\e[37m"
    cBgRed = "\e[41m"
    cBgGreen = "\e[42m"
    cBgYellow = "\e[43m"
    cBgBlue = "\e[44m"

proc colored*(text: string, colors: varargs[Color]): string =
  if not isatty(stdout): return text
  var codes = ""
  for c in colors: codes &= $c
  codes & text & $cReset

proc success*(msg: string) = echo colored("✓ " & msg, cGreen)
proc error*(msg: string) = echo colored("✗ " & msg, cRed)
proc warn*(msg: string) = echo colored("⚠ " & msg, cYellow)
proc info*(msg: string) = echo colored("ℹ " & msg, cCyan)

# Progress bar
type
  ProgressBar* = ref object
    total*: int
    current*: int
    width*: int
    label*: string
    startTime*: float

proc newProgressBar*(total: int, label = "", width = 40): ProgressBar =
  ProgressBar(total: total, width: width, label: label,
    startTime: cpuTime())

proc render*(pb: ProgressBar) =
  let pct = if pb.total > 0: pb.current / pb.total else: 0.0
  let filled = int(pct * pb.width.float)
  let bar = "█".repeat(filled) & "░".repeat(pb.width - filled)
  let elapsed = cpuTime() - pb.startTime
  
  var eta = ""
  if pb.current > 0 and pb.current < pb.total:
    let rate = pb.current.float / elapsed
    let remaining = (pb.total - pb.current).float / rate
    eta = &" ETA: {remaining:.1f}s"
  
  stdout.write(&"\r{pb.label} [{bar}] {pb.current}/{pb.total} ({pct*100:.1f}%){eta}   ")
  stdout.flushFile()

proc update*(pb: var ProgressBar, n = 1) =
  pb.current += n
  pb.render()

proc finish*(pb: ProgressBar) =
  pb.current = pb.total
  pb.render()
  echo ""

# Spinner
type
  Spinner* = ref object
    frames*: seq[string]
    current*: int
    label*: string
    running*: bool

proc newSpinner*(label = ""): Spinner =
  Spinner(
    frames: @["⠋","⠙","⠹","⠸","⠼","⠴","⠦","⠧","⠇","⠏"],
    label: label
  )

proc tick*(s: Spinner) =
  stdout.write(&"\r{s.frames[s.current]} {s.label}")
  stdout.flushFile()
  s.current = (s.current + 1) mod s.frames.len

proc done*(s: Spinner, msg = "") =
  stdout.write(&"\r{colored(\"✓\", cGreen)} {if msg.len > 0: msg else: s.label}   \n")
  stdout.flushFile()

# Table
type
  TableColumn* = object
    header*: string
    width*: int
    align*: enum taLeft, taRight, taCenter

  Table* = ref object
    columns*: seq[TableColumn]
    rows*: seq[seq[string]]
    border*: bool

proc newTable*(border = true): Table =
  Table(border: border)

proc addColumn*(t: Table, header: string, width = 0,
    align = taLeft): Table =
  t.columns.add(TableColumn(header: header, width: width, align: align))
  t

proc addRow*(t: Table, values: seq[string]) =
  # Auto-size columns
  for i, v in values:
    if i < t.columns.len:
      t.columns[i].width = max(t.columns[i].width,
        max(v.len, t.columns[i].header.len))
  t.rows.add(values)

proc printTable*(t: Table) =
  if t.columns.len == 0: return
  
  # Header separator
  var sep = "+"
  for col in t.columns:
    sep &= "-".repeat(col.width + 2) & "+"
  
  if t.border: echo sep
  
  # Headers
  var header = "|"
  for col in t.columns:
    let h = col.header.alignLeft(col.width)
    header &= &" {colored(h, cBold, cCyan)} |"
  echo header
  
  if t.border: echo sep
  
  # Rows
  for row in t.rows:
    var line = "|"
    for i, col in t.columns:
      let val = if i < row.len: row[i] else: ""
      let padded = case col.align
        of taLeft: val.alignLeft(col.width)
        of taRight: val.align(col.width)
        of taCenter:
          let pad = (col.width - val.len) div 2
          " ".repeat(pad) & val & " ".repeat(col.width - val.len - pad)
      line &= &" {padded} |"
    echo line
  
  if t.border: echo sep

when isMainModule:
  # Demo
  let t = newTable()
    .addColumn("Name", align = taLeft)
    .addColumn("Version", align = taRight)
    .addColumn("Status", align = taCenter)
  
  t.addRow(@["Nim", "2.0.0", "stable"])
  t.addRow(@["Python", "3.12.0", "stable"])
  t.addRow(@["Rust", "1.73.0", "stable"])
  
  t.printTable()
  
  echo ""
  
  # Progress bar demo
  var pb = newProgressBar(100, "Processing")
  for i in 0..100:
    pb.update()
    sleep(10)
  pb.finish()
  
  success("All done!")
  warn("This is a warning")
  error("This is an error")
```

---

## 3. Interactive TUI (ncurses-style)

```nim
# tui.nim
# Simple TUI framework โดยใช้ ANSI escape codes

import std/[terminal, strformat, strutils, tables, rdstdin]

type
  Key* = enum
    kUp = "UP"
    kDown = "DOWN"
    kLeft = "LEFT"
    kRight = "RIGHT"
    kEnter = "ENTER"
    kEscape = "ESC"
    kBackspace = "BS"
    kTab = "TAB"
    kChar

  KeyEvent* = object
    key*: Key
    ch*: char

  Rect* = object
    x*, y*, w*, h*: int

  Widget* = ref object of RootObj
    rect*: Rect
    focused*: bool

  Screen* = ref object
    width*, height*: int
    buffer*: seq[seq[char]]
    dirty*: bool

proc newScreen*(): Screen =
  result = Screen(
    width: terminalWidth(),
    height: terminalHeight()
  )
  result.buffer = newSeqWith(result.height, newSeq[char](result.width))

proc clear*(s: Screen) =
  for row in s.buffer.mitems:
    for c in row.mitems:
      c = ' '

proc put*(s: Screen, x, y: int, c: char) =
  if y >= 0 and y < s.height and x >= 0 and x < s.width:
    s.buffer[y][x] = c
    s.dirty = true

proc puts*(s: Screen, x, y: int, text: string) =
  for i, c in text:
    s.put(x + i, y, c)

proc drawBox*(s: Screen, r: Rect) =
  # Corners
  s.put(r.x, r.y, '┌')
  s.put(r.x + r.w - 1, r.y, '┐')
  s.put(r.x, r.y + r.h - 1, '└')
  s.put(r.x + r.w - 1, r.y + r.h - 1, '┘')
  
  # Horizontal borders
  for x in r.x+1..<r.x+r.w-1:
    s.put(x, r.y, '─')
    s.put(x, r.y + r.h - 1, '─')
  
  # Vertical borders
  for y in r.y+1..<r.y+r.h-1:
    s.put(r.x, y, '│')
    s.put(r.x + r.w - 1, y, '│')

proc render*(s: Screen) =
  if not s.dirty: return
  
  # Move to top-left and render buffer
  stdout.write("\e[H")  # Home position
  for row in s.buffer:
    stdout.write(row.join(""))
    stdout.write("\n")
  stdout.flushFile()
  s.dirty = false

proc readKey*(): KeyEvent =
  let c = getch()
  case c
  of '\e':
    # ESC sequence
    let c2 = getch()
    if c2 == '[':
      let c3 = getch()
      case c3
      of 'A': KeyEvent(key: kUp)
      of 'B': KeyEvent(key: kDown)
      of 'C': KeyEvent(key: kRight)
      of 'D': KeyEvent(key: kLeft)
      else: KeyEvent(key: kEscape)
    else:
      KeyEvent(key: kEscape)
  of '\r', '\n': KeyEvent(key: kEnter)
  of '\x7f', '\b': KeyEvent(key: kBackspace)
  of '\t': KeyEvent(key: kTab)
  else: KeyEvent(key: kChar, ch: c)

# List widget
type
  ListWidget* = ref object of Widget
    items*: seq[string]
    selected*: int
    scrollOffset*: int

proc newListWidget*(rect: Rect, items: seq[string]): ListWidget =
  ListWidget(rect: rect, items: items, selected: 0)

proc draw*(w: ListWidget, s: Screen) =
  s.drawBox(w.rect)
  
  let visibleHeight = w.rect.h - 2
  let start = w.scrollOffset
  let endIdx = min(start + visibleHeight, w.items.len)
  
  for i in start..<endIdx:
    let y = w.rect.y + 1 + (i - start)
    let x = w.rect.x + 1
    let maxWidth = w.rect.w - 2
    
    var text = if w.items[i].len > maxWidth:
      w.items[i][0..<maxWidth]
    else:
      w.items[i].alignLeft(maxWidth)
    
    if i == w.selected:
      # Highlight selected item
      stdout.write(&"\e[{y+1};{x+1}H\e[7m{text}\e[0m")
    else:
      s.puts(x, y, text)

proc handleKey*(w: ListWidget, key: KeyEvent): bool =
  case key.key
  of kUp:
    if w.selected > 0:
      dec w.selected
      if w.selected < w.scrollOffset:
        dec w.scrollOffset
    true
  of kDown:
    if w.selected < w.items.len - 1:
      inc w.selected
      let visibleHeight = w.rect.h - 2
      if w.selected >= w.scrollOffset + visibleHeight:
        inc w.scrollOffset
    true
  else: false

# Simple dialog
proc confirm*(msg: string): bool =
  stdout.write(&"\n{msg} [y/N]: ")
  stdout.flushFile()
  let answer = readLine(stdin).toLowerAscii().strip()
  answer == "y" or answer == "yes"

proc prompt*(msg: string, default = ""): string =
  stdout.write(&"{msg}")
  if default.len > 0:
    stdout.write(&" [{default}]")
  stdout.write(": ")
  stdout.flushFile()
  let answer = readLine(stdin).strip()
  if answer.len == 0: default else: answer

when isMainModule:
  hideCursor()
  defer: showCursor()
  
  let scr = newScreen()
  let list = newListWidget(
    Rect(x: 2, y: 1, w: 30, h: 10),
    @["Option 1", "Option 2", "Option 3", "Option 4", "Option 5",
      "Option 6", "Option 7", "Option 8"]
  )
  
  eraseScreen()
  
  while true:
    scr.clear()
    scr.puts(2, 0, "Use UP/DOWN to navigate, ENTER to select, Q to quit")
    list.draw(scr)
    scr.render()
    
    let key = readKey()
    if key.key == kChar and key.ch == 'q': break
    if key.key == kEnter:
      echo &"\nSelected: {list.items[list.selected]}"
      break
    discard list.handleKey(key)
  
  eraseScreen()
  showCursor()
```

---

## สรุป

| Feature | ใช้เมื่อ |
|---------|---------|
| Argument parsing | CLI tools ทุกประเภท |
| Colors/formatting | User-friendly output |
| Progress bars | Long-running operations |
| Tables | Structured data display |
| TUI | Interactive terminal apps |

---

**Next**: [Part 82 - WebAssembly with Nim](../advanced/part82_wasm.md)
