# Part 68 - Nim สำหรับ Embedded Systems และ IoT

## บทนำ

Nim เหมาะกับ embedded systems เพราะ:
- Compile เป็น C แล้วค่อย compile ด้วย cross-compiler (arm-none-eabi-gcc)
- Control ระดับ memory ได้ด้วย `--mm:none`
- Zero-cost abstractions
- ขนาด binary เล็กมากถ้า strip ไม่ใช้ standard library

---

## 1. Nim ไม่ใช้ Standard Library (freestanding)

```nim
# bare_metal.nim
# Compile: nim c --mm:none --os:any --cpu:arm -d:release --noMain bare_metal.nim

# ปิดการใช้ GC และ stdlib ทั้งหมด
{.push stackTrace:off, boundChecks:off, assertions:off.}

# ARM Cortex-M4 register definitions
const
  # STM32F4 peripheral base addresses
  RCC_BASE*    = 0x40023800'u32
  GPIOA_BASE*  = 0x40020000'u32
  GPIOC_BASE*  = 0x40020800'u32
  
  RCC_AHB1ENR* = RCC_BASE + 0x30
  GPIOA_MODER* = GPIOA_BASE + 0x00
  GPIOA_ODR*   = GPIOA_BASE + 0x14
  GPIOC_MODER* = GPIOC_BASE + 0x00
  GPIOC_IDR*   = GPIOC_BASE + 0x10

type
  Reg32* = ptr uint32

proc reg*(address: uint32): Reg32 {.inline.} =
  cast[Reg32](address)

template setBit*(r: Reg32, bit: uint32) =
  r[] = r[] or (1'u32 shl bit)

template clearBit*(r: Reg32, bit: uint32) =
  r[] = r[] and not(1'u32 shl bit)

template testBit*(r: Reg32, bit: uint32): bool =
  (r[] and (1'u32 shl bit)) != 0

# GPIO initialization for LED (PA5 on Nucleo board)
proc gpioInit*() =
  # Enable GPIOA clock
  reg(RCC_AHB1ENR).setBit(0)
  
  # Set PA5 as output (MODER bits 11:10 = 01)
  let moder = reg(GPIOA_MODER)
  moder[] = (moder[] and not(0b11'u32 shl 10)) or (0b01'u32 shl 10)

proc ledOn*() =
  reg(GPIOA_ODR).setBit(5)

proc ledOff*() =
  reg(GPIOA_ODR).clearBit(5)

proc ledToggle*() =
  let odr = reg(GPIOA_ODR)
  odr[] = odr[] xor (1'u32 shl 5)

# Simple busy-wait delay (not recommended for real apps)
proc delay*(cycles: uint32) =
  var i = cycles
  while i > 0:
    dec i
    # Prevent optimizer from removing loop
    {.emit: "asm volatile(\"\");" .}

# Entry point — called by startup code
proc NimMain*() {.exportc.} =
  gpioInit()
  
  while true:
    ledOn()
    delay(1_000_000)
    ledOff()
    delay(1_000_000)

{.pop.}
```

---

## 2. UART Communication

