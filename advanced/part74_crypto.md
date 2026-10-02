# Part 74 - Blockchain & Cryptography Fundamentals in Nim

## บทนำ

เรียนรู้การ implement cryptographic primitives และ blockchain concepts ด้วย Nim:
hash functions, digital signatures, Merkle trees, และ simple blockchain

---

## 1. Hash Functions

```nim
# hashing.nim
# SHA-256, SHA-3, BLAKE2 implementations

import std/[sha2, strformat, strutils, base64]

# SHA-256 (built into Nim stdlib)
proc sha256Hex*(data: string): string =
  let hash = sha256(data)
  result = ""
  for b in hash:
    result &= b.toHex(2).toLowerAscii()

proc sha256Bytes*(data: seq[byte]): array[32, byte] =
  let s = cast[string](data)
  sha256(s)

# Double SHA-256 (used in Bitcoin)
proc doubleSha256*(data: string): string =
  let first = sha256(data)
  let second = sha256(cast[string](first))
  result = ""
  for b in second:
    result &= b.toHex(2).toLowerAscii()

# SHA-3 / Keccak-256 (used in Ethereum) via C binding
proc keccak256*(data: string): string {.importc, header: "keccak.h".}

# HMAC-SHA256 (for message authentication)
proc hmacSha256*(key, data: string): string =
  const blockSize = 64
  var paddedKey: array[64, byte]
  
  let keyBytes = if key.len > blockSize:
    cast[seq[byte]](sha256(key))
  else:
    cast[seq[byte]](key)
  
  for i in 0..<keyBytes.len:
    paddedKey[i] = keyBytes[i]
  
  var ipad: array[64, byte]
  var opad: array[64, byte]
  
  for i in 0..<blockSize:
    ipad[i] = paddedKey[i] xor 0x36
    opad[i] = paddedKey[i] xor 0x5c
  
  let innerHash = sha256(cast[string](ipad) & data)
  let outerHash = sha256(cast[string](opad) & cast[string](innerHash))
  
  result = ""
  for b in outerHash:
    result &= b.toHex(2).toLowerAscii()

# Merkle Tree
type
  MerkleNode* = ref object
    hash*: string
    left*: MerkleNode
    right*: MerkleNode
    data*: string  # For leaf nodes

proc merkleHash*(left, right: string): string =
  sha256Hex(left & right)

proc buildMerkleTree*(data: seq[string]): MerkleNode =
  if data.len == 0:
    return MerkleNode(hash: sha256Hex(""))
  
  if data.len == 1:
    return MerkleNode(hash: sha256Hex(data[0]), data: data[0])
  
  # Build leaf nodes
  var nodes: seq[MerkleNode]
  for d in data:
    nodes.add(MerkleNode(hash: sha256Hex(d), data: d))
  
  # Build tree bottom-up
  while nodes.len > 1:
    var nextLevel: seq[MerkleNode]
    var i = 0
    while i < nodes.len:
      if i + 1 < nodes.len:
        let parent = MerkleNode(
          hash: merkleHash(nodes[i].hash, nodes[i+1].hash),
          left: nodes[i],
          right: nodes[i+1]
        )
        nextLevel.add(parent)
        inc i, 2
      else:
        # Odd node: duplicate it
        let parent = MerkleNode(
          hash: merkleHash(nodes[i].hash, nodes[i].hash),
          left: nodes[i],
          right: nodes[i]
        )
        nextLevel.add(parent)
        inc i
    nodes = nextLevel
  
  nodes[0]

proc merkleRoot*(tree: MerkleNode): string =
  if tree == nil: "" else: tree.hash

proc getMerkleProof*(tree: MerkleNode, data: string): seq[tuple[hash: string, isRight: bool]] =
  ## Get proof that data is in tree (for verification)
  result = @[]
  
  proc findProof(node: MerkleNode, target: string,
      proof: var seq[tuple[hash: string, isRight: bool]]): bool =
    if node == nil: return false
    if node.data == target: return true
    
    if findProof(node.left, target, proof):
      proof.add((hash: node.right.hash, isRight: true))
      return true
    
    if findProof(node.right, target, proof):
      proof.add((hash: node.left.hash, isRight: false))
      return true
    
    false
  
  discard findProof(tree, data, result)

proc verifyMerkleProof*(data, root: string,
    proof: seq[tuple[hash: string, isRight: bool]]): bool =
  var current = sha256Hex(data)
  
  for step in proof:
    if step.isRight:
      current = merkleHash(current, step.hash)
    else:
      current = merkleHash(step.hash, current)
  
  current == root

when isMainModule:
  let transactions = @[
    "Alice sends 1 BTC to Bob",
    "Bob sends 0.5 BTC to Carol",
    "Carol sends 0.2 BTC to Dave",
    "Dave sends 0.1 BTC to Eve"
  ]
  
  let tree = buildMerkleTree(transactions)
  echo &"Merkle Root: {tree.merkleRoot()}"
  
  let proof = getMerkleProof(tree, transactions[1])
  let valid = verifyMerkleProof(transactions[1], tree.merkleRoot(), proof)
  echo &"Proof valid: {valid}"
```

