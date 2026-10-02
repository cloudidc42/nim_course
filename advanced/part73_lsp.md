# Part 73 - Language Server Protocol & IDE Integration

## บทนำ

Language Server Protocol (LSP) คือ protocol มาตรฐานที่ช่วยให้ language tools
ทำงานกับ editor ทุกตัวได้ (VS Code, Vim, Emacs, Sublime, etc.)
เรียนรู้การสร้าง LSP server ด้วย Nim และ integration กับ nimsuggest

---

## 1. LSP Protocol Basics

```nim
# lsp_types.nim
# LSP JSON-RPC 2.0 types

import std/[json, options, strformat, tables]

type
  # JSON-RPC 2.0 base types
  JsonRPCId* = object
    case isInt*: bool
    of true:  intVal*: int
    of false: strVal*: string

  RequestMessage* = object
    jsonrpc*: string     # "2.0"
    id*: JsonRPCId
    `method`*: string
    params*: JsonNode

  ResponseError* = object
    code*: int
    message*: string
    data*: JsonNode

  ResponseMessage* = object
    jsonrpc*: string
    id*: JsonRPCId
    result*: JsonNode
    error*: Option[ResponseError]

  NotificationMessage* = object
    jsonrpc*: string
    `method`*: string
    params*: JsonNode

  # LSP Position and Range
  Position* = object
    line*: int        # 0-indexed line number
    character*: int   # 0-indexed character offset

  Range* = object
    start*: Position
    `end`*: Position

  Location* = object
    uri*: string
    `range`*: Range

  TextDocumentIdentifier* = object
    uri*: string

  TextDocumentItem* = object
    uri*: string
    languageId*: string
    version*: int
    text*: string

  TextDocumentPositionParams* = object
    textDocument*: TextDocumentIdentifier
    position*: Position

# LSP error codes
const
  ParseError*           = -32700
  InvalidRequest*       = -32600
  MethodNotFound*       = -32601
  InvalidParams*        = -32602
  InternalError*        = -32603
  ServerNotInitialized* = -32002
  UnknownErrorCode*     = -32001

# LSP Completion types
type
  CompletionItemKind* = enum
    cikText         = 1
    cikMethod       = 2
    cikFunction     = 3
    cikConstructor  = 4
    cikField        = 5
    cikVariable     = 6
    cikClass        = 7
    cikInterface    = 8
    cikModule       = 9
    cikProperty     = 10
    cikUnit         = 11
    cikValue        = 12
    cikEnum         = 13
    cikKeyword      = 14
    cikSnippet      = 15
    cikColor        = 16
    cikFile         = 17
    cikReference    = 18

  CompletionItem* = object
    label*: string
    kind*: CompletionItemKind
    detail*: string
    documentation*: string
    insertText*: string
    filterText*: string

  DiagnosticSeverity* = enum
    dsSevError       = 1
    dsSevWarning     = 2
    dsSevInformation = 3
    dsSevHint        = 4

  Diagnostic* = object
    `range`*: Range
    severity*: DiagnosticSeverity
    code*: string
    source*: string
    message*: string

proc toJson*(pos: Position): JsonNode =
  %*{"line": pos.line, "character": pos.character}

proc toJson*(r: Range): JsonNode =
  %*{"start": r.start.toJson(), "end": r.`end`.toJson()}

proc toJson*(diag: Diagnostic): JsonNode =
  %*{
    "range": diag.`range`.toJson(),
    "severity": diag.severity.int,
    "message": diag.message,
    "source": diag.source
  }
```

---

## 2. LSP Server Implementation