```nim
# uart.nim
# UART สำหรับ debug output บน STM32

{.push stackTrace:off, boundChecks:off.}

const
  USART2_BASE*  = 0x40004400'u32
  USART2_SR*    = USART2_BASE + 0x00  # Status register
  USART2_DR*    = USART2_BASE + 0x04  # Data register
  USART2_BRR*   = USART2_BASE + 0x08  # Baud rate register
  USART2_CR1*   = USART2_BASE + 0x0C  # Control register 1
  
  RCC_APB1ENR* = RCC_BASE + 0x40
  
  # USART_SR bits
  USART_SR_TXE  = 7'u32  # Transmit Data Register Empty
  USART_SR_RXNE = 5'u32  # Read Data Register Not Empty
  USART_SR_TC   = 6'u32  # Transmission Complete

proc uartInit*(baudRate: uint32 = 115200, systemClock: uint32 = 84_000_000) =
  # Enable USART2 clock
  reg(RCC_APB1ENR).setBit(17)
  
  # Configure baud rate: BRR = fCK / baudRate
  let brr = systemClock div baudRate
  reg(USART2_BRR)[] = brr
  
  # Enable USART2, TX, RX
  reg(USART2_CR1)[] = (1'u32 shl 13) or  # UE: USART Enable
                       (1'u32 shl 3)  or  # TE: Transmitter Enable
                       (1'u32 shl 2)      # RE: Receiver Enable

proc uartSendByte*(b: uint8) =
  # Wait for TXE (transmit register empty)
  while not reg(USART2_SR).testBit(USART_SR_TXE):
    discard
  reg(USART2_DR)[] = uint32(b)

proc uartSend*(s: string) =
  for c in s:
    uartSendByte(uint8(ord(c)))

proc uartSendLine*(s: string) =
  uartSend(s)
  uartSendByte(uint8('\r'))
  uartSendByte(uint8('\n'))

proc uartReceiveByte*(): uint8 =
  while not reg(USART2_SR).testBit(USART_SR_RXNE):
    discard
  uint8(reg(USART2_DR)[])

# Printf-like output without stdlib
proc uartPrintHex*(n: uint32) =
  const digits = "0123456789ABCDEF"
  uartSend("0x")
  for i in countdown(7, 0):
    uartSendByte(uint8(ord(digits[(n shr (i * 4)) and 0xF])))

proc uartPrintDec*(n: uint32) =
  if n == 0:
    uartSendByte(uint8(ord('0')))
    return
  
  var buf: array[10, uint8]
  var pos = 0
  var num = n
  
  while num > 0:
    buf[pos] = uint8(ord('0') + (num mod 10))
    inc pos
    num = num div 10
  
  for i in countdown(pos - 1, 0):
    uartSendByte(buf[i])

{.pop.}
```

---

## 3. I2C Protocol

