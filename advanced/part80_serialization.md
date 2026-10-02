# Part 80 - Binary Serialization & Protocol Buffers

## บทนำ

Binary serialization มีประโยชน์เมื่อต้องการ:
- ขนาดข้อมูลเล็กกว่า JSON
- Parsing เร็วกว่า
- Type-safe schema
- เหมาะกับ high-performance RPC

---

## 1. Custom Binary Format

```nim
# binary_codec.nim
# Custom binary serialization ด้วย Nim

import std/[streams, endians, strutils, tables, options]

type
  BinaryWriter* = ref object
    stream*: StringStream

  BinaryReader* = ref object
    stream*: StringStream
    pos*: int

# Writer
proc newBinaryWriter*(): BinaryWriter =
  BinaryWriter(stream: newStringStream())

proc writeUint8*(w: BinaryWriter, v: uint8) =
  w.stream.write(v)

proc writeUint16*(w: BinaryWriter, v: uint16) =
  var buf: array[2, byte]
  bigEndian16(addr buf[0], unsafeAddr v)
  w.stream.writeData(addr buf[0], 2)

proc writeUint32*(w: BinaryWriter, v: uint32) =
  var buf: array[4, byte]
  bigEndian32(addr buf[0], unsafeAddr v)
  w.stream.writeData(addr buf[0], 4)

proc writeUint64*(w: BinaryWriter, v: uint64) =
  var buf: array[8, byte]
  bigEndian64(addr buf[0], unsafeAddr v)
  w.stream.writeData(addr buf[0], 8)

proc writeInt32*(w: BinaryWriter, v: int32) = w.writeUint32(cast[uint32](v))
proc writeInt64*(w: BinaryWriter, v: int64) = w.writeUint64(cast[uint64](v))

proc writeFloat32*(w: BinaryWriter, v: float32) =
  w.writeUint32(cast[uint32](v))

proc writeFloat64*(w: BinaryWriter, v: float64) =
  w.writeUint64(cast[uint64](v))

proc writeString*(w: BinaryWriter, s: string) =
  w.writeUint32(s.len.uint32)
  if s.len > 0:
    w.stream.write(s)

proc writeBool*(w: BinaryWriter, b: bool) =
  w.writeUint8(if b: 1u8 else: 0u8)

proc writeBytes*(w: BinaryWriter, data: seq[byte]) =
  w.writeUint32(data.len.uint32)
  if data.len > 0:
    w.stream.writeData(unsafeAddr data[0], data.len)

proc bytes*(w: BinaryWriter): seq[byte] =
  let s = w.stream.data
  result = newSeq[byte](s.len)
  if s.len > 0:
    copyMem(addr result[0], unsafeAddr s[0], s.len)

# Variable-length encoding (like protobuf varint)
proc writeVarInt*(w: BinaryWriter, v: uint64) =
  var x = v
  while x >= 0x80:
    w.writeUint8(uint8(x and 0x7F) or 0x80)
    x = x shr 7
  w.writeUint8(uint8(x))

proc writeZigZag*(w: BinaryWriter, v: int64) =
  # ZigZag encoding: map negative numbers to positive
  let encoded = uint64((v shl 1) xor (v shr 63))
  w.writeVarInt(encoded)

# Reader
proc newBinaryReader*(data: seq[byte]): BinaryReader =
  var s = newString(data.len)
  if data.len > 0:
    copyMem(addr s[0], unsafeAddr data[0], data.len)
  BinaryReader(stream: newStringStream(s))

proc readUint8*(r: BinaryReader): uint8 =
  r.stream.read(result)

proc readUint16*(r: BinaryReader): uint16 =
  var buf: array[2, byte]
  discard r.stream.readData(addr buf[0], 2)
  bigEndian16(addr result, addr buf[0])

proc readUint32*(r: BinaryReader): uint32 =
  var buf: array[4, byte]
  discard r.stream.readData(addr buf[0], 4)
  bigEndian32(addr result, addr buf[0])

proc readUint64*(r: BinaryReader): uint64 =
  var buf: array[8, byte]
  discard r.stream.readData(addr buf[0], 8)
  bigEndian64(addr result, addr buf[0])

proc readInt32*(r: BinaryReader): int32 = cast[int32](r.readUint32())
proc readInt64*(r: BinaryReader): int64 = cast[int64](r.readUint64())
proc readFloat32*(r: BinaryReader): float32 = cast[float32](r.readUint32())
proc readFloat64*(r: BinaryReader): float64 = cast[float64](r.readUint64())

proc readString*(r: BinaryReader): string =
  let len = r.readUint32().int
  if len == 0: return ""
  result = newString(len)
  discard r.stream.readData(addr result[0], len)

proc readBool*(r: BinaryReader): bool = r.readUint8() != 0

proc readBytes*(r: BinaryReader): seq[byte] =
  let len = r.readUint32().int
  result = newSeq[byte](len)
  if len > 0:
    discard r.stream.readData(addr result[0], len)

proc readVarInt*(r: BinaryReader): uint64 =
  var shift = 0
  result = 0
  while true:
    let b = r.readUint8()
    result = result or (uint64(b and 0x7F) shl shift)
    if (b and 0x80) == 0: break
    shift += 7

proc readZigZag*(r: BinaryReader): int64 =
  let encoded = r.readVarInt()
  result = int64((encoded shr 1) xor -(encoded and 1))
```

