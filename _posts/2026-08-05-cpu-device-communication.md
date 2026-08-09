---
title: '[Device I/O] CPU는 Device와 어떻게 통신하는가'
date: 2026-08-05 00:20:00 +09:00
categories: [computer, device I/O]
published: false
tags:
  [
    device I/O,
    bus,
    device register,
    MMIO,
    doorbell
  ]
---

# 개요

CPU에서 다음과 같이 값을 저장한다고 생각해보자.

```c
*address = value;
```

이 명령은 DRAM에 값을 기록할 수도 있지만 device register에 명령을 전달할 수도 있다.

CPU는 SSD 내부 함수를 직접 호출하지 않는다.

CPU가 실행하는 것은 driver의 instruction이며, driver는 device가 공개한 register와 host memory의 자료 구조를 통해 작업을 요청한다.

```text
CPU가 driver instruction 실행
    ↓
Driver가 command를 host memory에 작성
    ↓
Driver가 device register에 작업 시작을 알림
    ↓
Device controller가 독립적으로 작업 수행
    ↓
Device가 완료 상태를 기록
```

# 학습 위치

| 항목 | 내용 |
| --- | --- |
| BFS Level | Level 2-D — Device I/O 기반 |
| 선수 글 | [[Low Latency Trading] CPU Interconnect와 Uncore — Core·Memory·PCIe 연결 경로](/posts/low-latency-cpu-interconnect-uncore/) |
| 다음 글 | [[Device I/O] Interrupt와 Polling, DMA는 어떻게 연결되는가](/posts/interrupt-polling-dma/) |

선수 글은 hardware transaction이 지나가는 topology를 다루고, 이 글은 software와 device가 MMIO, descriptor와 doorbell로 그 경로를 사용하는 방법에 집중한다.

# 1. CPU와 Device는 서로 다른 실행 주체다

CPU는 application과 OS의 instruction stream을 실행한다.

Device controller는 자체 state machine, processor 또는 firmware에 따라 비동기적으로 동작한다.

따라서 다음 호출처럼 CPU가 device의 내부 함수를 직접 실행하는 구조가 아니다.

```c
ssd_read(logical_block, buffer);
```

Driver가 이런 형태의 interface를 제공할 수는 있지만 그 내부에서는 register와 memory를 이용한 통신이 일어난다.

```text
Software function call
    ↓
Driver가 command와 buffer 준비
    ↓
Hardware protocol을 통해 device에 요청
```

CPU와 device는 서로 다른 속도로 동작한다.

CPU가 요청을 보낸 뒤 완료될 때까지 멈춰 있기보다 다른 작업을 수행하고 나중에 완료를 확인하는 비동기 구조가 필요하다.

# 2. Bus란 무엇인가

Bus는 CPU, memory, device 사이에 transaction을 전달하는 통신 체계다.

단순히 여러 부품을 연결하는 전선만을 의미하지 않는다.

Transaction에는 다음 정보가 필요하다.

- 어느 대상에 접근하는가
- Read와 write 중 어떤 작업인가
- 어떤 data를 전달하는가
- 요청에 대한 응답과 오류를 어떻게 처리하는가
- 여러 transaction의 순서를 어떻게 다루는가

현대 computer는 하나의 공유 bus만 사용하지 않는다.

```text
CPU Core
  ↓
CPU Interconnect
  ↓
Memory Controller 또는 PCIe Root Complex
  ↓
DRAM 또는 PCIe Device
```

각 구간의 protocol과 구현은 다를 수 있다.

Software는 이 복잡한 연결을 address와 OS interface를 통해 사용한다.

# 3. Device Controller

Device controller는 host의 요청을 해석하고 실제 hardware 동작을 제어한다.

NVMe SSD controller를 예로 들면 다음과 같은 작업을 수행한다.

- PCIe와 NVMe protocol 처리
- Submission Queue의 command fetch
- Host memory와 SSD 사이의 DMA
- Command scheduling
- FTL mapping
- NAND channel과 die scheduling
- 오류 검출과 복구
- Completion 생성

Controller가 어떤 작업을 hardware logic으로 수행하고 어떤 작업을 firmware로 수행하는지는 구현에 따라 달라진다.

# 4. Device Register

Device register는 controller가 host software에 공개하는 제어·상태 interface다.

| 역할 | 예시 |
| --- | --- |
| Configuration | 기능 활성화와 mode 설정 |
| Command notification | 새 command가 있음을 알리는 doorbell |
| Status | ready, busy, error 확인 |
| Interrupt control | interrupt mask와 pending 상태 처리 |

Device register와 일반 RAM은 동일한 load/store 문법으로 접근할 수 있어도 의미가 다르다.

## 4.1 일반 Memory Write

```c
memory[index] = value;
```

일반적으로 해당 위치에 값을 저장한다.

## 4.2 Register Write

```c
device_register = command;
```

단순한 값 저장이 아니라 다음 side effect를 일으킬 수 있다.

- Device 동작 시작
- Queue tail 갱신
- Interrupt acknowledge
- Error 상태 초기화
- Device reset

Register read에도 side effect가 존재할 수 있다.