```nim
# i2c.nim
# I2C master implementation สำหรับ sensor communication

{.push stackTrace:off.}

const
  I2C1_BASE*   = 0x40005400'u32
  I2C1_CR1*    = I2C1_BASE + 0x00
  I2C1_CR2*    = I2C1_BASE + 0x04
  I2C1_OAR1*   = I2C1_BASE + 0x08
  I2C1_DR*     = I2C1_BASE + 0x10
  I2C1_SR1*    = I2C1_BASE + 0x14
  I2C1_SR2*    = I2C1_BASE + 0x18
  I2C1_CCR*    = I2C1_BASE + 0x1C
  I2C1_TRISE*  = I2C1_BASE + 0x20

type
  I2CError* = enum
    i2cOk
    i2cTimeout
    i2cNack
    i2cArbitrationLost

proc i2cInit*(clockHz: uint32 = 400_000, pclk1Hz: uint32 = 42_000_000) =
  # Reset I2C
  reg(I2C1_CR1)[] = 1'u32 shl 15  # SWRST
  reg(I2C1_CR1)[] = 0
  
  # Set PCLK1 frequency in MHz
  reg(I2C1_CR2)[] = pclk1Hz div 1_000_000
  
  # Fast mode (400kHz): CCR = PCLK1 / (3 * clockHz) for duty cycle 16/9
  let ccr = pclk1Hz div (25 * clockHz)
  reg(I2C1_CCR)[] = ccr or (1'u32 shl 15) or (1'u32 shl 14)
  
  # TRISE for fast mode: PCLK1_MHz * 0.3 + 1
  reg(I2C1_TRISE)[] = (pclk1Hz div 1_000_000) * 3 div 10 + 1
  
  # Enable I2C
  reg(I2C1_CR1)[] = 1'u32  # PE: Peripheral Enable

proc i2cStart*(): I2CError =
  reg(I2C1_CR1)[] = reg(I2C1_CR1)[] or (1'u32 shl 8)  # Generate START
  var timeout = 10000'u32
  while not reg(I2C1_SR1).testBit(0):  # SB: Start bit
    dec timeout
    if timeout == 0: return i2cTimeout
  i2cOk

proc i2cStop*() =
  reg(I2C1_CR1)[] = reg(I2C1_CR1)[] or (1'u32 shl 9)  # Generate STOP

proc i2cWriteByte*(data: uint8): I2CError =
  reg(I2C1_DR)[] = uint32(data)
  var timeout = 10000'u32
  while not reg(I2C1_SR1).testBit(7):  # TXE: TX empty
    if reg(I2C1_SR1).testBit(10):  # AF: ACK failure
      return i2cNack
    dec timeout
    if timeout == 0: return i2cTimeout
  i2cOk

proc i2cReadByte*(ack: bool = true): uint8 =
  if ack:
    reg(I2C1_CR1)[] = reg(I2C1_CR1)[] or (1'u32 shl 10)  # ACK
  else:
    reg(I2C1_CR1)[] = reg(I2C1_CR1)[] and not(1'u32 shl 10)  # NACK
  
  var timeout = 10000'u32
  while not reg(I2C1_SR1).testBit(6):  # RXNE: RX not empty
    dec timeout
    if timeout == 0: return 0xFF
  
  uint8(reg(I2C1_DR)[])

# High-level I2C transaction
proc i2cWrite*(addr: uint8, data: openArray[uint8]): I2CError =
  let err = i2cStart()
  if err != i2cOk: return err
  
  # Send address + write bit
  let e = i2cWriteByte(addr shl 1)
  if e != i2cOk:
    i2cStop()
    return e
  
  # Read SR2 to clear ADDR flag
  discard reg(I2C1_SR2)[]
  
  for b in data:
    let e = i2cWriteByte(b)
    if e != i2cOk:
      i2cStop()
      return e
  
  i2cStop()
  i2cOk

proc i2cRead*(addr: uint8, reg: uint8, buf: var openArray[uint8]): I2CError =
  # Write register address
  let e = i2cWrite(addr, [reg])
  if e != i2cOk: return e
  
  # Repeated start for read
  let err = i2cStart()
  if err != i2cOk: return err
  
  let e2 = i2cWriteByte((addr shl 1) or 1)  # Read bit
  if e2 != i2cOk:
    i2cStop()
    return e2
  
  discard reg(I2C1_SR2)[]
  
  for i in 0..<buf.len:
    buf[i] = i2cReadByte(ack = i < buf.len - 1)
  
  i2cStop()
  i2cOk

{.pop.}
```

---

## 4. BME280 Temperature/Humidity/Pressure Sensor