---

## 2. Protobuf-Like Schema (Manual)

```nim
# proto_manual.nim
# Manual protobuf-style encoding (wire types)

import std/[strformat, tables, options]

type
  WireType* = enum
    wtVarint = 0       # int32, int64, uint32, uint64, bool, enum
    wt64Bit = 1        # fixed64, double
    wtLenDelim = 2     # string, bytes, embedded messages
    wt32Bit = 5        # fixed32, float

  FieldTag* = object
    fieldNumber*: int
    wireType*: WireType

  ProtoWriter* = ref object
    buf*: seq[byte]

  ProtoReader* = ref object
    buf*: seq[byte]
    pos*: int

# Protobuf varint encoding
proc encodeVarint*(v: uint64): seq[byte] =
  var x = v
  while x >= 0x80:
    result.add(byte(x and 0x7F) or 0x80)
    x = x shr 7
  result.add(byte(x))

proc decodeVarint*(buf: seq[byte], pos: var int): uint64 =
  var shift = 0
  while pos < buf.len:
    let b = buf[pos]
    inc pos
    result = result or (uint64(b and 0x7F) shl shift)
    if (b and 0x80) == 0: break
    shift += 7

proc encodeTag*(fieldNumber: int, wireType: WireType): seq[byte] =
  let tag = uint64(fieldNumber shl 3) or uint64(wireType)
  encodeVarint(tag)

proc decodeTag*(buf: seq[byte], pos: var int): FieldTag =
  let v = decodeVarint(buf, pos)
  FieldTag(fieldNumber: int(v shr 3), wireType: WireType(v and 0x7))

# Write primitives
proc newProtoWriter*(): ProtoWriter = ProtoWriter(buf: @[])

proc writeVarintField*(w: ProtoWriter, fieldNum: int, v: uint64) =
  w.buf.add(encodeTag(fieldNum, wtVarint))
  w.buf.add(encodeVarint(v))

proc writeInt32Field*(w: ProtoWriter, fieldNum: int, v: int32) =
  w.writeVarintField(fieldNum, cast[uint64](int64(v)))

proc writeInt64Field*(w: ProtoWriter, fieldNum: int, v: int64) =
  w.writeVarintField(fieldNum, cast[uint64](v))

proc writeBoolField*(w: ProtoWriter, fieldNum: int, v: bool) =
  w.writeVarintField(fieldNum, if v: 1 else: 0)

proc writeStringField*(w: ProtoWriter, fieldNum: int, s: string) =
  w.buf.add(encodeTag(fieldNum, wtLenDelim))
  w.buf.add(encodeVarint(s.len.uint64))
  for c in s: w.buf.add(byte(c))

proc writeBytesField*(w: ProtoWriter, fieldNum: int, data: seq[byte]) =
  w.buf.add(encodeTag(fieldNum, wtLenDelim))
  w.buf.add(encodeVarint(data.len.uint64))
  w.buf.add(data)

proc writeFixed32Field*(w: ProtoWriter, fieldNum: int, v: uint32) =
  w.buf.add(encodeTag(fieldNum, wt32Bit))
  w.buf.add([byte(v), byte(v shr 8), byte(v shr 16), byte(v shr 24)])

proc writeFixed64Field*(w: ProtoWriter, fieldNum: int, v: uint64) =
  w.buf.add(encodeTag(fieldNum, wt64Bit))
  for i in 0..7:
    w.buf.add(byte(v shr (i * 8)))

proc writeFloatField*(w: ProtoWriter, fieldNum: int, v: float32) =
  w.writeFixed32Field(fieldNum, cast[uint32](v))

proc writeDoubleField*(w: ProtoWriter, fieldNum: int, v: float64) =
  w.writeFixed64Field(fieldNum, cast[uint64](v))

proc writeEmbedded*(w: ProtoWriter, fieldNum: int, inner: ProtoWriter) =
  w.writeBytesField(fieldNum, inner.buf)

# Example: User message
# message User {
#   int32 id = 1;
#   string name = 2;
#   string email = 3;
#   bool active = 4;
#   float score = 5;
# }
type
  User* = object
    id*: int32
    name*: string
    email*: string
    active*: bool
    score*: float32

proc encode*(u: User): seq[byte] =
  let w = newProtoWriter()
  if u.id != 0: w.writeInt32Field(1, u.id)
  if u.name.len > 0: w.writeStringField(2, u.name)
  if u.email.len > 0: w.writeStringField(3, u.email)
  if u.active: w.writeBoolField(4, u.active)
  if u.score != 0: w.writeFloatField(5, u.score)
  w.buf

proc decode*(buf: seq[byte], T: typedesc[User]): User =
  var pos = 0
  while pos < buf.len:
    let tag = decodeTag(buf, pos)
    case tag.fieldNumber
    of 1: result.id = int32(decodeVarint(buf, pos))
    of 2:
      let len = int(decodeVarint(buf, pos))
      result.name = newString(len)
      for i in 0..<len:
        result.name[i] = char(buf[pos + i])
      pos += len
    of 3:
      let len = int(decodeVarint(buf, pos))
      result.email = newString(len)
      for i in 0..<len:
        result.email[i] = char(buf[pos + i])
      pos += len
    of 4: result.active = decodeVarint(buf, pos) != 0
    of 5:
      var v: uint32
      v = uint32(buf[pos]) or
          (uint32(buf[pos+1]) shl 8) or
          (uint32(buf[pos+2]) shl 16) or
          (uint32(buf[pos+3]) shl 24)
      result.score = cast[float32](v)
      pos += 4
    else:
      # Skip unknown field
      case tag.wireType
      of wtVarint: discard decodeVarint(buf, pos)
      of wt64Bit: pos += 8
      of wtLenDelim:
        let len = int(decodeVarint(buf, pos))
        pos += len
      of wt32Bit: pos += 4
      else: break
```