```nim
# lsp_server.nim
# Minimal LSP server สำหรับ Nim language

import std/[asyncdispatch, asyncio, json, options, strformat, tables]
import std/[strutils, streams, os]
import ./lsp_types

type
  DocumentState* = object
    uri*: string
    version*: int
    content*: string
    lines*: seq[string]

  LSPServer* = ref object
    documents*: Table[string, DocumentState]
    initialized*: bool
    rootUri*: string
    capabilities*: JsonNode
    running*: bool

proc newLSPServer*(): LSPServer =
  LSPServer(
    documents: initTable[string, DocumentState](),
    running: true
  )

proc readHeader*(stream: AsyncFile): Future[int] {.async.} =
  ## Read LSP Content-Length header
  var contentLength = -1
  
  while true:
    let line = (await stream.readLine()).strip()
    if line.len == 0:
      break
    if line.startsWith("Content-Length: "):
      contentLength = parseInt(line["Content-Length: ".len..^1])
  
  contentLength

proc readMessage*(stream: AsyncFile): Future[JsonNode] {.async.} =
  ## Read one LSP message from stdin
  let contentLength = await stream.readHeader()
  if contentLength < 0:
    return newJNull()
  
  let body = await stream.read(contentLength)
  parseJson(body)

proc sendMessage*(msg: JsonNode) =
  ## Send LSP message to stdout
  let body = $msg
  let header = &"Content-Length: {body.len}\r\n\r\n"
  stdout.write(header & body)
  stdout.flushFile()

proc sendResponse*(id: JsonRPCId, result: JsonNode) =
  let idNode = if id.isInt: %id.intVal else: %id.strVal
  sendMessage(%*{
    "jsonrpc": "2.0",
    "id": idNode,
    "result": result
  })

proc sendError*(id: JsonRPCId, code: int, message: string) =
  let idNode = if id.isInt: %id.intVal else: %id.strVal
  sendMessage(%*{
    "jsonrpc": "2.0",
    "id": idNode,
    "error": {"code": code, "message": message}
  })

proc sendNotification*(methd: string, params: JsonNode) =
  sendMessage(%*{
    "jsonrpc": "2.0",
    "method": methd,
    "params": params
  })

proc publishDiagnostics*(server: LSPServer, uri: string,
    diagnostics: seq[Diagnostic]) =
  let diagArray = newJArray()
  for d in diagnostics:
    diagArray.add(d.toJson())
  
  sendNotification("textDocument/publishDiagnostics", %*{
    "uri": uri,
    "diagnostics": diagArray
  })

# Request handlers
proc handleInitialize*(server: LSPServer, params: JsonNode): JsonNode =
  server.initialized = true
  
  if params.hasKey("rootUri"):
    server.rootUri = params["rootUri"].getStr()
  
  %*{
    "capabilities": {
      "textDocumentSync": {
        "openClose": true,
        "change": 1  # Full document sync
      },
      "completionProvider": {
        "triggerCharacters": [".", "("]
      },
      "hoverProvider": true,
      "definitionProvider": true,
      "referencesProvider": true,
      "documentSymbolProvider": true,
      "workspaceSymbolProvider": true,
      "codeActionProvider": false,
      "renameProvider": false,
      "documentFormattingProvider": true
    },
    "serverInfo": {
      "name": "nim-lsp",
      "version": "0.1.0"
    }
  }

proc handleDidOpen*(server: LSPServer, params: JsonNode) =
  let textDoc = params["textDocument"]
  let uri = textDoc["uri"].getStr()
  let content = textDoc["text"].getStr()
  
  server.documents[uri] = DocumentState(
    uri: uri,
    version: textDoc["version"].getInt(),
    content: content,
    lines: content.splitLines()
  )
  
  # Run diagnostics asynchronously
  # asyncCheck server.runDiagnostics(uri)
  echo &"[LSP] Opened: {uri}"

proc handleDidChange*(server: LSPServer, params: JsonNode) =
  let uri = params["textDocument"]["uri"].getStr()
  let version = params["textDocument"]["version"].getInt()
  
  # Full document sync (incremental sync would be more efficient)
  let changes = params["contentChanges"]
  if changes.len > 0:
    let newContent = changes[0]["text"].getStr()
    server.documents[uri] = DocumentState(
      uri: uri,
      version: version,
      content: newContent,
      lines: newContent.splitLines()
    )

proc handleCompletion*(server: LSPServer, params: JsonNode): JsonNode =
  let uri = params["textDocument"]["uri"].getStr()
  let line = params["position"]["line"].getInt()
  let character = params["position"]["character"].getInt()
  
  # Simple keyword completion
  let nimKeywords = [
    "proc", "func", "method", "template", "macro",
    "type", "var", "let", "const", "import", "from", "export",
    "if", "elif", "else", "when", "case", "of",
    "for", "while", "do", "break", "continue", "return",
    "try", "except", "finally", "raise", "defer",
    "object", "ref", "ptr", "enum", "tuple", "seq",
    "string", "int", "float", "bool", "void",
    "true", "false", "nil", "discard", "result",
    "echo", "inc", "dec", "new", "alloc", "dealloc"
  ]
  
  let items = newJArray()
  for kw in nimKeywords:
    items.add(%*{
      "label": kw,
      "kind": cikKeyword.int,
      "detail": "Nim keyword"
    })
  
  %*{"isIncomplete": false, "items": items}

proc handleHover*(server: LSPServer, params: JsonNode): JsonNode =
  let uri = params["textDocument"]["uri"].getStr()
  let line = params["position"]["line"].getInt()
  let char = params["position"]["character"].getInt()
  
  if not server.documents.hasKey(uri):
    return newJNull()
  
  let doc = server.documents[uri]
  if line >= doc.lines.len:
    return newJNull()
  
  let lineText = doc.lines[line]
  
  # Find word at position
  var start = char
  var stop = char
  while start > 0 and lineText[start - 1].isAlphaAscii():
    dec start
  while stop < lineText.len and lineText[stop].isAlphaAscii():
    inc stop
  
  let word = lineText[start..<stop]
  if word.len == 0:
    return newJNull()
  
  %*{
    "contents": {
      "kind": "markdown",
      "value": &"**{word}**\n\nNim identifier"
    }
  }

proc handleFormatting*(server: LSPServer, params: JsonNode): JsonNode =
  ## Document formatting — call nimpretty
  let uri = params["textDocument"]["uri"].getStr()
  
  if not server.documents.hasKey(uri):
    return newJNull()
  
  let doc = server.documents[uri]
  let tmpFile = getTempDir() / "nim_format_tmp.nim"
  writeFile(tmpFile, doc.content)
  
  let exitCode = execShellCmd(&"nimpretty --indent:2 {tmpFile}")
  if exitCode != 0:
    return newJArray()
  
  let formatted = readFile(tmpFile)
  removeFile(tmpFile)
  
  if formatted == doc.content:
    return newJArray()
  
  # Return text edit replacing entire document
  let lineCount = doc.lines.len
  let lastLine = doc.lines[^1]
  
  %*[{
    "range": {
      "start": {"line": 0, "character": 0},
      "end": {"line": lineCount - 1, "character": lastLine.len}
    },
    "newText": formatted
  }]

proc dispatch*(server: LSPServer, msg: JsonNode): Future[void] {.async.} =
  let methd = msg.getOrDefault("method").getStr()
  let params = msg.getOrDefault("params")
  
  # Parse ID
  var id = JsonRPCId(isInt: true, intVal: 0)
  if msg.hasKey("id"):
    let idNode = msg["id"]
    if idNode.kind == JInt:
      id = JsonRPCId(isInt: true, intVal: idNode.getInt())
    else:
      id = JsonRPCId(isInt: false, strVal: idNode.getStr())
  
  case methd
  of "initialize":
    sendResponse(id, server.handleInitialize(params))
  of "initialized":
    discard  # Notification, no response
  of "shutdown":
    server.running = false
    sendResponse(id, newJNull())
  of "exit":
    quit(0)
  of "textDocument/didOpen":
    server.handleDidOpen(params)
  of "textDocument/didChange":
    server.handleDidChange(params)
  of "textDocument/completion":
    sendResponse(id, server.handleCompletion(params))
  of "textDocument/hover":
    sendResponse(id, server.handleHover(params))
  of "textDocument/formatting":
    sendResponse(id, server.handleFormatting(params))
  else:
    if msg.hasKey("id"):
      sendError(id, MethodNotFound, &"Method not found: {methd}")
```

