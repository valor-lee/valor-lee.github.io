---
title: '[Device I/O] Interrupt와 Polling, DMA는 어떻게 연결되는가'
date: 2026-08-05 00:30:00 +09:00
categories: [computer, device I/O]
published: false
tags:
  [
    interrupt,
    polling,
    DMA,
    MSI-X,
    device I/O
  ]
---

# 개요

Device I/O를 공부할 때 interrupt, polling과 DMA를 하나의 선택지처럼 비교하기 쉽다.

하지만 이들은 서로 다른 문제를 해결한다.

```text
DMA
  → Payload를 누가 이동하는가?

Interrupt와 Polling
  → CPU가 완료 사실을 어떻게 발견하는가?
```

따라서 device가 DMA로 데이터를 옮긴 뒤 interrupt로 완료를 알릴 수도 있고, CPU가 completion queue를 polling하여 완료를 발견할 수도 있다.

# 1. 비동기 Device I/O

CPU와 device는 서로 독립적으로 동작한다.

CPU가 I/O command를 제출한 직후에는 device의 작업이 끝나지 않았을 수 있다.

```text
CPU: command 제출 ─────────────── completion 처리
                         ▲
Device:        command 처리 → data 이동 → 완료 기록
```

이 구조에는 두 가지 문제가 있다.

1. CPU와 device 사이에서 payload를 어떻게 이동할 것인가?
2. Device의 작업 완료를 CPU가 어떻게 알 것인가?

첫 번째 문제에 DMA가 사용되고, 두 번째 문제에 interrupt 또는 polling이 사용된다.

# 2. CPU Copy

DMA가 없다면 CPU가 device register나 작은 I/O window를 반복해서 읽고 쓰며 데이터를 옮기는 방식을 생각할 수 있다.

```text
Device → CPU Register → CPU Instruction → Memory
```

이 방식에서는 데이터 이동 중 CPU가 계속 instruction을 실행해야 한다.

Payload가 크거나 I/O 요청이 많으면 CPU가 단순 복사에 많은 시간을 사용한다.

# 3. DMA

DMA(Direct Memory Access)는 DMA-capable device 또는 DMA engine이 CPU 대신 memory와 device 사이의 데이터를 전송하는 방식이다.

CPU가 아무 일도 하지 않는다는 뜻은 아니다.

CPU와 driver는 전송을 준비하고 device는 실제 payload를 옮긴다.

```text
CPU / Driver
  1. Buffer 준비
  2. DMA mapping
  3. Descriptor 작성
  4. Device에 시작 알림
          ↓
Device DMA Engine
  5. Descriptor 읽기
  6. Payload read 또는 write
  7. Completion 기록
          ↓
CPU / Driver
  8. 완료 처리
  9. Buffer 재사용 또는 해제
```

## 3.1 DMA Read와 DMA Write

관점은 device를 기준으로 구분한다.

- DMA read: device가 host memory의 data를 읽는다.
- DMA write: device가 host memory에 data를 쓴다.

NVMe read command에서는 SSD가 NAND에서 읽은 payload를 host buffer에 DMA write한다.

NVMe write command에서는 SSD가 host buffer의 payload를 DMA read한다.

## 3.2 DMA의 핵심 질문

DMA 코드를 읽을 때는 다음 질문을 확인해야 한다.

1. Buffer를 누가 할당했는가?
2. CPU가 사용하는 address와 device가 사용하는 address는 무엇인가?
3. Mapping의 방향은 무엇인가?
4. 현재 buffer의 소유권은 CPU와 device 중 누구에게 있는가?
5. Device에 buffer가 보이도록 어떤 synchronization을 수행하는가?
6. Completion 후 언제 unmap하거나 재사용할 수 있는가?
7. Timeout과 reset 시 outstanding buffer를 어떻게 회수하는가?

# 4. Interrupt

Interrupt는 device가 CPU에 처리할 event가 있음을 알리는 방법이다.

```text
Device가 작업 완료
    ↓
Completion 정보 기록
    ↓
Interrupt 발생
    ↓
CPU가 현재 실행 흐름 전환
    ↓
Interrupt handler가 원인 확인
    ↓
Completion 처리 또는 후속 작업 예약
```

CPU는 I/O가 끝날 때까지 같은 위치에서 기다릴 필요가 없다.

다른 작업을 실행하거나 idle 상태에 들어갔다가 interrupt를 받고 완료를 처리할 수 있다.

## 4.1 Interrupt의 장점

- Event가 드물 때 CPU를 효율적으로 사용할 수 있다.
- CPU가 busy loop를 수행하지 않아도 된다.
- Device의 비동기 event를 처리하기 적합하다.

## 4.2 Interrupt의 비용

- 현재 실행 흐름을 전환해야 한다.
- Handler와 후속 처리 비용이 발생한다.
- Cache와 branch predictor locality가 나빠질 수 있다.
- Event가 너무 많으면 interrupt storm이 발생할 수 있다.

## 4.3 Interrupt Coalescing

Device가 completion마다 interrupt를 발생시키지 않고 여러 completion을 모아 한 번에 알릴 수 있다.

