# 🚀 좋습니다. FIFO 시작합니다.

UART에서 배운 내용을 그대로 이어서 가겠습니다.

오늘은 **FIFO의 개념 → 구조 → 동작 원리 → SpinalHDL 구현**까지 한 단계씩 가겠습니다.

---

# 1. FIFO가 왜 필요한가?

FIFO는 **First In, First Out**의 약자입니다.

말 그대로:

> **먼저 들어온 데이터가 먼저 나간다.**

입니다.

가장 쉬운 예는 줄서기입니다.

```text
들어오는 순서

A → B → C → D

       FIFO

A → B → C → D

나가는 순서
```

A가 먼저 들어갔으므로 A가 먼저 나옵니다.

---

# 2. UART에서 FIFO가 왜 필요한가?

지금까지 만든 UART RX를 생각해봅시다.

```text
UART RX
   │
   ▼
data
valid
```

예를 들어 UART가 다음 데이터를 계속 수신한다고 해봅시다.

```text
'A'
'B'
'C'
'D'
```

RX가 `valid`를 발생시킬 때마다 CPU가 **그 순간 바로 데이터를 읽어야 한다면** 문제가 생길 수 있습니다.

CPU가 바쁜 상황이라면:

```text
UART RX
   │
   ▼
'A' ──────► CPU ?

'B' ──────► CPU ?

'C' ──────► CPU ?
```

CPU가 `'A'`를 처리하는 동안 `'B'`, `'C'`가 들어올 수 있습니다.

그래서 중간에 저장 공간을 둡니다.

```text
UART RX
   │
   ▼
┌──────────────┐
│     FIFO     │
│              │
│ A            │
│ B            │
│ C            │
└──────────────┘
       │
       ▼
      CPU
```

UART RX가 빠르게 데이터를 넣고 CPU가 조금 늦게 데이터를 꺼내도 FIFO가 중간에서 버퍼 역할을 합니다.

---

# 3. FIFO의 핵심 인터페이스

가장 단순한 FIFO를 생각해봅시다.

```text
                 FIFO
          ┌────────────────┐
write ───►│                │
data  ───►│                │───► readData
          │                │
read  ───►│                │
          └────────────────┘
             │          │
             ▼          ▼
           full       empty
```

입력은 보통:

```text
write
writeData
```

출력은:

```text
readData
full
empty
```

그리고:

```text
read
```

가 있습니다.

---

# 4. FIFO의 내부 구조

FIFO를 구현하려면 결국 **메모리**가 필요합니다.

예를 들어:

> 깊이 4, 데이터 폭 8 bit

FIFO라면:

```text
Depth = 4
Data width = 8
```

이므로 내부에는:

```text
┌──────────┐
│ Address  │ Data
├──────────┤
│    0     │ A
│    1     │ B
│    2     │ C
│    3     │ D
└──────────┘
```

같은 저장 공간이 있습니다.

그리고 중요한 것이 두 개 있습니다.

```text
writePtr
readPtr
```

---

# 5. Write Pointer

`writePtr`는:

> **다음 데이터를 어디에 쓸 것인가?**

를 가리킵니다.

처음에는:

```text
writePtr = 0
```

입니다.

A를 write하면:

```text
memory[0] = A
writePtr = 1
```

B를 write하면:

```text
memory[1] = B
writePtr = 2
```

C:

```text
memory[2] = C
writePtr = 3
```

D:

```text
memory[3] = D
writePtr = 0
```

여기서 중요한 것이 있습니다.

**Pointer가 끝까지 가면 다시 처음으로 돌아갑니다.**

즉:

```text
0 → 1 → 2 → 3 → 0 → 1 → ...
```

이것을 **circular buffer**라고 합니다.

---

# 6. Read Pointer

`readPtr`는:

> **다음 데이터를 어디에서 읽을 것인가?**

를 가리킵니다.

처음에는:

