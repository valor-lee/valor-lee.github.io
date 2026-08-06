---
title: '[Memory System] CPU부터 NVMe·NAND까지 읽는 순서'
date: 2026-08-06 00:50:00 +09:00
categories: [computer, memory]
published: false
tags:
  [
    memory system,
    CPU,
    atomic,
    device I/O,
    NVMe,
    study guide
  ]
---

# 개요

CPU, cache, atomic, DMA, NVMe와 NAND는 서로 독립적인 주제가 아니다.

CPU가 memory를 읽고 쓰는 원리에서 시작해 여러 core의 동기화, CPU와 device의 비동기 통신, storage media의 동작으로 확장된다.

```text
CPU와 Memory
    ↓
Cache와 Address Translation
    ↓
Multi-Core 동기화
    ↓
CPU와 Device I/O
    ↓
PCIe와 NVMe
    ↓
FTL과 NAND
```

이 글은 블로그에 정리한 내용을 의존 관계에 따라 읽기 위한 안내서다.

# 1. 전체 흐름 먼저 보기

가장 먼저 다음 글을 읽어 전체 학습 범위와 application에서 NAND까지 이어지는 경로를 확인한다.

1. `[Memory System] 삼성전자 메모리사업부 지원을 위한 학습 로드맵`
2. [[Storage] Application의 I/O는 NAND까지 어떻게 전달되는가](/posts/storage-io-end-to-end/)

로드맵은 `_posts/2026-08-05-memory-system-storage-study-roadmap.md`에 있으며 현재 `published: false`로 설정되어 있다. 공개 전에도 source 문서로 먼저 읽을 수 있다.

첫 단계에서는 PCIe, DMA, FTL의 세부 내용을 모두 이해할 필요는 없다.

각 용어가 전체 경로의 어느 위치에 있는지만 확인한다.

# 2. CPU와 Memory의 기본 구조

## 2.1 Memory Access와 Cache

3. [[CPU] Memory Access와 Cache Hierarchy](/posts/cpu-memory-cache-hierarchy/)

다음 내용을 확인한다.

- CPU가 register, load와 store를 사용하는 방식
- CPU와 DRAM의 속도 차이
- Locality
- L1/L2/last-level cache
- Cache line, hit와 miss
- Write policy
- Multi-core cache의 기본 문제

완료 기준:

> Load가 L1 cache에서 miss한 뒤 아래 cache와 DRAM을 거쳐 data를 가져오는 과정을 설명할 수 있다.

## 2.2 Virtual Memory와 TLB

4. [[Memory] Virtual Memory와 TLB](/posts/virtual-memory-tlb/)

다음 개념을 연결한다.

```text
Virtual Address
  → TLB
  → Page Table
  → Physical Address
  → Cache / Memory
```

완료 기준:

> TLB miss와 page fault의 차이를 설명하고, application virtual address와 device DMA address가 다른 이유를 말할 수 있다.

## 2.3 Data Layout

5. [[Computer Architecture] Endianness와 Alignment](/posts/endianness-alignment/)

Binary data, C structure, MMIO register와 DMA descriptor를 읽기 위한 기반이다.

완료 기준:

> Endianness와 alignment를 구분하고 structure를 그대로 protocol이나 file에 저장하면 위험한 이유를 설명할 수 있다.

## 2.4 Memory Locality

6. [[Memory] NUMA 구조와 Local·Remote Memory](/posts/numa-memory-architecture/)

완료 기준:

> CPU affinity만으로 NUMA locality가 보장되지 않는 이유와 NVMe queue·interrupt·DMA buffer의 위치가 중요한 이유를 설명할 수 있다.

# 3. Atomic Operation과 Cache Coherence

기본 memory 구조를 이해한 뒤 여러 thread가 같은 data를 사용할 때의 문제로 이동한다.

## 3.1 Atomic이 필요한 이유

7. [[C언어] Atomic Operation 1. 왜 Atomic이 필요한가](/posts/c-atomic-operation-1/)

다음 순서로 이해한다.

```text
counter++
  → 여러 단계의 read-modify-write
  → Race condition과 lost update
  → C의 data race
  → Atomic operation
```

## 3.2 C Atomic API

8. [[C언어] Atomic Operation 2. stdatomic.h 사용법](/posts/c-atomic-operation-2/)

`atomic_load`, `atomic_store`, `atomic_fetch_add`, `atomic_exchange`와 compare-exchange의 기본 사용법을 확인한다.

이 시점에는 모든 memory order를 외우지 않는다.

## 3.3 Hardware Atomic과 Cache Coherence

9. [[C언어] Atomic Operation 3. Atomic은 실제로 어떻게 동작하는가](/posts/c-atomic-operation-3/)

C의 atomic 보장이 x86-64와 ARM64 instruction, cache line ownership과 coherence protocol로 연결되는 과정을 읽는다.

완료 기준:

> Atomicity와 “CPU instruction 하나”가 같은 뜻이 아닌 이유, 서로 다른 변수가 false sharing을 일으키는 이유를 설명할 수 있다.

# 4. Store Buffer와 Memory Ordering

## 4.1 Store Buffer

10. [[CPU] Store Buffer는 어떻게 동작하는가](/posts/cpu-store-buffer/)