```text
Completion 1 ┐
Completion 2 ├─ 하나의 Interrupt
Completion 3 ┘
```

Interrupt 수가 감소하므로 CPU overhead와 처리량에는 유리할 수 있다.

반면 첫 번째 completion이 다음 completion을 기다리게 되면 latency가 증가할 수 있다.

# 5. MSI와 MSI-X

전통적인 interrupt pin 대신 PCIe device는 memory write 형태의 message로 interrupt를 전달할 수 있다.

이를 MSI(Message Signaled Interrupt)라고 한다.

MSI-X는 더 많은 interrupt vector와 독립적인 설정을 제공하여 multi-queue device에 적합하다.

```text
NVMe Queue 0 → MSI-X Vector 0 → CPU 0
NVMe Queue 1 → MSI-X Vector 1 → CPU 1
NVMe Queue 2 → MSI-X Vector 2 → CPU 2
```

Queue와 CPU를 적절히 연결하면 여러 core가 하나의 interrupt와 lock을 두고 경쟁하는 문제를 줄일 수 있다.

하지만 queue, interrupt와 application thread가 서로 다른 NUMA node에 배치되면 remote memory access와 cache 이동이 증가할 수 있다.

# 6. Polling

Polling은 CPU가 status register 또는 host memory의 completion queue를 반복해서 확인하는 방식이다.

```c
while (!completion_ready()) {
    // 계속 확인
}
```

실제 구현은 단순 loop보다 batching, pause instruction, timeout과 scheduler 연동 등을 포함할 수 있다.

## 6.1 Polling의 장점

- Interrupt 전달과 context 전환 지연을 줄일 수 있다.
- 지속적으로 event가 발생하는 환경에서 한 번에 여러 completion을 처리할 수 있다.
- 낮고 예측 가능한 latency가 중요한 상황에 유리할 수 있다.

## 6.2 Polling의 비용

- 완료되지 않은 시간에도 CPU cycle을 소비한다.
- 전력 소비가 증가한다.
- 공유 memory를 지나치게 읽으면 cache와 interconnect traffic이 증가한다.
- Polling thread에 전용 CPU를 할당하면 다른 작업에 사용할 core가 줄어든다.

# 7. Interrupt와 Polling의 선택

| 기준 | Interrupt | Polling |
| --- | --- | --- |
| 낮은 요청 빈도 | 유리 | CPU 낭비가 큼 |
| 지속적인 고부하 | Interrupt overhead 증가 | Batching에 유리할 수 있음 |
| 완료 latency | 전달 및 scheduling 지연 존재 | 전용 CPU 사용 시 낮출 수 있음 |
| CPU 사용량 | Event가 적으면 낮음 | 대기 중에도 높음 |
| 전력 | 상대적으로 유리 | 상대적으로 불리 |
| 구현 고려사항 | Affinity, coalescing | Timeout, fairness, core 할당 |

둘 중 하나만 고집할 필요는 없다.

낮은 부하에서는 interrupt를 사용하고 높은 부하에서는 polling으로 전환하는 hybrid 방식도 가능하다.

# 8. NVMe Read에서의 연결

NVMe read command 하나의 흐름을 살펴보자.

```text
1. Driver가 host memory에 SQ entry 작성
2. Driver가 MMIO doorbell write
3. Controller가 SQ entry를 DMA read
4. Controller가 FTL을 통해 NAND 위치 확인
5. NAND page read
6. Controller가 host buffer에 payload DMA write
7. Controller가 CQ entry를 DMA write
8. Controller가 MSI-X interrupt 발생
   또는 CPU가 CQ polling
9. Driver가 request 완료 처리
```

여기에서 각 메커니즘의 역할은 다음과 같다.

- MMIO doorbell: 새 command의 존재를 알린다.
- DMA: Command, payload와 completion을 memory에서 읽거나 쓴다.
- Interrupt/polling: CPU가 completion을 발견한다.

# 9. Memory Ordering과 Ownership

비동기 I/O에서는 buffer의 소유권 전환이 중요하다.

```text
CPU가 descriptor 작성 중
  → 소유자: CPU

Doorbell을 통해 device에 제출
  → 소유자: Device

Device가 completion 기록
  → 소유자: CPU로 반환
```

CPU는 device에 제출한 buffer를 completion 전에 수정하거나 해제하면 안 된다.

Device가 completion을 기록하기 전에 payload write가 완료되어야 하며, CPU는 completion을 확인한 뒤 올바른 ordering으로 payload를 읽어야 한다.

이 규칙은 다음 한 문장으로 정리할 수 있다.

> Buffer를 상대에게 넘기기 전에 내용을 완성하고, 소유권을 돌려받기 전에는 다시 사용하지 않는다.

# 정리

DMA와 interrupt는 서로 대체하는 기술이 아니다.

```text
DMA: Data 이동
Interrupt/Polling: 완료 발견
```

Driver는 buffer와 descriptor를 준비하고 device에 작업을 알린다.

Device는 DMA를 통해 command와 payload를 처리하고 completion을 기록한다.

CPU는 interrupt 또는 polling으로 completion을 발견한다.

이 전체 흐름을 이해하려면 address, memory ordering과 buffer ownership을 함께 봐야 한다.