```text
readPtr = 0
```

입니다.

A를 read하면:

```text
memory[0] → A

readPtr = 1
```

B를 read하면:

```text
memory[1] → B

readPtr = 2
```

이렇게 움직입니다.

---

# 7. FIFO의 진짜 핵심: Empty와 Full

여기서 아주 중요한 문제가 발생합니다.

Depth가 4라고 합시다.

처음:

```text
writePtr = 0
readPtr  = 0
```

아무 데이터도 없습니다.

```text
FIFO = EMPTY
```

그런데 데이터를 4개 넣으면:

```text
writePtr = 0
readPtr  = 0
```

가 다시 됩니다.

하지만 이번에는:

```text
FIFO = FULL
```

입니다.

즉:

```text
writePtr == readPtr
```

만 보고서는

```text
EMPTY
```

인지

```text
FULL
```

인지 구분할 수 없습니다.

---

# 8. 그래서 Occupancy Counter를 사용하자

교육용 FIFO에서는 이 방법이 아주 이해하기 쉽습니다.

```text
count
```

를 하나 둡니다.

`count`는:

> **현재 FIFO에 몇 개의 데이터가 들어 있는가?**

입니다.

Depth = 4라면:

```text
count = 0
```

→ EMPTY

```text
count = 1
```

→ 데이터 1개

```text
count = 2
```

→ 데이터 2개

```text
count = 3
```

→ 데이터 3개

```text
count = 4
```

→ FULL

따라서:

```text
empty = count == 0
full  = count == depth
```

가 됩니다.

이 방식은 처음 FIFO를 배우기에 매우 좋습니다.

---

# 9. Write가 발생하면

정상적인 write 조건을:

```text
write && !full
```

이라고 하겠습니다.

그러면:

```text
memory[writePtr] = writeData
writePtr         = writePtr + 1
count            = count + 1
```

입니다.

---

# 10. Read가 발생하면

정상적인 read 조건은:

```text
read && !empty
```

입니다.

그러면:

```text
readData = memory[readPtr]
readPtr  = readPtr + 1
count    = count - 1
```

입니다.

여기서 잠깐.

**SpinalHDL에서 Memory read 방식은 동기식/비동기식 등에 따라 동작이 달라질 수 있습니다.**

그래서 첫 FIFO에서는 복잡한 RAM inference를 바로 다루지 않고, **명확한 `Vec(Reg(...))` 기반 저장소**로 시작하겠습니다.

이렇게 하면 하드웨어 동작을 눈으로 따라가기 쉽습니다.

---

# 11. 첫 번째 FIFO

우선:

```text
Data width = 8
Depth      = 4
```

로 고정해서 배우겠습니다.

```scala id="fifobase1"
import spinal.core._

case class SimpleFifo(
    dataWidth: Int,
    depth: Int
) extends Component {

  require(depth >= 2)
  require(dataWidth >= 1)

  val io = new Bundle {
    val write     = in Bool()
    val writeData = in Bits(dataWidth bits)

    val read      = in Bool()
    val readData  = out Bits(dataWidth bits)

    val full      = out Bool()
    val empty     = out Bool()
  }

  // ------------------------------------------------------------
  // Memory
  // ------------------------------------------------------------

  val mem = Vec(
    Reg(Bits(dataWidth bits)) init(0),
    depth
  )

  // ------------------------------------------------------------
  // Pointer width
  // ------------------------------------------------------------

  val ptrWidth =
    log2Up(depth)

  val writePtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val readPtr =
    Reg(UInt(ptrWidth bits)) init(0)

  // ------------------------------------------------------------
  // Number of stored elements
  // ------------------------------------------------------------

  val countWidth =
    log2Up(depth + 1)

  val count =
    Reg(UInt(countWidth bits)) init(0)

  // ------------------------------------------------------------
  // Status
  // ------------------------------------------------------------

  io.empty := count === 0
  io.full  := count === depth

  // ------------------------------------------------------------
  // Read data
  // ------------------------------------------------------------

  io.readData := mem(readPtr)

  // ------------------------------------------------------------
  // Write / Read
  // ------------------------------------------------------------

  val doWrite =
    io.write && !io.full

  val doRead =
    io.read && !io.empty

  when(doWrite) {

    mem(writePtr) := io.writeData

    when(writePtr === depth - 1) {
      writePtr := 0
    } otherwise {
      writePtr := writePtr + 1
    }
  }

  when(doRead) {

    when(readPtr === depth - 1) {
      readPtr := 0
    } otherwise {
      readPtr := readPtr + 1
    }
  }

  // ------------------------------------------------------------
  // Occupancy counter
  // ------------------------------------------------------------

  when(doWrite && !doRead) {

    count := count + 1

  } elsewhen(!doWrite && doRead) {

    count := count - 1
  }
}
```

