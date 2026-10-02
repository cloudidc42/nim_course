# Part 89 - Database Internals

## บทนำ

เข้าใจว่า database ทำงานอย่างไรภายใน — B-tree index, Write-Ahead Logging (WAL), MVCC

---

## 1. B-Tree Implementation

```nim
# btree.nim
# B-tree index structure

import std/[strformat, algorithm, sequtils, options]

const
  BTREE_ORDER* = 3  # Max keys per node = 2*ORDER - 1

type
  BTreeKey* = int64
  BTreeValue* = string

  BTreeNode* = ref object
    keys*: seq[BTreeKey]
    values*: seq[BTreeValue]  # For leaf nodes
    children*: seq[BTreeNode]
    isLeaf*: bool
    parent*: BTreeNode

  BTree* = ref object
    root*: BTreeNode
    order*: int
    size*: int

proc newNode*(isLeaf: bool): BTreeNode =
  BTreeNode(isLeaf: isLeaf)

proc newBTree*(order = BTREE_ORDER): BTree =
  BTree(root: newNode(true), order: order)

proc maxKeys*(tree: BTree): int = 2 * tree.order - 1
proc minKeys*(tree: BTree): int = tree.order - 1

proc isFull*(node: BTreeNode, tree: BTree): bool =
  node.keys.len >= tree.maxKeys

proc find*(node: BTreeNode, key: BTreeKey): int =
  ## Binary search for key position
  var lo = 0
  var hi = node.keys.len
  while lo < hi:
    let mid = (lo + hi) div 2
    if node.keys[mid] < key: lo = mid + 1
    else: hi = mid
  lo

proc search*(node: BTreeNode, key: BTreeKey): Option[BTreeValue] =
  let i = node.find(key)
  
  if i < node.keys.len and node.keys[i] == key:
    if node.isLeaf:
      return some(node.values[i])
    # Key found in internal node (for non-leaf, value is in leaf)
  
  if node.isLeaf:
    return none(BTreeValue)
  
  # Recurse into child
  node.children[i].search(key)

proc splitChild*(parent: BTreeNode, childIndex: int, tree: BTree) =
  let child = parent.children[childIndex]
  let mid = tree.order - 1
  
  let newNode = newNode(child.isLeaf)
  newNode.parent = parent
  
  # Copy right half to new node
  newNode.keys = child.keys[mid+1..^1]
  if child.isLeaf:
    newNode.values = child.values[mid+1..^1]
    child.values = child.values[0..mid]
  else:
    newNode.children = child.children[mid+1..^1]
    child.children = child.children[0..mid]
  
  # Promote middle key to parent
  let promotedKey = child.keys[mid]
  let promotedVal = if child.isLeaf: child.values[mid] else: ""
  
  child.keys = child.keys[0..mid-1]
  if child.isLeaf:
    child.values = child.values[0..mid-1]
  
  parent.keys.insert(promotedKey, childIndex)
  if child.isLeaf:
    parent.values.insert(promotedVal, childIndex)
  parent.children.insert(newNode, childIndex + 1)

proc insertNonFull*(node: BTreeNode, key: BTreeKey, value: BTreeValue, tree: BTree) =
  var i = node.keys.len - 1
  
  if node.isLeaf:
    # Find insertion point
    let pos = node.find(key)
    if pos < node.keys.len and node.keys[pos] == key:
      # Update existing
      node.values[pos] = value
      return
    node.keys.insert(key, pos)
    node.values.insert(value, pos)
  else:
    # Find child to descend into
    while i >= 0 and key < node.keys[i]:
      dec i
    inc i
    
    if node.children[i].isFull(tree):
      splitChild(node, i, tree)
      if key > node.keys[i]: inc i
    
    insertNonFull(node.children[i], key, value, tree)

proc insert*(tree: BTree, key: BTreeKey, value: BTreeValue) =
  inc tree.size
  
  if tree.root.isFull(tree):
    let newRoot = newNode(false)
    newRoot.children.add(tree.root)
    tree.root.parent = newRoot
    splitChild(newRoot, 0, tree)
    tree.root = newRoot
  
  insertNonFull(tree.root, key, value, tree)

proc get*(tree: BTree, key: BTreeKey): Option[BTreeValue] =
  tree.root.search(key)

# Range scan
proc rangeQuery*(node: BTreeNode, minKey, maxKey: BTreeKey,
    result: var seq[(BTreeKey, BTreeValue)]) =
  if node.isLeaf:
    for i, k in node.keys:
      if k >= minKey and k <= maxKey:
        result.add((k, node.values[i]))
    return
  
  var i = 0
  while i < node.keys.len and node.keys[i] < minKey:
    inc i
  
  if i < node.children.len:
    rangeQuery(node.children[i], minKey, maxKey, result)
  
  while i < node.keys.len and node.keys[i] <= maxKey:
    if node.isLeaf:
      result.add((node.keys[i], node.values[i]))
    if i + 1 < node.children.len:
      rangeQuery(node.children[i+1], minKey, maxKey, result)
    inc i

proc rangeQuery*(tree: BTree, minKey, maxKey: BTreeKey): seq[(BTreeKey, BTreeValue)] =
  rangeQuery(tree.root, minKey, maxKey, result)

when isMainModule:
  let tree = newBTree()
  
  for i in @[5, 3, 7, 1, 4, 6, 8, 2, 9, 10]:
    tree.insert(i, &"value_{i}")
  
  echo "B-Tree contents:"
  for (k, v) in tree.rangeQuery(1, 10):
    echo &"  {k}: {v}"
  
  echo &"\nGet 5: {tree.get(5)}"
  echo &"Get 11: {tree.get(11)}"
```