```nim
# bme280.nim
# Driver สำหรับ BME280 sensor ผ่าน I2C

{.push stackTrace:off.}

const
  BME280_ADDR*          = 0x76'u8
  BME280_REG_ID*        = 0xD0'u8
  BME280_REG_RESET*     = 0xE0'u8
  BME280_REG_CTRL_HUM*  = 0xF2'u8
  BME280_REG_STATUS*    = 0xF3'u8
  BME280_REG_CTRL_MEAS* = 0xF4'u8
  BME280_REG_CONFIG*    = 0xF5'u8
  BME280_REG_DATA*      = 0xF7'u8
  BME280_CALIB00*       = 0x88'u8
  BME280_CALIB26*       = 0xE1'u8
  BME280_CHIP_ID*       = 0x60'u8

type
  BME280Calib* = object
    T1*: uint16
    T2*, T3*: int16
    P1*: uint16
    P2*, P3*, P4*, P5*, P6*, P7*, P8*, P9*: int16
    H1*: uint8
    H2*: int16
    H3*: uint8
    H4*, H5*: int16
    H6*: int8

  BME280Data* = object
    temperature*: float  # Celsius
    humidity*: float     # %RH
    pressure*: float     # hPa

  BME280* = object
    calib*: BME280Calib
    tFine*: int32

proc bme280Init*(sensor: var BME280): bool =
  # Check chip ID
  var id: array[1, uint8]
  if i2cRead(BME280_ADDR, BME280_REG_ID, id) != i2cOk:
    return false
  if id[0] != BME280_CHIP_ID:
    return false
  
  # Read calibration data
  var calib1: array[26, uint8]
  var calib2: array[7, uint8]
  
  if i2cRead(BME280_ADDR, BME280_CALIB00, calib1) != i2cOk:
    return false
  if i2cRead(BME280_ADDR, BME280_CALIB26, calib2) != i2cOk:
    return false
  
  # Parse calibration
  sensor.calib.T1 = uint16(calib1[0]) or (uint16(calib1[1]) shl 8)
  sensor.calib.T2 = int16(uint16(calib1[2]) or (uint16(calib1[3]) shl 8))
  sensor.calib.T3 = int16(uint16(calib1[4]) or (uint16(calib1[5]) shl 8))
  
  # Configure: normal mode, 16x oversampling, 500ms standby
  discard i2cWrite(BME280_ADDR, [BME280_REG_CTRL_HUM, 0x05'u8])   # H x16
  discard i2cWrite(BME280_ADDR, [BME280_REG_CTRL_MEAS, 0xB7'u8])  # T x16, P x16, Normal
  discard i2cWrite(BME280_ADDR, [BME280_REG_CONFIG, 0xA0'u8])     # 1000ms standby
  
  true

proc compensateTemp*(sensor: var BME280, rawTemp: int32): float =
  let var1 = ((rawTemp shr 3) - (int32(sensor.calib.T1) shl 1)) * int32(sensor.calib.T2) shr 11
  let var2 = (((rawTemp shr 4) - int32(sensor.calib.T1)) * ((rawTemp shr 4) - int32(sensor.calib.T1)) shr 12) * int32(sensor.calib.T3) shr 14
  sensor.tFine = var1 + var2
  float(sensor.tFine * 5 + 128) / 32768.0

proc bme280Read*(sensor: var BME280): BME280Data =
  var raw: array[8, uint8]
  if i2cRead(BME280_ADDR, BME280_REG_DATA, raw) != i2cOk:
    return BME280Data()
  
  let rawPress = (int32(raw[0]) shl 12) or (int32(raw[1]) shl 4) or (int32(raw[2]) shr 4)
  let rawTemp  = (int32(raw[3]) shl 12) or (int32(raw[4]) shl 4) or (int32(raw[5]) shr 4)
  let rawHum   = (int32(raw[6]) shl 8)  or int32(raw[7])
  
  let temp = sensor.compensateTemp(rawTemp)
  
  # Simplified pressure and humidity compensation
  let var1p = float(sensor.tFine) / 2.0 - 64000.0
  let pressure = 1013.25  # Placeholder — real formula is more complex
  
  let humUncomp = float(sensor.tFine) - 76800.0
  let humidity = if humUncomp != 0.0:
    float(rawHum) / 65536.0
  else: 0.0
  
  BME280Data(temperature: temp, pressure: pressure, humidity: humidity)

{.pop.}
```

---

## 5. FreeRTOS Integration