---

## 2. Digital Signatures (ECDSA)

```nim
# ecdsa.nim
# Elliptic Curve Digital Signature Algorithm
# ใช้ libssl via FFI สำหรับ production

import std/[strformat, base64, strutils]

# For real ECDSA, use nimcrypto or openssl bindings
# This demonstrates the concept

type
  PrivateKey* = array[32, byte]
  PublicKey*  = array[64, byte]  # x,y coordinates (uncompressed)
  Signature*  = tuple[r, s: array[32, byte]]

# Simplified EC operations (concept demo — not cryptographically secure)
# In production: use secp256k1 library (Bitcoin/Ethereum curve)

proc generateKeyPair*(): tuple[priv: PrivateKey, pub: PublicKey] =
  ## Generate EC key pair (demo using random)
  var priv: PrivateKey
  var pub: PublicKey
  
  # In real code: use secure random + EC point multiplication
  for i in 0..<32:
    priv[i] = byte(rand(255))
  
  # pub = priv * G (generator point multiplication)
  # Simplified: just use hash for demo
  let pubHash = sha256Hex(cast[string](priv))
  for i in 0..<32:
    pub[i] = byte(parseHexInt($pubHash[i*2..i*2+1]))
    pub[i+32] = byte(255 - pub[i])  # Demo second half
  
  (priv: priv, pub: pub)

# OpenSSL FFI for real ECDSA
{.passL: "-lssl -lcrypto".}

type
  EC_KEY* = pointer
  EC_GROUP* = pointer
  BIGNUM* = pointer
  ECDSA_SIG* = pointer

proc EC_KEY_new_by_curve_name*(nid: cint): EC_KEY {.importc, header: "openssl/ec.h".}
proc EC_KEY_generate_key*(key: EC_KEY): cint {.importc, header: "openssl/ec.h".}
proc EC_KEY_free*(key: EC_KEY) {.importc, header: "openssl/ec.h".}
proc ECDSA_do_sign*(dgst: ptr byte, dgst_len: cint, key: EC_KEY): ECDSA_SIG {.importc, header: "openssl/ecdsa.h".}
proc ECDSA_do_verify*(dgst: ptr byte, dgst_len: cint, sig: ECDSA_SIG, key: EC_KEY): cint {.importc, header: "openssl/ecdsa.h".}

const NID_secp256k1* = 714  # secp256k1 curve (Bitcoin/Ethereum)

proc signMessage*(message: string, privateKeyHex: string): string =
  ## Sign message with secp256k1 private key (via OpenSSL)
  let key = EC_KEY_new_by_curve_name(NID_secp256k1)
  defer: EC_KEY_free(key)
  
  # In real code: import private key from hex
  # key.importPrivKey(privateKeyHex)
  
  let hash = sha256(message)
  let sig = ECDSA_do_sign(unsafeAddr hash[0], 32, key)
  
  # Serialize signature to DER format
  # Return base64 encoded DER
  "signature_placeholder"  # Placeholder

proc verifySignature*(message, signature, publicKeyHex: string): bool =
  ## Verify ECDSA signature
  let key = EC_KEY_new_by_curve_name(NID_secp256k1)
  defer: EC_KEY_free(key)
  
  # Import public key from hex
  # key.importPubKey(publicKeyHex)
  
  let hash = sha256(message)
  # Parse DER signature
  # Verify
  true  # Placeholder
```