---

## 2. Write-Ahead Log (WAL)

```nim
# wal.nim
# Write-Ahead Log สำหรับ crash recovery

import std/[streams, strformat, times, strutils, tables, options, endians]

type
  WalRecordType* = enum
    wrtBegin = 1
    wrtInsert = 2
    wrtUpdate = 3
    wrtDelete = 4
    wrtCommit = 5
    wrtAbort = 6
    wrtCheckpoint = 7

  WalRecord* = object
    lsn*: uint64          # Log Sequence Number
    txId*: uint64          # Transaction ID
    timestamp*: int64
    recordType*: WalRecordType
    tableName*: string
    key*: string
    oldValue*: string
    newValue*: string

  WAL* = ref object
    path*: string
    file*: File
    currentLsn*: uint64
    txCounter*: uint64
    dirty*: Table[uint64, seq[WalRecord]]  # txId -> records (uncommitted)

proc newWAL*(path: string): WAL =
  let f = open(path, fmReadWriteExisting)
  result = WAL(path: path, file: f, dirty: initTable[uint64, seq[WalRecord]]())
  
  # Read current LSN from file
  if f.getFileSize() > 0:
    f.setFilePos(f.getFileSize() - 8)
    discard f.readBuffer(addr result.currentLsn, 8)

proc nextLsn*(wal: WAL): uint64 =
  inc wal.currentLsn
  wal.currentLsn

proc nextTxId*(wal: WAL): uint64 =
  inc wal.txCounter
  wal.txCounter

proc writeRecord*(wal: WAL, rec: var WalRecord) =
  rec.lsn = wal.nextLsn()
  rec.timestamp = getTime().toUnix()
  
  # Write to file (simplified format)
  let payload = &"{rec.lsn}|{rec.txId}|{rec.timestamp}|{rec.recordType.ord}|{rec.tableName}|{rec.key}|{rec.oldValue}|{rec.newValue}\n"
  wal.file.write(payload)
  wal.file.flushFile()

proc beginTx*(wal: WAL): uint64 =
  let txId = wal.nextTxId()
  var rec = WalRecord(txId: txId, recordType: wrtBegin)
  wal.writeRecord(rec)
  wal.dirty[txId] = @[]
  txId

proc logInsert*(wal: WAL, txId: uint64, table, key, value: string) =
  var rec = WalRecord(
    txId: txId,
    recordType: wrtInsert,
    tableName: table,
    key: key,
    newValue: value
  )
  wal.writeRecord(rec)
  if txId in wal.dirty:
    wal.dirty[txId].add(rec)

proc logUpdate*(wal: WAL, txId: uint64, table, key, oldVal, newVal: string) =
  var rec = WalRecord(
    txId: txId,
    recordType: wrtUpdate,
    tableName: table,
    key: key,
    oldValue: oldVal,
    newValue: newVal
  )
  wal.writeRecord(rec)
  if txId in wal.dirty:
    wal.dirty[txId].add(rec)

proc commitTx*(wal: WAL, txId: uint64) =
  var rec = WalRecord(txId: txId, recordType: wrtCommit)
  wal.writeRecord(rec)
  wal.dirty.del(txId)

proc abortTx*(wal: WAL, txId: uint64) =
  var rec = WalRecord(txId: txId, recordType: wrtAbort)
  wal.writeRecord(rec)
  wal.dirty.del(txId)

proc recover*(wal: WAL): Table[string, Table[string, string]] =
  ## Replay WAL to rebuild state
  result = initTable[string, Table[string, string]]()
  
  let f = open(wal.path, fmRead)
  defer: f.close()
  
  var committedTxs: Table[uint64, seq[WalRecord]]
  var pendingTxs: Table[uint64, seq[WalRecord]]
  
  for line in f.lines:
    let parts = line.split('|')
    if parts.len < 8: continue
    
    let rec = WalRecord(
      lsn: parseUInt(parts[0]),
      txId: parseUInt(parts[1]),
      timestamp: parseBiggestInt(parts[2]),
      recordType: WalRecordType(parseInt(parts[3])),
      tableName: parts[4],
      key: parts[5],
      oldValue: parts[6],
      newValue: parts[7]
    )
    
    case rec.recordType
    of wrtBegin:
      pendingTxs[rec.txId] = @[]
    of wrtInsert, wrtUpdate, wrtDelete:
      if rec.txId in pendingTxs:
        pendingTxs[rec.txId].add(rec)
    of wrtCommit:
      if rec.txId in pendingTxs:
        committedTxs[rec.txId] = pendingTxs[rec.txId]
        pendingTxs.del(rec.txId)
    of wrtAbort:
      pendingTxs.del(rec.txId)
    of wrtCheckpoint:
      # Reset — only apply changes after checkpoint
      committedTxs.clear()
      pendingTxs.clear()
  
  # Apply committed transactions
  for txId, records in committedTxs:
    for rec in records:
      if rec.tableName notin result:
        result[rec.tableName] = initTable[string, string]()
      
      case rec.recordType
      of wrtInsert, wrtUpdate:
        result[rec.tableName][rec.key] = rec.newValue
      of wrtDelete:
        result[rec.tableName].del(rec.key)
      else: discard
  
  echo &"Recovery complete: {committedTxs.len} committed, {pendingTxs.len} rolled back"
```