```nim
# freertos_nim.nim
# ใช้ FreeRTOS API จาก Nim ผ่าน C FFI

# FreeRTOS types
type
  TaskHandle* = pointer
  QueueHandle* = pointer
  SemaphoreHandle* = pointer
  TickType* = uint32

const
  portMAX_DELAY* = TickType.high
  pdPASS* = 1'i32
  pdFAIL* = 0'i32
  pdTRUE* = 1'i32
  pdFALSE* = 0'i32

type
  TaskFunction* = proc(param: pointer) {.cdecl.}

# FreeRTOS API bindings
proc xTaskCreate*(
  pvTaskCode: TaskFunction,
  pcName: cstring,
  usStackDepth: uint16,
  pvParameters: pointer,
  uxPriority: uint32,
  pxCreatedTask: ptr TaskHandle
): int32 {.importc, header: "FreeRTOS.h".}

proc vTaskDelete*(xTask: TaskHandle) {.importc, header: "task.h".}
proc vTaskDelay*(xTicksToDelay: TickType) {.importc, header: "task.h".}
proc vTaskStartScheduler*() {.importc, header: "task.h".}
proc xTaskGetTickCount*(): TickType {.importc, header: "task.h".}

proc xQueueCreate*(uxQueueLength: uint32, uxItemSize: uint32): QueueHandle {.importc, header: "queue.h".}
proc xQueueSend*(xQueue: QueueHandle, pvItemToQueue: pointer, xTicksToWait: TickType): int32 {.importc, header: "queue.h".}
proc xQueueReceive*(xQueue: QueueHandle, pvBuffer: pointer, xTicksToWait: TickType): int32 {.importc, header: "queue.h".}

proc xSemaphoreCreateMutex*(): SemaphoreHandle {.importc, header: "semphr.h".}
proc xSemaphoreTake*(xSemaphore: SemaphoreHandle, xTicksToWait: TickType): int32 {.importc, header: "semphr.h".}
proc xSemaphoreGive*(xSemaphore: SemaphoreHandle): int32 {.importc, header: "semphr.h".}

# Helper: delay in milliseconds
proc delayMs*(ms: uint32) =
  vTaskDelay(TickType(ms))  # configTICK_RATE_HZ assumed to be 1000

# High-level task wrapper
template task*(name: string, stackSize: uint16 = 512, priority: uint32 = 1,
    body: untyped) =
  proc `task _ name`(param: pointer) {.cdecl.} =
    body
  discard xTaskCreate(`task _ name`, name, stackSize, nil, priority, nil)

# Mutex wrapper
type
  Mutex* = object
    handle*: SemaphoreHandle

proc newMutex*(): Mutex =
  Mutex(handle: xSemaphoreCreateMutex())

template withMutex*(m: Mutex, body: untyped) =
  if xSemaphoreTake(m.handle, portMAX_DELAY) == pdTRUE:
    try:
      body
    finally:
      discard xSemaphoreGive(m.handle)

# Message queue wrapper
type
  RTOSQueue*[T] = object
    handle*: QueueHandle

proc newRTOSQueue*[T](size: uint32): RTOSQueue[T] =
  RTOSQueue[T](handle: xQueueCreate(size, uint32(sizeof(T))))

proc send*[T](q: RTOSQueue[T], item: T, timeoutMs: uint32 = 0): bool =
  var tmp = item
  xQueueSend(q.handle, addr tmp, TickType(timeoutMs)) == pdPASS

proc receive*[T](q: RTOSQueue[T], timeoutMs: uint32 = portMAX_DELAY.uint32): T =
  var item: T
  discard xQueueReceive(q.handle, addr item, TickType(timeoutMs))
  item

# Example FreeRTOS application
var
  sensorQueue = newRTOSQueue[BME280Data](10)
  displayMutex = newMutex()

proc sensorTaskFn(param: pointer) {.cdecl.} =
  var sensor: BME280
  discard bme280Init(sensor)
  
  while true:
    let data = bme280Read(sensor)
    discard sensorQueue.send(data)
    delayMs(1000)

proc displayTaskFn(param: pointer) {.cdecl.} =
  while true:
    let data = sensorQueue.receive()
    
    withMutex(displayMutex):
      uartSend("T: ")
      # Print temperature...
      uartSend("C H: ")
      # Print humidity...
      uartSendLine("%")
    
    delayMs(100)

proc NimMain*() {.exportc.} =
  uartInit()
  
  discard xTaskCreate(sensorTaskFn, "Sensor", 256, nil, 2, nil)
  discard xTaskCreate(displayTaskFn, "Display", 256, nil, 1, nil)
  
  vTaskStartScheduler()
  # Should never reach here
  while true: discard
```

---

## 6. Raspberry Pi GPIO (Linux)