예를 들어 status register를 읽는 순간 pending bit가 해제되는 read-to-clear register가 있을 수 있다.

따라서 device specification이 정의한 access width, 허용 값, 순서와 side effect를 확인해야 한다.

# 5. MMIO

MMIO(Memory-Mapped I/O)는 device register를 CPU의 physical address space 일부에 배치하는 방식이다.

CPU는 load/store instruction을 사용하지만 address에 따라 transaction의 목적지가 달라진다.

```text
CPU Load / Store
       ↓
Address Decode
       ├─ DRAM 영역 → Memory Controller
       └─ MMIO 영역 → Device Register
```

MMIO는 device register가 실제 RAM에 저장된다는 의미가 아니다.

동일한 address access instruction으로 device register에 접근할 수 있도록 mapping한다는 의미다.

## 5.1 MMIO Mapping

OS는 PCIe BAR 같은 hardware resource를 확인하고 driver가 접근할 수 있는 virtual address에 mapping한다.

개념적으로 다음 변환이 일어난다.

```text
Driver Virtual Address
    ↓ page table / OS mapping
MMIO Physical Address Range
    ↓ interconnect routing
Device Register
```

Driver는 mapping된 virtual address를 사용하지만 실제 transaction은 DRAM이 아니라 device로 전달된다.

## 5.2 MMIO 접근 시 주의점

MMIO는 일반 memory와 동일하게 다루면 안 된다.

- Compiler가 access를 제거하거나 합치지 않아야 한다.
- CPU와 interconnect의 ordering 규칙을 고려해야 한다.
- 일반 cacheable memory와 같은 cache 정책을 가정하면 안 된다.
- Register가 요구하는 access width를 지켜야 한다.
- Read/write가 일으키는 side effect를 고려해야 한다.

Linux driver에서는 pointer를 직접 역참조하기보다 `readl()`과 `writel()` 같은 accessor를 사용한다.

Architecture별 MMIO access와 ordering 차이를 OS가 제공하는 API를 통해 처리하기 위해서다.

# 6. volatile이면 충분할까

`volatile`은 compiler가 access를 불필요하다고 판단하여 제거하거나 임의로 합치는 것을 제한하는 데 사용될 수 있다.

그러나 `volatile`만으로 다음 사항이 모두 보장되지는 않는다.

- CPU의 memory ordering
- 다른 core와의 synchronization
- DMA 결과의 visibility
- Cache coherence가 없는 device와의 일관성
- Descriptor write와 doorbell write의 순서

따라서 MMIO에는 OS가 제공하는 accessor를 사용하고, DMA와 descriptor에는 해당 환경이 제공하는 memory barrier와 DMA API를 사용해야 한다.

# 7. Doorbell

Command 전체를 MMIO register에 하나씩 쓰는 구조는 MMIO transaction이 많아지고 확장성이 떨어진다.

고성능 device는 command를 host memory의 queue에 저장하고 MMIO에는 작은 알림만 쓰는 방식을 사용할 수 있다.

이 알림 register를 doorbell이라고 부른다.

```text
1. CPU가 host memory의 Submission Queue entry 작성
2. Entry가 device에 보이도록 ordering 보장
3. CPU가 MMIO doorbell에 새 tail 값 기록
4. Device가 Submission Queue entry를 DMA read
```

Doorbell은 command payload 자체가 아니다.

새 command가 memory의 어디까지 준비되었는지 device에 알려주는 신호에 가깝다.

# 8. Ordering이 필요한 이유

CPU 관점의 program order가 device가 관찰하는 순서와 항상 같다고 가정할 수는 없다.

다음 두 작업을 생각해보자.

```text
A. Host memory에 descriptor 작성
B. MMIO doorbell write
```

Device는 반드시 완성된 descriptor를 읽어야 하므로 A가 B보다 먼저 관찰되어야 한다.

순서가 보장되지 않으면 device가 doorbell을 먼저 확인하고 아직 완성되지 않은 descriptor를 DMA read할 수 있다.

구체적으로 어떤 barrier와 accessor가 필요한지는 CPU architecture, OS와 device의 DMA coherence model에 따라 달라진다.

중요한 것은 다음 불변식이다.

> Descriptor의 모든 field가 device에 보이는 상태가 된 후에 device에 descriptor의 존재를 알린다.

# 정리

CPU와 device의 통신은 다음 구조로 이해할 수 있다.

```text
CPU / Driver
    │ host memory에 command 작성
    │ MMIO register로 알림
    ▼
Bus / Interconnect
    ▼
Device Controller
    │ command 해석
    │ hardware와 firmware 동작
    ▼
Completion
```

Device register는 일반 memory와 의미가 다르며, register access는 hardware side effect를 일으킬 수 있다.

MMIO를 통해 CPU instruction으로 register에 접근할 수 있지만 compiler ordering, CPU ordering과 DMA visibility까지 `volatile` 하나로 해결할 수는 없다.

다음 글에서는 device가 완료 사실을 알리는 interrupt와 CPU가 직접 완료를 확인하는 polling을 비교하고, DMA가 이 흐름에서 어떤 역할을 하는지 살펴본다.