---

## 3. MVCC (Multi-Version Concurrency Control)

```nim
# mvcc.nim
# Simplified MVCC

import std/[tables, strformat, times, sequtils, options]

type
  TxId* = uint64
  Version* = uint64

  MVCCEntry* = object
    key*: string
    value*: string
    createdAt*: TxId   # Created by this transaction
    deletedAt*: TxId   # Deleted by this transaction (0 = not deleted)

  MVCCStore* = ref object
    data*: Table[string, seq[MVCCEntry]]  # key -> versions
    currentTx*: TxId
    activeTxs*: seq[TxId]    # Transactions in progress
    commitLog*: seq[TxId]    # Order of commits

  Transaction* = ref object
    id*: TxId
    store*: MVCCStore
    snapshot*: TxId          # Read snapshot (committed txs at start time)
    writes*: Table[string, MVCCEntry]
    active*: bool

proc newMVCCStore*(): MVCCStore =
  MVCCStore(data: initTable[string, seq[MVCCEntry]]())

proc beginTx*(store: MVCCStore): Transaction =
  inc store.currentTx
  let txId = store.currentTx
  store.activeTxs.add(txId)
  
  # Snapshot = last committed tx
  let snapshot = if store.commitLog.len > 0: store.commitLog[^1] else: 0u64
  
  Transaction(
    id: txId,
    store: store,
    snapshot: snapshot,
    writes: initTable[string, MVCCEntry](),
    active: true
  )

proc isVisible*(entry: MVCCEntry, tx: Transaction): bool =
  ## An entry is visible if:
  ## 1. Created by a committed tx before snapshot
  ## 2. Or created by this tx
  ## AND not deleted before snapshot
  
  let createdCommitted = entry.createdAt == tx.id or
    (entry.createdAt <= tx.snapshot and entry.createdAt in tx.store.commitLog)
  
  if not createdCommitted: return false
  
  if entry.deletedAt == 0: return true  # Not deleted
  
  # Deleted by a committed tx after our snapshot?
  if entry.deletedAt == tx.id: return false  # Deleted by us
  
  let deleteCommitted = entry.deletedAt <= tx.snapshot and
    entry.deletedAt in tx.store.commitLog
  
  not deleteCommitted

proc read*(tx: Transaction, key: string): Option[string] =
  # Check local writes first
  if key in tx.writes:
    let entry = tx.writes[key]
    if entry.deletedAt != 0: return none(string)
    return some(entry.value)
  
  if key notin tx.store.data:
    return none(string)
  
  # Find most recent visible version
  for entry in tx.store.data[key].reversed():
    if entry.isVisible(tx):
      return some(entry.value)
  
  none(string)

proc write*(tx: Transaction, key, value: string) =
  tx.writes[key] = MVCCEntry(
    key: key,
    value: value,
    createdAt: tx.id,
    deletedAt: 0
  )

proc delete*(tx: Transaction, key: string) =
  tx.writes[key] = MVCCEntry(
    key: key,
    value: "",
    createdAt: 0,
    deletedAt: tx.id
  )

proc commit*(tx: Transaction): bool =
  if not tx.active: return false
  tx.active = false
  
  # Write-write conflict detection
  for key in tx.writes.keys:
    if key in tx.store.data:
      for entry in tx.store.data[key].reversed():
        if entry.createdAt > tx.snapshot and entry.createdAt notin tx.store.commitLog:
          continue
        if entry.createdAt > tx.snapshot:
          echo &"Write-write conflict on key: {key}"
          return false
        break
  
  # Apply writes
  for key, entry in tx.writes:
    if key notin tx.store.data:
      tx.store.data[key] = @[]
    tx.store.data[key].add(entry)
  
  # Mark committed
  tx.store.activeTxs = tx.store.activeTxs.filterIt(it != tx.id)
  tx.store.commitLog.add(tx.id)
  true

proc rollback*(tx: Transaction) =
  tx.active = false
  tx.store.activeTxs = tx.store.activeTxs.filterIt(it != tx.id)

proc vacuum*(store: MVCCStore) =
  ## Remove old versions no longer needed
  let minActiveTx = if store.activeTxs.len > 0:
    store.activeTxs.min()
  else:
    store.currentTx
  
  for key in store.data.keys.toSeq():
    store.data[key] = store.data[key].filterIt(
      it.createdAt >= minActiveTx or
      (it.deletedAt == 0 or it.deletedAt >= minActiveTx)
    )

when isMainModule:
  let store = newMVCCStore()
  
  # Tx1: Insert
  let tx1 = store.beginTx()
  tx1.write("user:1", "Alice")
  tx1.write("user:2", "Bob")
  discard tx1.commit()
  
  # Tx2: Read (should see tx1's writes)
  let tx2 = store.beginTx()
  echo &"user:1 = {tx2.read(\"user:1\")}"  # Some("Alice")
  
  # Tx3: Update concurrently with tx2
  let tx3 = store.beginTx()
  tx3.write("user:1", "Alice Updated")
  discard tx3.commit()
  
  # tx2 still sees old version (snapshot isolation)
  echo &"user:1 (tx2 snapshot) = {tx2.read(\"user:1\")}"  # Some("Alice")
  
  let tx4 = store.beginTx()
  echo &"user:1 (after tx3) = {tx4.read(\"user:1\")}"  # Some("Alice Updated")
  
  store.vacuum()
```

---

## สรุป

| Component | Role |
|-----------|------|
| B-Tree | Index structure — O(log n) operations |
| WAL | Crash recovery — REDO/UNDO |
| MVCC | Concurrent reads without locks |

---

**Next**: [Part 90 - Capstone: Full-Stack Production Application](../advanced/part90_capstone.md)