```nim
# rpi_gpio.nim
# GPIO control บน Raspberry Pi ผ่าน /dev/mem หรือ sysfs

import std/[os, strformat, posix]

type
  GPIOMode* = enum
    gmInput
    gmOutput

  GPIOEdge* = enum
    geNone
    geRising
    geFalling
    geBoth

  GPIO* = ref object
    pin*: int
    mode*: GPIOMode
    exported*: bool

const SYSFS_GPIO = "/sys/class/gpio"

proc exportPin*(pin: int) =
  let path = &"{SYSFS_GPIO}/gpio{pin}"
  if not dirExists(path):
    writeFile(&"{SYSFS_GPIO}/export", $pin)
    sleep(100)  # Wait for sysfs to create files

proc unexportPin*(pin: int) =
  writeFile(&"{SYSFS_GPIO}/unexport", $pin)

proc newGPIO*(pin: int, mode: GPIOMode): GPIO =
  exportPin(pin)
  
  let direction = if mode == gmOutput: "out" else: "in"
  writeFile(&"{SYSFS_GPIO}/gpio{pin}/direction", direction)
  
  GPIO(pin: pin, mode: mode, exported: true)

proc close*(gpio: GPIO) =
  if gpio.exported:
    unexportPin(gpio.pin)
    gpio.exported = false

proc write*(gpio: GPIO, value: bool) =
  assert gpio.mode == gmOutput
  writeFile(&"{SYSFS_GPIO}/gpio{gpio.pin}/value", if value: "1" else: "0")

proc read*(gpio: GPIO): bool =
  assert gpio.mode == gmInput
  readFile(&"{SYSFS_GPIO}/gpio{gpio.pin}/value").strip() == "1"

proc setEdge*(gpio: GPIO, edge: GPIOEdge) =
  let edgeStr = case edge
    of geNone: "none"
    of geRising: "rising"
    of geFalling: "falling"
    of geBoth: "both"
  writeFile(&"{SYSFS_GPIO}/gpio{gpio.pin}/edge", edgeStr)

proc waitForEdge*(gpio: GPIO, timeoutMs: int = -1): bool =
  ## Poll for GPIO edge event using select()
  gpio.setEdge(geBoth)
  
  let fd = open(&"{SYSFS_GPIO}/gpio{gpio.pin}/value", O_RDONLY or O_NONBLOCK)
  if fd < 0: return false
  
  defer: discard close(fd)
  
  # Read initial value to clear interrupt
  var buf: array[1, char]
  discard read(fd, addr buf[0], 1)
  
  var pollFd: TTPollfd
  pollFd.fd = fd
  pollFd.events = POLLPRI or POLLERR
  
  let ret = poll(addr pollFd, 1, cint(timeoutMs))
  ret > 0 and (pollFd.revents and POLLPRI) != 0

# /dev/mem direct memory access (faster, requires root)
type
  GPIOMemMap* = ref object
    fd*: cint
    gpioMap*: pointer
    mapSize*: int

const
  BCM2835_GPIO_BASE* = 0x20200000'u32  # Pi 1/2
  BCM2711_GPIO_BASE* = 0xFE200000'u32  # Pi 4
  BLOCK_SIZE* = 4096

proc newGPIOMemMap*(gpioBase: uint32 = BCM2711_GPIO_BASE): GPIOMemMap =
  let fd = open("/dev/mem", O_RDWR or O_SYNC)
  if fd < 0:
    raise newException(IOError, "Cannot open /dev/mem — need root")
  
  let map = mmap(nil, BLOCK_SIZE, PROT_READ or PROT_WRITE,
    MAP_SHARED, fd, Off(gpioBase))
  
  if map == MAP_FAILED:
    discard close(fd)
    raise newException(IOError, "mmap failed")
  
  GPIOMemMap(fd: fd, gpioMap: map, mapSize: BLOCK_SIZE)

proc destroy*(m: GPIOMemMap) =
  discard munmap(m.gpioMap, m.mapSize)
  discard close(m.fd)

proc setOutput*(m: GPIOMemMap, pin: int) =
  ## Set GPIO pin as output via direct register access
  let regOffset = pin div 10
  let bitOffset = (pin mod 10) * 3
  
  let gpfsel = cast[ptr uint32](cast[int](m.gpioMap) + regOffset * 4)
  gpfsel[] = (gpfsel[] and not(7'u32 shl bitOffset)) or (1'u32 shl bitOffset)

proc setHigh*(m: GPIOMemMap, pin: int) =
  ## GPSET register: offset 7 (SET0), pin bitmask
  let gpset = cast[ptr uint32](cast[int](m.gpioMap) + 28)  # GPSET0 offset
  gpset[] = 1'u32 shl pin

proc setLow*(m: GPIOMemMap, pin: int) =
  ## GPCLR register: offset 10 (CLR0)
  let gpclr = cast[ptr uint32](cast[int](m.gpioMap) + 40)  # GPCLR0 offset
  gpclr[] = 1'u32 shl pin

when isMainModule:
  # Example: Blink LED on pin 17 using sysfs
  let led = newGPIO(17, gmOutput)
  
  for _ in 0..9:
    led.write(true)
    sleep(500)
    led.write(false)
    sleep(500)
  
  led.close()
```