---

## 3. Simple Blockchain

```nim
# blockchain.nim
# Simplified blockchain implementation

import std/[times, json, strformat, strutils, sha2, options]
import std/[sequtils, algorithm, tables]

type
  Transaction* = object
    id*: string
    sender*: string
    recipient*: string
    amount*: float
    timestamp*: int64
    signature*: string

  Block* = object
    index*: int
    timestamp*: int64
    transactions*: seq[Transaction]
    previousHash*: string
    hash*: string
    nonce*: uint64     # Proof of work nonce
    difficulty*: int   # Mining difficulty

  Blockchain* = ref object
    chain*: seq[Block]
    pendingTransactions*: seq[Transaction]
    difficulty*: int
    miningReward*: float
    wallets*: Table[string, float]

proc hashBlock*(b: Block): string =
  let data = $b.index & $b.timestamp & $b.previousHash & $b.nonce &
    b.transactions.mapIt(it.id).join(",")
  let hash = sha256(data)
  result = ""
  for byte in hash:
    result &= byte.toHex(2).toLowerAscii()

proc isValidHash*(hash: string, difficulty: int): bool =
  ## Hash must start with `difficulty` zeros
  hash.startsWith("0".repeat(difficulty))

proc mineBlock*(b: var Block) =
  ## Proof of Work: find nonce that produces hash with required leading zeros
  b.nonce = 0
  b.hash = hashBlock(b)
  
  while not isValidHash(b.hash, b.difficulty):
    inc b.nonce
    b.hash = hashBlock(b)
  
  echo &"Block {b.index} mined! nonce={b.nonce} hash={b.hash[0..15]}..."

proc createGenesisBlock*(): Block =
  var genesis = Block(
    index: 0,
    timestamp: epochTime().int64,
    previousHash: "0".repeat(64),
    difficulty: 4
  )
  genesis.hash = hashBlock(genesis)
  genesis

proc newBlockchain*(difficulty = 4, miningReward = 50.0): Blockchain =
  result = Blockchain(
    difficulty: difficulty,
    miningReward: miningReward,
    wallets: initTable[string, float]()
  )
  result.chain.add(createGenesisBlock())
  # Genesis allocation
  result.wallets["genesis"] = 1_000_000.0

proc latestBlock*(bc: Blockchain): Block =
  bc.chain[^1]

proc addTransaction*(bc: Blockchain, sender, recipient: string, amount: float) =
  let senderBalance = bc.wallets.getOrDefault(sender, 0.0)
  if sender != "genesis" and senderBalance < amount:
    raise newException(ValueError, &"Insufficient balance: {senderBalance} < {amount}")
  
  bc.pendingTransactions.add(Transaction(
    id: sha256Hex(&"{sender}{recipient}{amount}{epochTime()}"),
    sender: sender,
    recipient: recipient,
    amount: amount,
    timestamp: epochTime().int64
  ))

proc minePendingTransactions*(bc: Blockchain, minerAddress: string) =
  ## Create new block with pending transactions and mine it
  bc.pendingTransactions.add(Transaction(
    id: sha256Hex(&"reward-{minerAddress}-{epochTime()}"),
    sender: "genesis",
    recipient: minerAddress,
    amount: bc.miningReward,
    timestamp: epochTime().int64
  ))
  
  var newBlock = Block(
    index: bc.chain.len,
    timestamp: epochTime().int64,
    transactions: bc.pendingTransactions,
    previousHash: bc.latestBlock().hash,
    difficulty: bc.difficulty
  )
  
  let t0 = epochTime()
  newBlock.mineBlock()
  let t1 = epochTime()
  
  echo &"Mining took {t1-t0:.2f}s"
  
  bc.chain.add(newBlock)
  
  # Apply transactions to wallets
  for tx in newBlock.transactions:
    if tx.sender != "genesis":
      bc.wallets[tx.sender] = bc.wallets.getOrDefault(tx.sender, 0.0) - tx.amount
    bc.wallets[tx.recipient] = bc.wallets.getOrDefault(tx.recipient, 0.0) + tx.amount
  
  bc.pendingTransactions.setLen(0)

proc isChainValid*(bc: Blockchain): bool =
  for i in 1..<bc.chain.len:
    let current = bc.chain[i]
    let previous = bc.chain[i - 1]
    
    # Check hash
    if current.hash != hashBlock(current):
      echo &"Block {i}: invalid hash"
      return false
    
    # Check link
    if current.previousHash != previous.hash:
      echo &"Block {i}: broken chain"
      return false
    
    # Check proof of work
    if not isValidHash(current.hash, current.difficulty):
      echo &"Block {i}: invalid proof of work"
      return false
  
  true

proc getBalance*(bc: Blockchain, address: string): float =
  bc.wallets.getOrDefault(address, 0.0)

when isMainModule:
  let bc = newBlockchain(difficulty = 3)  # Lower difficulty for demo
  
  bc.addTransaction("genesis", "Alice", 100.0)
  bc.addTransaction("genesis", "Bob", 50.0)
  bc.minePendingTransactions("Miner1")
  
  echo &"Alice: {bc.getBalance('Alice')}"
  echo &"Bob: {bc.getBalance('Bob')}"
  echo &"Miner1: {bc.getBalance('Miner1')}"
  
  bc.addTransaction("Alice", "Bob", 30.0)
  bc.minePendingTransactions("Miner1")
  
  echo &"\nAfter transfer:"
  echo &"Alice: {bc.getBalance('Alice')}"
  echo &"Bob: {bc.getBalance('Bob')}"
  
  echo &"\nChain valid: {bc.isChainValid()}"
  echo &"Chain length: {bc.chain.len}"
```