---

## 3. MessagePack Implementation

```nim
# msgpack.nim
# MessagePack binary format encoder/decoder

import std/[tables, streams, strformat, math]

type
  MsgPackType* = enum
    mpNil, mpBool, mpInt, mpUInt, mpFloat32, mpFloat64
    mpString, mpBinary, mpArray, mpMap, mpExt

  MsgPackValue* = ref object
    case kind*: MsgPackType
    of mpNil: discard
    of mpBool: boolVal*: bool
    of mpInt: intVal*: int64
    of mpUInt: uintVal*: uint64
    of mpFloat32: f32Val*: float32
    of mpFloat64: f64Val*: float64
    of mpString: strVal*: string
    of mpBinary: binVal*: seq[byte]
    of mpArray: arrVal*: seq[MsgPackValue]
    of mpMap: mapVal*: seq[(MsgPackValue, MsgPackValue)]
    of mpExt:
      extType*: int8
      extData*: seq[byte]

proc nilVal*(): MsgPackValue = MsgPackValue(kind: mpNil)
proc boolVal*(v: bool): MsgPackValue = MsgPackValue(kind: mpBool, boolVal: v)
proc intVal*(v: int64): MsgPackValue = MsgPackValue(kind: mpInt, intVal: v)
proc uintVal*(v: uint64): MsgPackValue = MsgPackValue(kind: mpUInt, uintVal: v)
proc strVal*(v: string): MsgPackValue = MsgPackValue(kind: mpString, strVal: v)
proc arrVal*(v: seq[MsgPackValue]): MsgPackValue = MsgPackValue(kind: mpArray, arrVal: v)

proc pack*(v: MsgPackValue): seq[byte] =
  var buf: seq[byte]
  
  proc write8(x: byte) = buf.add(x)
  proc write16(x: uint16) =
    buf.add(byte(x shr 8)); buf.add(byte(x))
  proc write32(x: uint32) =
    buf.add(byte(x shr 24)); buf.add(byte(x shr 16))
    buf.add(byte(x shr 8)); buf.add(byte(x))
  proc write64(x: uint64) =
    write32(uint32(x shr 32)); write32(uint32(x))
  
  case v.kind
  of mpNil: write8(0xC0)
  of mpBool: write8(if v.boolVal: 0xC3 else: 0xC2)
  
  of mpInt:
    let x = v.intVal
    if x >= 0: buf.add(pack(uintVal(uint64(x))))
    elif x >= -32: write8(byte(0xE0 or uint8(x + 32)))
    elif x >= -128: write8(0xD0); write8(byte(x))
    elif x >= -32768: write8(0xD1); write16(uint16(x))
    elif x >= -2147483648: write8(0xD2); write32(uint32(x))
    else: write8(0xD3); write64(uint64(x))
  
  of mpUInt:
    let x = v.uintVal
    if x <= 127: write8(byte(x))
    elif x <= 255: write8(0xCC); write8(byte(x))
    elif x <= 65535: write8(0xCD); write16(uint16(x))
    elif x <= 4294967295: write8(0xCE); write32(uint32(x))
    else: write8(0xCF); write64(x)
  
  of mpFloat32:
    write8(0xCA); write32(cast[uint32](v.f32Val))
  
  of mpFloat64:
    write8(0xCB); write64(cast[uint64](v.f64Val))
  
  of mpString:
    let s = v.strVal
    if s.len <= 31: write8(byte(0xA0 or s.len))
    elif s.len <= 255: write8(0xD9); write8(byte(s.len))
    elif s.len <= 65535: write8(0xDA); write16(uint16(s.len))
    else: write8(0xDB); write32(uint32(s.len))
    for c in s: buf.add(byte(c))
  
  of mpArray:
    let arr = v.arrVal
    if arr.len <= 15: write8(byte(0x90 or arr.len))
    elif arr.len <= 65535: write8(0xDC); write16(uint16(arr.len))
    else: write8(0xDD); write32(uint32(arr.len))
    for item in arr: buf.add(pack(item))
  
  of mpMap:
    let m = v.mapVal
    if m.len <= 15: write8(byte(0x80 or m.len))
    elif m.len <= 65535: write8(0xDE); write16(uint16(m.len))
    else: write8(0xDF); write32(uint32(m.len))
    for (k, val) in m:
      buf.add(pack(k)); buf.add(pack(val))
  
  else: write8(0xC0)  # nil fallback
  
  buf

when isMainModule:
  let user = arrVal(@[
    strVal("Alice"),
    intVal(30),
    boolVal(true)
  ])
  
  let encoded = pack(user)
  echo &"Encoded {encoded.len} bytes: {encoded.mapIt(it.toHex).join(\" \")}"
```