---

# 12. 여기서 정말 중요한 부분

이 부분을 잘 보세요.

```scala id="9a8x2p"
when(doWrite && !doRead) {
  count := count + 1

} elsewhen(!doWrite && doRead) {
  count := count - 1
}
```

그런데:

```text
write = 1
read  = 1
```

이면?

```text
doWrite = true
doRead  = true
```

입니다.

이 경우:

```text
데이터 하나 들어옴
데이터 하나 나감
```

이므로 FIFO에 들어 있는 데이터 개수는 **변하지 않아야 합니다.**

그래서:

```text
count = 그대로
```

입니다.

이것이 FIFO 설계에서 매우 중요한 부분입니다.

---

# 13. FIFO 상태 변화

초기:

```text
count = 0
empty = 1
full  = 0
```

A write:

```text
count = 1
empty = 0
full  = 0
```

B write:

```text
count = 2
```

C write:

```text
count = 3
```

D write:

```text
count = 4
empty = 0
full  = 1
```

이제 또 write를 하면:

```text
doWrite = write && !full
        = 1 && 0
        = 0
```

따라서 데이터가 들어가지 않습니다.

---

# 14. 그 다음 Read

현재:

```text
FIFO:

A
B
C
D
```

read:

```text
A ← read
```

그러면:

```text
FIFO:

B
C
D
```

그리고:

```text
count = 3
```

다시 read:

```text
B ← read
```

이제:

```text
C
D
```

입니다.

이것이 바로:

> **First In, First Out**

입니다.

---

# 15. 그런데 이 코드에서 한 가지 주의할 점

방금 구현한:

```scala
io.readData := mem(readPtr)
```

는 **read 요청이 발생한 순간에 데이터를 register에 저장하는 구조가 아닙니다.**

즉 `readData`는 현재 `readPtr`가 가리키는 메모리 내용을 보여주는 형태입니다.

따라서 첫 FIFO에서는 이것을:

> **"readData는 현재 FIFO의 head를 가리킨다."**

라고 이해하면 됩니다.

실제 FIFO IP나 FPGA RAM을 사용하면:

- synchronous read
- asynchronous read
- read-first
- write-first
- no-change

등의 메모리 동작 정책이 등장합니다.

이것은 나중에 제대로 다루겠습니다.

---

# 16. 다음은 Simulation

다음 단계에서는 바로 코드를 더 늘리지 않고,

```text
FIFO
 ↓
write A
write B
write C
 ↓
read A
read B
read C
```

를 simulation하고 waveform에서:

```text
writePtr
readPtr
count
full
empty
writeData
readData
```

를 직접 확인하겠습니다.

그리고 **동시에 `write`와 `read`가 들어오는 경우**, **FULL 상태에서 write**, **EMPTY 상태에서 read**까지 테스트하겠습니다.

이 테스트까지 통과시키면 그 다음에 **parameterized FIFO → UART RX FIFO 연결**로 넘어가면 됩니다.