---

## 4. Smart Contract Simulator

```nim
# smart_contract.nim
# Simple EVM-like virtual machine สำหรับ smart contracts

import std/[tables, strformat, json, options]

type
  OpCode* = enum
    opPush   # Push value onto stack
    opPop    # Pop from stack
    opAdd    # Add top 2 values
    opSub    # Subtract
    opMul    # Multiply
    opDiv    # Divide
    opEq     # Equal comparison
    opLt     # Less than
    opGt     # Greater than
    opJump   # Unconditional jump
    opJumpIf # Conditional jump
    opStore  # Store to state
    opLoad   # Load from state
    opReturn # Return value
    opRevert # Revert execution
    opEmit   # Emit event
    opCall   # Call another contract

  Instruction* = object
    opcode*: OpCode
    value*: Option[int64]  # For PUSH

  ContractEvent* = object
    name*: string
    data*: Table[string, int64]

  ExecutionContext* = object
    stack*: seq[int64]
    memory*: seq[byte]
    storage*: Table[string, int64]
    events*: seq[ContractEvent]
    pc*: int       # Program counter
    gas*: int64    # Gas remaining
    caller*: string
    value*: int64  # ETH value sent

  Contract* = ref object
    address*: string
    code*: seq[Instruction]
    storage*: Table[string, int64]
    balance*: int64
    abi*: Table[string, int]  # function name -> entry point

proc newContext*(caller: string, value: int64 = 0, gasLimit = 100_000'i64): ExecutionContext =
  ExecutionContext(
    stack: @[],
    memory: @[],
    storage: initTable[string, int64](),
    events: @[],
    gas: gasLimit,
    caller: caller,
    value: value
  )

proc push*(ctx: var ExecutionContext, v: int64) =
  ctx.stack.add(v)
  dec ctx.gas  # Each operation costs gas

proc pop*(ctx: var ExecutionContext): int64 =
  if ctx.stack.len == 0:
    raise newException(IOError, "Stack underflow")
  ctx.stack.pop()

proc execute*(contract: Contract, ctx: var ExecutionContext): int64 =
  ctx.pc = 0
  
  while ctx.pc < contract.code.len:
    if ctx.gas <= 0:
      raise newException(IOError, "Out of gas")
    
    let instr = contract.code[ctx.pc]
    inc ctx.pc
    
    case instr.opcode
    of opPush:
      ctx.push(instr.value.get(0))
    
    of opPop:
      discard ctx.pop()
    
    of opAdd:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(a + b)
    
    of opSub:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(a - b)
    
    of opMul:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(a * b)
    
    of opDiv:
      let b = ctx.pop()
      let a = ctx.pop()
      if b == 0:
        raise newException(DivByZeroDefect, "Division by zero")
      ctx.push(a div b)
    
    of opEq:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(if a == b: 1 else: 0)
    
    of opLt:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(if a < b: 1 else: 0)
    
    of opGt:
      let b = ctx.pop()
      let a = ctx.pop()
      ctx.push(if a > b: 1 else: 0)
    
    of opJump:
      let target = ctx.pop()
      ctx.pc = int(target)
    
    of opJumpIf:
      let target = ctx.pop()
      let condition = ctx.pop()
      if condition != 0:
        ctx.pc = int(target)
    
    of opStore:
      let value = ctx.pop()
      let key = ctx.pop()
      contract.storage[$key] = value
      ctx.storage[$key] = value
    
    of opLoad:
      let key = ctx.pop()
      let value = contract.storage.getOrDefault($key, 0)
      ctx.push(value)
    
    of opEmit:
      let dataLen = int(ctx.pop())
      var eventData = initTable[string, int64]()
      for _ in 0..<dataLen:
        let v = ctx.pop()
        let k = ctx.pop()
        eventData[$k] = v
      let name = $ctx.pop()
      ctx.events.add(ContractEvent(name: name, data: eventData))
    
    of opReturn:
      return ctx.pop()
    
    of opRevert:
      let reason = ctx.pop()
      raise newException(IOError, &"Contract reverted: {reason}")
    
    else:
      raise newException(IOError, &"Unknown opcode: {instr.opcode}")
  
  0  # No explicit return

# Compile a simple "token transfer" contract
proc compileTokenContract*(): Contract =
  ## Simplified ERC-20-like token contract
  ## Functions:
  ## - mint(to, amount): mint tokens
  ## - transfer(from, to, amount): transfer tokens
  ## - balanceOf(addr): get balance
  
  var code: seq[Instruction]
  
  # balanceOf: LOAD address from stack
  let balanceOfEntry = code.len
  code.add(Instruction(opcode: opLoad))  # Load balance[address]
  code.add(Instruction(opcode: opReturn))
  
  result = Contract(
    address: "0x" & sha256Hex("token-contract")[0..39],
    code: code,
    storage: initTable[string, int64](),
    abi: {"balanceOf": balanceOfEntry}.toTable
  )
```

---

## สรุป

| Concept | Implementation |
|---------|---------------|
| Hash | SHA-256, Keccak-256, BLAKE2 |
| HMAC | HMAC-SHA256 |
| Merkle Tree | Binary tree of hashes |
| Proof of Work | Find nonce for target hash |
| ECDSA | secp256k1 (via OpenSSL) |
| Blockchain | Linked blocks with PoW |
| Smart Contract | Stack-based VM |

---

**Next**: [Part 75 - AI/ML Integration in Nim](../advanced/part75_aiml.md)
