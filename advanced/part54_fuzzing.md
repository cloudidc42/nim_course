# Part 54: Fuzzing และ Bug Finding ใน Nim

## Fuzzing คืออะไร

```
Fuzzing = การส่ง input สุ่ม / random เข้าไปในโปรแกรม
เพื่อหา bugs, crashes, security vulnerabilities

ประเภท:
- Dumb fuzzing: random bytes
- Smart fuzzing: grammar-based
- Coverage-guided: AFL, libFuzzer (track which code paths were hit)
- Mutational: ปรับปรุง valid input เล็กน้อย

Tools:
- AFL++ (coverage-guided)
- libFuzzer (LLVM)
- Honggfuzz
- Boofuzz (network)
```

## Simple Fuzzer

```nim
# simple_fuzzer.nim
# Mutation-based fuzzer

import random, strutils, os, osproc, strformat, times

type
  FuzzResult = object
    input: seq[byte]
    crash: bool
    exitCode: int
    output: string
    duration: float

proc randomBytes(size: int): seq[byte] =
  result = newSeq[byte](size)
  for i in 0..<size:
    result[i] = rand(255).byte

proc mutate(data: seq[byte]): seq[byte] =
  result = data
  if result.len == 0: return
  
  let mutationType = rand(5)
  case mutationType:
  of 0:  # Bit flip
    let pos = rand(result.len - 1)
    let bit = rand(7)
    result[pos] = result[pos] xor (1.byte shl bit)
  
  of 1:  # Byte replacement
    let pos = rand(result.len - 1)
    result[pos] = rand(255).byte
  
  of 2:  # Insert bytes
    let pos = rand(result.len)
    let insert = randomBytes(rand(3) + 1)
    result = result[0..<pos] & insert & result[pos..^1]
  
  of 3:  # Delete bytes
    if result.len > 1:
      let pos = rand(result.len - 1)
      result.del(pos)
  
  of 4:  # Interesting values
    let pos = rand(result.len - 1)
    let interesting = [0x00.byte, 0x01.byte, 0x7F.byte, 0x80.byte,
                       0xFE.byte, 0xFF.byte]
    result[pos] = interesting[rand(interesting.len - 1)]
  
  of 5:  # Copy bytes within buffer
    if result.len >= 2:
      let src = rand(result.len - 1)
      let dst = rand(result.len - 1)
      result[dst] = result[src]
  
  else: discard

proc fuzzProgram(target: string, seeds: seq[seq[byte]],
                 maxIter: int = 10000): seq[FuzzResult] =
  result = @[]
  randomize()
  var corpus = seeds
  if corpus.len == 0:
    corpus.add(randomBytes(64))
  
  var crashes = 0
  let startTime = epochTime()
  
  for i in 1..maxIter:
    # Pick random seed
    let seed = corpus[rand(corpus.len - 1)]
    let input = mutate(seed)
    
    # Write input to temp file
    let tmpFile = getTempDir() / &"fuzz_input_{i}"
    writeFile(tmpFile, cast[string](input))
    defer: removeFile(tmpFile)
    
    # Run target
    let t0 = epochTime()
    let (output, exitCode) = execCmdEx(&"{target} {tmpFile}")
    let duration = epochTime() - t0
    
    # Check for crash (signal, not just error)
    let crashed = exitCode < 0 or exitCode == 139 or  # SIGSEGV
                  exitCode == 134 or  # SIGABRT  
                  exitCode == 136     # SIGFPE
    
    if crashed:
      inc crashes
      let fuzzResult = FuzzResult(
        input: input,
        crash: true,
        exitCode: exitCode,
        output: output,
        duration: duration
      )
      result.add(fuzzResult)
      
      # Save crash
      let crashFile = &"crash_{crashes}_{i}.bin"
      writeFile(crashFile, cast[string](input))
      echo &"[CRASH #{crashes}] Saved to {crashFile}"
    
    # Add interesting input to corpus (when output is different)
    if rand(100) < 5:
      corpus.add(input)
    
    if i mod 1000 == 0:
      let elapsed = epochTime() - startTime
      let rate = float(i) / elapsed
      echo &"[{i}/{maxIter}] Crashes: {crashes}, Rate: {rate:.0f}/s"

# Demo with a simple target
proc vulnerableParser(data: string) =
  ## This function has intentional bugs for demo
  if data.len > 0 and data[0] == 'A':
    var buf: array[16, char]
    for i in 0..<data.len:  # Buffer overflow!
      buf[i] = data[i]      # No bounds check
  echo "Parsed: ", data.len, " bytes"

echo "Fuzzer ready. Use: fuzzProgram(\"./target\", seeds)"
```

## Grammar-based Fuzzer