---

## 3. Nimsuggest Integration

```nim
# nimsuggest_client.nim
# nimsuggest คือ tool ของ Nim ที่ให้ completion, definition, etc.

import std/[asyncdispatch, asyncnet, asyncio, json, strformat, strutils]
import std/[options, tables, os]

type
  SuggestKind* = enum
    skDef      # Definition
    skDup      # Duplicate
    skField    # Object field
    skEnumField = "skEnumField"
    skForeign  # Foreign module
    skLabel    # Label
    skLet      # Let binding
    skMacro    # Macro
    skMethod   # Method
    skParam    # Parameter
    skProc     # Procedure
    skResult   # Result variable
    skTemplate # Template
    skType     # Type
    skVar      # Variable

  SuggestResult* = object
    kind*: string
    symbolKind*: string
    qualifiedName*: string
    typeSig*: string
    filepath*: string
    line*: int
    col*: int
    doc*: string

  NimSuggest* = ref object
    process*: Process
    sock*: AsyncSocket
    port*: int
    projectFile*: string
    pending*: Table[int, Future[seq[SuggestResult]]]
    nextId*: int

proc findFreePort*(): int =
  let s = newSocket()
  s.bindAddr(Port(0))
  let (_, port) = s.getLocalAddr()
  s.close()
  int(port)

proc startNimsuggest*(projectFile: string): Future[NimSuggest] {.async.} =
  let port = findFreePort()
  
  let process = startProcess(
    "nimsuggest",
    args = [&"--port:{port}", projectFile],
    options = {poUsePath}
  )
  
  # Wait for nimsuggest to start
  await sleepAsync(1000)
  
  let sock = newAsyncSocket()
  await sock.connect("127.0.0.1", Port(port))
  
  result = NimSuggest(
    process: process,
    sock: sock,
    port: port,
    projectFile: projectFile
  )

proc query*(ns: NimSuggest, command: string, file: string,
    line, col: int): Future[seq[SuggestResult]] {.async.} =
  ## Send query to nimsuggest
  ## Commands: sug (suggest), def (definition), use (usages), 
  ##           chk (check), highlight, outline, known
  let query = &"{command} {file}:{line}:{col}\n"
  await ns.sock.send(query)
  
  var results: seq[SuggestResult]
  
  while true:
    let response = await ns.sock.recvLine()
    if response.len == 0:
      break
    
    let parts = response.split('\t')
    if parts.len >= 8:
      results.add(SuggestResult(
        kind: parts[0],
        symbolKind: parts[1],
        qualifiedName: parts[2],
        typeSig: parts[3],
        filepath: parts[4],
        line: parseInt(parts[5]),
        col: parseInt(parts[6]),
        doc: parts[7].unescape()
      ))
  
  results

proc suggest*(ns: NimSuggest, file: string, line, col: int): Future[seq[SuggestResult]] {.async.} =
  await ns.query("sug", file, line, col)

proc getDefinition*(ns: NimSuggest, file: string, line, col: int): Future[seq[SuggestResult]] {.async.} =
  await ns.query("def", file, line, col)

proc getUsages*(ns: NimSuggest, file: string, line, col: int): Future[seq[SuggestResult]] {.async.} =
  await ns.query("use", file, line, col)

proc check*(ns: NimSuggest, file: string): Future[seq[SuggestResult]] {.async.} =
  await ns.query("chk", file, -1, -1)

proc stop*(ns: NimSuggest) =
  ns.sock.close()
  ns.process.terminate()
```