---

## 7. MQTT สำหรับ IoT

```nim
# mqtt_client.nim
# Simple MQTT client สำหรับ IoT telemetry

import std/[asyncdispatch, asyncnet, strformat, strutils, endians, json]
import std/[times, tables, options]

type
  MQTTPacketType* = enum
    mpConnect    = 1
    mpConnack    = 2
    mpPublish    = 3
    mpPuback     = 4
    mpSubscribe  = 8
    mpSuback     = 9
    mpPingreq    = 12
    mpPingresp   = 13
    mpDisconnect = 14

  MQTTQoS* = enum
    qoAt Most Once  = 0   # Fire and forget
    qoAt Least Once = 1   # At least once delivery
    qoExact Once    = 2   # Exactly once

  MQTTMessage* = object
    topic*: string
    payload*: string
    qos*: int
    retain*: bool

  MQTTClient* = ref object
    sock*: AsyncSocket
    host*: string
    port*: int
    clientId*: string
    keepAlive*: int
    connected*: bool
    packetId*: uint16
    subscribers*: Table[string, proc(msg: MQTTMessage)]

proc encodeLength*(length: int): seq[byte] =
  ## MQTT variable-length encoding
  var len = length
  result = @[]
  while true:
    var encodedByte = byte(len mod 128)
    len = len div 128
    if len > 0:
      encodedByte = encodedByte or 0x80
    result.add(encodedByte)
    if len == 0: break

proc encodeString*(s: string): seq[byte] =
  result = @[byte(s.len shr 8), byte(s.len and 0xFF)]
  for c in s:
    result.add(byte(ord(c)))

proc buildConnect*(clientId: string, keepAlive = 60,
    user = "", pass = "", cleanSession = true): seq[byte] =
  var payload: seq[byte]
  
  # Variable header
  var varHeader: seq[byte]
  # Protocol name: "MQTT"
  varHeader.add(encodeString("MQTT"))
  varHeader.add(0x04)  # Protocol level 4 (MQTT 3.1.1)
  
  # Connect flags
  var connectFlags = 0x02'u8  # Clean session
  if user.len > 0: connectFlags = connectFlags or 0x80
  if pass.len > 0: connectFlags = connectFlags or 0x40
  varHeader.add(connectFlags)
  
  # Keep alive (2 bytes big-endian)
  varHeader.add(byte(keepAlive shr 8))
  varHeader.add(byte(keepAlive and 0xFF))
  
  # Payload: client ID
  payload.add(encodeString(clientId))
  if user.len > 0: payload.add(encodeString(user))
  if pass.len > 0: payload.add(encodeString(pass))
  
  let totalLen = varHeader.len + payload.len
  
  result = @[byte(mpConnect.int shl 4)]  # Fixed header
  result.add(encodeLength(totalLen))
  result.add(varHeader)
  result.add(payload)

proc buildPublish*(topic, payload: string, qos = 0, retain = false,
    packetId: uint16 = 0): seq[byte] =
  var varHeader: seq[byte]
  var topicBytes = encodeString(topic)
  varHeader.add(topicBytes)
  
  if qos > 0:
    varHeader.add(byte(packetId shr 8))
    varHeader.add(byte(packetId and 0xFF))
  
  let fixedHeader = byte(mpPublish.int shl 4) or
    byte(qos shl 1) or
    (if retain: 1 else: 0)
  
  let payloadBytes = payload.mapIt(byte(ord(it)))
  let totalLen = varHeader.len + payloadBytes.len
  
  result = @[fixedHeader]
  result.add(encodeLength(totalLen))
  result.add(varHeader)
  result.add(payloadBytes)

proc buildSubscribe*(topic: string, qos = 0, packetId: uint16 = 1): seq[byte] =
  var varHeader = @[byte(packetId shr 8), byte(packetId and 0xFF)]
  var payload: seq[byte]
  payload.add(encodeString(topic))
  payload.add(byte(qos))
  
  let totalLen = varHeader.len + payload.len
  result = @[byte((mpSubscribe.int shl 4) or 0x02)]
  result.add(encodeLength(totalLen))
  result.add(varHeader)
  result.add(payload)

proc newMQTTClient*(host: string, port = 1883, clientId = "nim-client",
    keepAlive = 60): MQTTClient =
  MQTTClient(
    host: host,
    port: port,
    clientId: clientId,
    keepAlive: keepAlive,
    subscribers: initTable[string, proc(msg: MQTTMessage)]()
  )

proc connect*(c: MQTTClient) {.async.} =
  c.sock = newAsyncSocket()
  await c.sock.connect(c.host, Port(c.port))
  
  let packet = buildConnect(c.clientId, c.keepAlive)
  await c.sock.send(cast[string](packet))
  
  # Read CONNACK
  let header = await c.sock.recv(4)
  if header.len >= 4 and ord(header[0]) == (mpConnack.int shl 4):
    c.connected = ord(header[3]) == 0
    if c.connected:
      echo &"[MQTT] Connected to {c.host}:{c.port}"
    else:
      echo &"[MQTT] Connection refused: code {ord(header[3])}"
  
  # Start keepalive
  asyncCheck c.keepAliveLoop()

proc keepAliveLoop*(c: MQTTClient) {.async.} =
  while c.connected:
    await sleepAsync(c.keepAlive * 1000)
    await c.sock.send("\xC0\x00")  # PINGREQ

proc publish*(c: MQTTClient, topic, payload: string, qos = 0) {.async.} =
  inc c.packetId
  let packet = buildPublish(topic, payload, qos, packetId = c.packetId)
  await c.sock.send(cast[string](packet))
  echo &"[MQTT] Published to {topic}: {payload}"

proc subscribe*(c: MQTTClient, topic: string, qos = 0,
    handler: proc(msg: MQTTMessage)) {.async.} =
  c.subscribers[topic] = handler
  inc c.packetId
  let packet = buildSubscribe(topic, qos, c.packetId)
  await c.sock.send(cast[string](packet))
  echo &"[MQTT] Subscribed to {topic}"

when isMainModule:
  proc main() {.async.} =
    let client = newMQTTClient("broker.hivemq.com")
    await client.connect()
    
    await client.subscribe("sensors/+/temperature", handler = proc(msg: MQTTMessage) =
      echo &"Received: {msg.topic} = {msg.payload}"
    )
    
    # Publish sensor data
    for i in 1..5:
      let data = %*{"value": 25.0 + float(i), "unit": "C", "timestamp": epochTime().int}
      await client.publish("sensors/device001/temperature", $data)
      await sleepAsync(2000)
  
  waitFor main()
```

---

## สรุป

| Target | Nim Config | Use Case |
|--------|-----------|----------|
| STM32 (bare metal) | `--mm:none --os:any --cpu:arm` | Real-time, resource-constrained |
| FreeRTOS | C FFI bindings | RTOS tasks, queues, semaphores |
| Raspberry Pi | Standard Nim | Linux GPIO, I2C, MQTT |
| ESP32 | `--cpu:xtensa` | WiFi IoT |
| WASM | `--cpu:wasm32` | Web/edge |

---

**Next**: [Part 69 - Game Development with Nim](../advanced/part69_game_dev.md)