---

## 4. FlatBuffers-Inspired Zero-Copy

```nim
# flatbuf.nim
# Zero-copy binary format (FlatBuffers-inspired)
# Data read directly from buffer without parsing

import std/[strformat, strutils]

type
  FlatBuilder* = ref object
    buf*: seq[byte]
    
  FlatTable* = object
    buf*: ptr UncheckedArray[byte]
    offset*: int  # Absolute position of table start

proc newFlatBuilder*(): FlatBuilder =
  FlatBuilder(buf: @[])

proc align*(builder: FlatBuilder, size: int) =
  let rem = builder.buf.len mod size
  if rem != 0:
    for _ in 0..<(size - rem):
      builder.buf.add(0)

proc writeInt32*(builder: FlatBuilder, v: int32): int =
  ## Returns offset of written value
  builder.align(4)
  result = builder.buf.len
  let bytes = cast[array[4, byte]](v)
  builder.buf.add(bytes[0]); builder.buf.add(bytes[1])
  builder.buf.add(bytes[2]); builder.buf.add(bytes[3])

proc writeString*(builder: FlatBuilder, s: string): int =
  ## Returns offset of string
  result = builder.writeInt32(s.len.int32)  # Length prefix
  for c in s: builder.buf.add(byte(c))
  builder.buf.add(0)  # Null terminator

proc readInt32*(buf: seq[byte], offset: int): int32 =
  cast[int32]([buf[offset], buf[offset+1], buf[offset+2], buf[offset+3]])

proc readString*(buf: seq[byte], offset: int): string =
  let len = readInt32(buf, offset).int
  result = newString(len)
  for i in 0..<len:
    result[i] = char(buf[offset + 4 + i])
```

---

## สรุป

| Format | Size | Speed | Schema | Use Case |
|--------|------|-------|--------|----------|
| JSON | Large | Slow | No | APIs, config |
| Custom Binary | Small | Fast | Manual | Internal protocol |
| Protobuf-like | Small | Fast | Yes | RPC, storage |
| MessagePack | Medium | Fast | No | Cache, queues |
| FlatBuffers | Small | Fastest | Yes | Zero-copy reads |

---

**Next**: [Part 81 - CLI Tools & TUI Development](../advanced/part81_cli_tui.md)