---

## 4. VS Code Extension Config

```json
{
  "name": "nim-lsp-client",
  "displayName": "Nim Language Support",
  "description": "Nim language support via nim-lsp",
  "version": "0.1.0",
  "engines": {"vscode": "^1.75.0"},
  "categories": ["Programming Languages"],
  "activationEvents": ["onLanguage:nim"],
  "main": "./out/extension.js",
  "contributes": {
    "languages": [{
      "id": "nim",
      "aliases": ["Nim", "nim"],
      "extensions": [".nim", ".nims", ".nimble"],
      "configuration": "./language-configuration.json"
    }],
    "configuration": {
      "title": "Nim",
      "properties": {
        "nim.lspPath": {
          "type": "string",
          "default": "nim-lsp",
          "description": "Path to nim-lsp server"
        },
        "nim.nimsuggestPath": {
          "type": "string",
          "default": "nimsuggest",
          "description": "Path to nimsuggest"
        }
      }
    }
  }
}
```

```typescript
// extension.ts (TypeScript VS Code extension)
import * as vscode from 'vscode';
import { LanguageClient, LanguageClientOptions, ServerOptions, TransportKind } from 'vscode-languageclient/node';

let client: LanguageClient;

export function activate(context: vscode.ExtensionContext) {
  const config = vscode.workspace.getConfiguration('nim');
  const serverPath = config.get<string>('lspPath', 'nim-lsp');
  
  const serverOptions: ServerOptions = {
    run: { command: serverPath, transport: TransportKind.stdio },
    debug: { command: serverPath, args: ['--debug'], transport: TransportKind.stdio }
  };
  
  const clientOptions: LanguageClientOptions = {
    documentSelector: [{ scheme: 'file', language: 'nim' }],
    synchronize: {
      fileEvents: vscode.workspace.createFileSystemWatcher('**/*.nim')
    }
  };
  
  client = new LanguageClient('nim-lsp', 'Nim Language Server', serverOptions, clientOptions);
  client.start();
}

export function deactivate(): Thenable<void> | undefined {
  return client?.stop();
}
```

---

## 5. เขียน LSP Server ให้ใช้งานได้จริง

```nim
# run_lsp.nim
# Entry point สำหรับ nim-lsp server

import std/[asyncdispatch, asyncio, json, strformat, os, logging]
import ./lsp_server
import ./nimsuggest_client

when isMainModule:
  addHandler(newFileLogger("nim-lsp.log", fmtStr = "$datetime [$levelname] "))
  
  let server = newLSPServer()
  
  proc main() {.async.} =
    info "nim-lsp starting"
    
    let stdin = openAsync("/dev/stdin", fmRead)
    
    while server.running:
      try:
        let msg = await stdin.readMessage()
        if msg.kind == JNull:
          break
        
        await server.dispatch(msg)
      except CatchableError as e:
        error &"Error handling message: {e.msg}"
  
  waitFor main()
```

---

## สรุป

| Component | Description |
|-----------|-------------|
| LSP Protocol | JSON-RPC 2.0 over stdio/TCP |
| nimsuggest | Nim's built-in analysis tool |
| Capabilities | textDocument/*, workspace/* |
| VS Code client | vscode-languageclient npm package |
| Diagnostics | publishDiagnostics notification |
| Completion | CompletionItem array |

---

**Next**: [Part 74 - Blockchain & Cryptography Fundamentals](../advanced/part74_crypto.md)