다음 시점을 구분한다.

```text
Store instruction 실행
  → Retire
  → Store buffer drain
  → 다른 core에서 관찰 가능
  → Dirty line의 DRAM write-back
```

이 글에서 store-to-load forwarding, store buffering litmus test와 barrier의 hardware 배경을 먼저 이해한다.

## 4.2 C Memory Ordering

11. [[C언어] Atomic Operation 4. Memory Order 이해하기](/posts/c-atomic-operation-4/)

다음 개념을 순서대로 읽는다.

- Atomicity와 ordering의 차이
- Happens-before
- Relaxed
- Release와 acquire
- Acquire-release
- Sequential consistency
- Atomic fence

완료 기준:

> Producer가 data를 쓴 뒤 release store로 flag를 publish하고 consumer가 acquire load로 data를 안전하게 읽는 이유를 설명할 수 있다.

## 4.3 Atomic과 Lock

12. [[C언어] Atomic Operation 5. Atomic과 Mutex 비교하기](/posts/c-atomic-operation-5/)

Atomic, mutex와 spinlock은 단순한 속도 비교 대상이 아니다.

보호할 불변식의 범위, critical section 길이, 경합과 scheduler 동작을 기준으로 선택하는 방법을 확인한다.

## 4.4 특수한 Cache 접근

13. [What Every Programmer Should Know About Memory - Chapter6.1 Bypassing the Cache](/posts/BypassingTheCache/)

일반 cacheable memory access를 이해한 뒤 non-temporal load/store, write-combining과 barrier를 살펴본다.

이 내용은 특수한 최적화이므로 일반 memory access보다 먼저 적용하지 않는다.

# 5. CPU와 Device의 통신

CPU 내부와 multi-core 동기화를 이해한 뒤 device I/O로 이동한다.

## 5.1 Bus, Register와 MMIO

14. [[Device I/O] CPU는 Device와 어떻게 통신하는가](/posts/cpu-device-communication/)

완료 기준:

> Device register와 일반 RAM의 차이, MMIO access에 `volatile`만으로 충분하지 않은 이유, descriptor를 완성한 뒤 doorbell을 써야 하는 이유를 설명할 수 있다.

## 5.2 Interrupt, Polling과 DMA

15. [[Device I/O] Interrupt와 Polling, DMA는 어떻게 연결되는가](/posts/interrupt-polling-dma/)

다음 두 질문을 구분한다.

```text
DMA: Payload를 누가 이동하는가?
Interrupt/Polling: CPU가 완료를 어떻게 발견하는가?
```

완료 기준:

> Driver가 buffer를 준비하고 device가 DMA를 수행한 뒤 interrupt 또는 polling으로 완료를 처리하는 lifecycle을 설명할 수 있다.

# 6. Storage Firmware와 성능 분석

16. `[Storage] Firmware Simulator·Test Platform과 성능 분석·Modeling 입문`

이 글에서는 NAND, FTL, NVMe queue, simulator와 성능 modeling을 직무 관점으로 연결한다.

이 글은 `_posts/2026-08-02-storage-firmware-simulator-performance-modeling.md`에 정리되어 있다. 현재 `published: false`로 설정되어 있으므로 블로그에서 공개하려면 내용을 최종 검토한 뒤 설정을 변경해야 한다.

# 7. 읽으면서 사용할 질문

모든 글을 읽은 뒤 다음 질문에 자신의 문장으로 답한다.

## CPU와 Memory

1. Cache line이 필요한 이유는 무엇인가?
2. Cache coherence와 memory ordering은 무엇이 다른가?
3. TLB miss와 page fault는 무엇이 다른가?
4. Huge page의 장점과 비용은 무엇인가?
5. NUMA에서 CPU, memory와 device locality를 함께 봐야 하는 이유는 무엇인가?

## Concurrency

6. `counter++`가 atomic하지 않은 이유는 무엇인가?
7. Atomicity와 visibility, ordering은 무엇이 다른가?
8. Release-acquire는 어떤 data를 안전하게 publish하는가?
9. Memory barrier는 cache 전체를 flush하는 명령인가?
10. Atomic 대신 mutex가 필요한 상황은 무엇인가?

## Device I/O와 Storage

11. MMIO register와 일반 memory는 무엇이 다른가?
12. DMA에서 CPU, driver와 device의 역할은 각각 무엇인가?
13. Descriptor write와 doorbell write 사이에 ordering이 필요한 이유는 무엇인가?
14. Interrupt와 polling의 trade-off는 무엇인가?
15. NVMe read command가 application에서 NAND까지 이동하고 완료되는 순서는 무엇인가?

# 8. 전체 완료 기준

다음 흐름을 보지 않고 직접 그릴 수 있으면 이론의 첫 번째 순회가 완료된 것이다.

```text
Application Virtual Address
  → Page Table / TLB
  → CPU Cache Hierarchy
  → Driver의 Descriptor
  → MMIO Doorbell
  → Device의 DMA
  → PCIe / NVMe Controller
  → FTL Mapping
  → NAND
  → DMA Completion
  → MSI-X Interrupt 또는 Polling
```

이후에는 같은 순서를 실제 driver 코드, simulator와 benchmark로 다시 확인한다.