```nim
# grammar_fuzzer.nim
# สร้าง input ตามโครงสร้าง

import random, strutils, strformat

type
  GrammarRule = object
    name: string
    alternatives: seq[seq[string]]  # alternatives of token sequences

# JSON grammar
const jsonGrammar = [
  GrammarRule(name: "<value>", alternatives: @[
    @["<object>"],
    @["<array>"],
    @["<string>"],
    @["<number>"],
    @["true"],
    @["false"],
    @["null"]
  ]),
  GrammarRule(name: "<object>", alternatives: @[
    @["{}"],
    @["{", "<pair>", ",", "<pair>", "}"]
  ]),
  GrammarRule(name: "<array>", alternatives: @[
    @["[]"],
    @["[", "<value>", ",", "<value>", "]"]
  ]),
  GrammarRule(name: "<string>", alternatives: @[
    @["\"\""],
    @["\"hello\""],
    @["\"\\u0000\""],  # Unicode edge case
    @["\"", "<char>", "\""],
  ]),
  GrammarRule(name: "<number>", alternatives: @[
    @["0"],
    @["-1"],
    @["2147483647"],   # INT_MAX
    @["-2147483648"],  # INT_MIN
    @["1.7976931348623157e+308"],  # MAX double
    @["0.0"],
  ]),
]

proc findRule(grammar: openArray[GrammarRule], name: string): int =
  for i, r in grammar:
    if r.name == name: return i
  -1

proc expand(grammar: openArray[GrammarRule], symbol: string,
            depth: int = 0): string =
  if depth > 5: return symbol
  
  let ruleIdx = grammar.findRule(symbol)
  if ruleIdx < 0: return symbol
  
  let rule = grammar[ruleIdx]
  let alt = rule.alternatives[rand(rule.alternatives.len - 1)]
  
  result = ""
  for token in alt:
    if token.startsWith("<") and token.endsWith(">"):
      result &= grammar.expand(token, depth + 1)
    else:
      result &= token

# Generate N JSON inputs
randomize()
for i in 1..10:
  let input = jsonGrammar.expand("<value>")
  echo &"Input {i}: {input}"

# Fuzz JSON parser
proc fuzzJsonParser(iterations: int = 1000) =
  import json
  var crashes = 0
  for i in 1..iterations:
    let input = jsonGrammar.expand("<value>")
    try:
      discard parseJson(input)  # This shouldn't crash Nim's json module
    except JsonParsingError: discard  # Expected
    except:
      inc crashes
      echo &"[!] Unexpected exception on input: {input}"
  echo &"Fuzzed {iterations} inputs, {crashes} unexpected crashes"

fuzzJsonParser()
```

## Network Fuzzer

```nim
# network_fuzzer.nim
# Fuzz network protocols

import asyncnet, asyncdispatch, random, strutils, strformat

type
  NetworkFuzzResult = object
    input: string
    response: string
    timeout: bool
    errorMsg: string

proc fuzzTcpService(host: string, port: int, 
                    inputs: seq[string]): Future[seq[NetworkFuzzResult]] {.async.} =
  result = @[]
  
  for input in inputs:
    let sock = newAsyncSocket()
    var res = NetworkFuzzResult(input: input)
    
    try:
      let connected = await withTimeout(sock.connect(host, Port(port)), 3000)
      if not connected:
        res.timeout = true
        result.add(res)
        sock.close()
        continue
      
      await sock.send(input)
      
      let recvFuture = sock.recv(4096)
      let received = await withTimeout(recvFuture, 2000)
      
      if received:
        res.response = await recvFuture
      else:
        res.timeout = true
    except:
      res.errorMsg = getCurrentExceptionMsg()
    finally:
      sock.close()
    
    result.add(res)

proc generateHttpFuzz(): seq[string] =
  result = @[]
  randomize()
  
  # Valid request
  result.add("GET / HTTP/1.1\r\nHost: target\r\n\r\n")
  
  # Oversized header
  result.add("GET / HTTP/1.1\r\nHost: " & "A".repeat(8192) & "\r\n\r\n")
  
  # Bad method
  result.add("INVALID / HTTP/1.1\r\n\r\n")
  
  # Missing version
  result.add("GET /\r\n\r\n")
  
  # Null bytes
  result.add("GET /\x00path HTTP/1.1\r\nHost: target\r\n\r\n")
  
  # Many headers
  var manyHeaders = "GET / HTTP/1.1\r\n"
  for i in 1..100:
    manyHeaders &= &"X-Header-{i}: value\r\n"
  manyHeaders &= "\r\n"
  result.add(manyHeaders)
  
  # Negative Content-Length
  result.add("POST / HTTP/1.1\r\nContent-Length: -1\r\n\r\nbody")

echo "Network Fuzzer ready"
echo "Inputs generated: ", generateHttpFuzz().len
```

## สรุป Part 54

- ␅ Fuzzing concepts
- ␅ Mutation-based fuzzer
- ␅ Grammar-based fuzzer สำหรับ JSON
- ␅ Network protocol fuzzer
- ␅ Crash detection และบันทึก

---
**Next**: [Part 55 - Memory Forensics](part55_forensics.md)
