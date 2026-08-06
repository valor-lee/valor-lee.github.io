---
title: '[CPU] Memory Access와 Cache Hierarchy'
date: 2026-08-06 00:10:00 +09:00
categories: [computer, CPU]
published: false
tags:
  [
    CPU,
    memory,
    cache hierarchy,
    cache line,
    locality
  ]
---

# 개요

CPU가 다음 instruction을 실행한다고 생각해보자.

```c
value = array[index];
```

소스 코드에는 memory read 한 번만 보인다.

실제로는 virtual address 변환, cache tag 검색, cache miss 처리와 DRAM 접근이 필요할 수 있다.

```text
Load instruction
  → Virtual address 생성
  → Address translation
  → L1 cache 검색
  → L2 cache 검색
  → Last-level cache 검색
  → Memory controller
  → DRAM
```

모든 단계가 항상 실행되는 것은 아니다.

원하는 data가 가까운 cache에 있으면 아래 단계로 내려가지 않는다.

이 글에서는 CPU와 memory의 속도 차이를 cache hierarchy가 어떻게 줄이는지 살펴본다.

# 1. CPU와 Memory의 역할

CPU core는 instruction을 가져오고 해석하고 실행한다.

Memory는 실행할 instruction과 program이 사용하는 data를 저장한다.

```text
CPU
  ├─ Instruction 실행
  ├─ Register 연산
  └─ Load/Store 요청

Memory
  ├─ Instruction 저장
  └─ Data 저장
```

CPU의 일반적인 arithmetic instruction은 register를 대상으로 동작한다.

Memory의 값을 계산하려면 먼저 load로 register에 가져오고, 결과를 memory에 남기려면 store를 수행한다.

```text
Memory → Load → Register → Operation → Register → Store → Memory
```

# 2. Register

Register는 CPU core 내부에서 instruction이 직접 사용하는 작은 저장 공간이다.

일반적으로 cache나 DRAM보다 빠르지만 개수가 제한되어 있다.

Compiler는 program의 변수와 중간 결과를 가능한 한 register에 배치한다.

Register가 부족하면 값을 stack memory 등에 잠시 저장하는 spill이 발생할 수 있다.

# 3. CPU와 DRAM의 속도 차이

CPU pipeline은 많은 instruction을 동시에 진행할 수 있다.

반면 DRAM 접근은 CPU instruction 하나를 실행하는 것보다 훨씬 긴 시간이 걸릴 수 있다.

CPU가 모든 load와 store마다 DRAM을 기다리면 pipeline이 자주 멈춘다.

이를 memory wall 문제라고 부른다.

Cache hierarchy는 자주 사용할 가능성이 높은 data를 CPU 가까이에 복사해 이 latency를 숨긴다.

# 4. Locality

Cache는 program의 memory access에 locality가 존재한다는 점을 이용한다.

## 4.1 Temporal Locality

최근 접근한 data를 다시 접근할 가능성이 높다는 성질이다.

```c
for (int i = 0; i < count; ++i) {
    total += frequently_used_value;
}
```

`frequently_used_value`는 반복해서 사용되므로 cache에 남겨두는 것이 유리하다.

## 4.2 Spatial Locality

접근한 주소 주변의 data를 곧 사용할 가능성이 높다는 성질이다.

```c
for (int i = 0; i < count; ++i) {
    total += array[i];
}
```

배열을 순서대로 읽으면 인접한 data를 계속 사용한다.

Cache는 요청한 byte 하나가 아니라 주변 data를 포함하는 cache line 단위로 가져온다.

# 5. Cache Hierarchy

현대 CPU는 속도와 용량이 다른 여러 cache level을 사용한다.

```text
CPU Core
  │
  ├─ L1 Instruction Cache
  ├─ L1 Data Cache
  │
  ▼
L2 Cache
  │
  ▼
Last-Level Cache
  │
  ▼
Memory Controller
  │
  ▼
DRAM
```

일반적인 성질은 다음과 같다.

| 계층 | 위치와 성질 | 목적 |
| --- | --- | --- |
| Register | Core 내부, 가장 작음 | Instruction의 직접 operand |
| L1 cache | Core에 가장 가까움 | 매우 낮은 latency |
| L2 cache | L1보다 크고 느림 | L1 miss 흡수 |
| Last-level cache | 여러 core가 공유할 수 있음 | DRAM 접근 감소 |
| DRAM | 크지만 cache보다 느림 | Program의 main memory |

정확한 cache 크기, 공유 방식과 latency는 CPU microarchitecture마다 다르다.

Software가 특정 숫자를 보편적인 규칙처럼 가정하면 안 된다.

# 6. Instruction Cache와 Data Cache

L1 cache는 instruction과 data를 별도로 관리할 수 있다.

- Instruction cache: CPU가 실행할 instruction 저장
- Data cache: Load와 store가 사용하는 data 저장

이 구조를 이용하면 instruction fetch와 data access를 병렬로 처리하기 쉽다.

아래 단계에서는 instruction과 data가 통합된 cache를 사용할 수 있다.

# 7. Cache Line

Cache가 memory와 data를 주고받는 기본 단위를 cache line이라고 한다.

예를 들어 cache line이 64 byte라면 `int` 하나를 읽더라도 주변 data가 함께 들어올 수 있다.

```text
Memory
Address 0x1000 ───────────────── Address 0x103F
          하나의 64-byte cache line
```

Cache line은 다음 현상을 설명하는 기준이다.

- Spatial locality
- Cache miss와 line fill
- Cache coherence
- False sharing
- Alignment에 따른 line crossing

# 8. Cache Hit와 Cache Miss

CPU가 요청한 address의 cache line이 cache에 있으면 cache hit다.

없으면 cache miss다.

```text
L1 lookup
  ├─ Hit  → Data 반환
  └─ Miss → L2 lookup
               ├─ Hit  → L1에 채운 뒤 반환
               └─ Miss → 다음 cache 또는 DRAM 접근
```

Cache miss는 원인에 따라 다음처럼 분류할 수 있다.

## 8.1 Compulsory Miss

해당 data에 처음 접근해 cache에 존재하지 않는 경우다.

## 8.2 Capacity Miss

Program의 working set이 cache 용량보다 커서 필요한 line이 밀려난 경우다.

## 8.3 Conflict Miss

Cache에 빈 공간이 있더라도 여러 address가 같은 cache set에 mapping되어 서로 밀어내는 경우다.

Multi-core에서는 다른 core의 write로 line이 invalidate되어 발생하는 coherence miss도 고려해야 한다.

# 9. Cache의 기본 구조

Cache entry에는 data뿐 아니라 어떤 memory address의 data인지 나타내는 tag와 상태 정보가 필요하다.

Address는 개념적으로 다음 영역으로 나뉜다.

```text
+-----------+-----------+--------+
| Tag       | Set Index | Offset |
+-----------+-----------+--------+
```

- Offset: cache line 안에서 원하는 byte 위치
- Set index: 어느 cache set을 검색할지 결정
- Tag: 해당 entry가 원하는 memory block인지 확인

# 10. Direct-Mapped와 Set-Associative Cache

Direct-mapped cache에서는 memory block이 들어갈 entry가 하나로 정해진다.

구조는 단순하지만 같은 entry에 mapping되는 block끼리 자주 충돌할 수 있다.

Set-associative cache에서는 하나의 set에 여러 way가 있다.

```text
Set 0: Way 0 | Way 1 | Way 2 | Way 3
Set 1: Way 0 | Way 1 | Way 2 | Way 3
```

같은 set에 mapping되더라도 여러 block을 함께 보관할 수 있어 conflict miss를 줄인다.

대신 여러 tag를 비교하고 replacement 대상을 선택해야 한다.

# 11. Write Policy

CPU의 store를 아래 cache와 memory에 언제 반영할지에 따라 정책이 나뉜다.

## 11.1 Write-Through

Cache를 변경할 때 아래 계층에도 즉시 write를 전달한다.

구조는 비교적 단순하지만 write traffic이 많아질 수 있다.

## 11.2 Write-Back

먼저 cache line을 변경하고 dirty 상태로 표시한다.

해당 line이 eviction될 때 아래 계층에 write back할 수 있다.

Write traffic을 줄일 수 있지만 dirty line 관리가 필요하다.

## 11.3 Write-Allocate와 No-Write-Allocate

Store miss가 발생했을 때 line을 cache에 가져올지 결정하는 정책이다.

- Write-allocate: line을 가져온 뒤 cache에서 수정
- No-write-allocate: cache에 가져오지 않고 아래 계층으로 write 전달

실제 조합은 cache level과 memory type에 따라 다를 수 있다.

# 12. Cache Replacement

Cache set이 가득 찬 상태에서 새 line을 넣으려면 기존 line을 선택해 내보내야 한다.

이 결정을 replacement policy가 수행한다.

정확한 LRU는 hardware 비용이 크기 때문에 실제 CPU는 pseudo-LRU나 다른 근사 정책을 사용할 수 있다.

Software는 특정 replacement algorithm을 가정하기보다 working set과 access pattern을 관리해야 한다.

# 13. Hardware Prefetcher

CPU는 연속적이거나 반복적인 access pattern을 감지해 앞으로 필요할 data를 미리 cache에 가져올 수 있다.

Sequential array traversal이 빠른 이유에는 spatial locality뿐 아니라 hardware prefetch도 포함될 수 있다.

예측이 맞으면 latency를 숨기지만 불필요한 line을 가져오면 bandwidth와 cache 공간을 낭비할 수 있다.

# 14. Load가 처리되는 흐름

Load instruction의 전체 흐름을 단순화하면 다음과 같다.

```text
1. CPU가 effective virtual address 계산
2. TLB를 이용해 physical address 확인
3. L1 data cache의 set과 tag 검색
4. Hit면 원하는 byte 반환
5. Miss면 아래 cache 또는 DRAM에 line 요청
6. 도착한 line을 cache에 채움
7. 원하는 data를 load에 전달
```

Out-of-order CPU는 독립적인 다른 instruction을 실행해 miss latency를 숨길 수 있다.

동시에 처리 가능한 miss 수에는 hardware resource의 한계가 있다.

# 15. Store가 처리되는 흐름

Store는 address와 data를 준비한 뒤 cache line의 쓰기 권한을 확보해야 한다.

다른 core가 같은 line을 가지고 있다면 cache coherence transaction이 필요할 수 있다.

CPU는 store buffer를 사용해 이 latency를 숨긴다.

```text
Store instruction
  → Address와 data 계산
  → Store buffer에 보관
  → Cache line ownership 획득
  → L1 cache 변경
```

Store instruction이 retire된 시점과 다른 core가 새 값을 관찰하는 시점, DRAM에 값이 기록되는 시점은 서로 다를 수 있다.

# 16. Multi-Core와 Cache Coherence

각 core가 동일한 memory의 cache copy를 가질 수 있다.

한 core가 값을 변경할 때 다른 core가 오래된 값을 계속 사용하면 문제가 된다.

Cache coherence protocol은 같은 cache line의 사본과 write ownership을 관리한다.

```text
Core A L1 ─┐
           ├─ Coherence ─ Shared Cache / Memory
Core B L1 ─┘
```

Cache coherence는 주로 같은 memory location의 값에 대한 일관성을 다룬다.

여러 memory access가 어떤 순서로 관찰되는지는 memory ordering의 문제다.

# 17. False Sharing

두 thread가 서로 다른 변수를 변경해도 변수가 같은 cache line에 있으면 coherence traffic이 발생할 수 있다.

```text
하나의 cache line
+----------------+----------------+
| Core A counter | Core B counter |
+----------------+----------------+
```

논리적으로 공유하지 않는 data가 cache line이라는 hardware 단위에서는 공유되는 현상을 false sharing이라고 한다.

Data structure에 padding이나 alignment를 적용해 자주 변경하는 field를 다른 line으로 분리할 수 있다.

다만 cache line 크기와 data locality, memory 사용량을 함께 고려해야 한다.

# 18. 성능을 분석할 때 확인할 것

Cache 성능은 단순히 “cache가 크면 빠르다”로 설명할 수 없다.

다음 항목을 함께 확인해야 한다.

- Working set 크기
- Sequential 또는 random access
- Read/write 비율
- Cache line utilization
- Data structure layout
- Thread 간 data sharing
- False sharing
- NUMA 배치
- TLB miss
- Memory bandwidth

# 19. 자주 생기는 오해

## 19.1 Cache Hit면 한 cycle이다

Cache level, access type, dependency와 CPU 구조에 따라 latency가 다르다.

## 19.2 Cache는 Software가 직접 채운다

일반적인 cacheable memory의 cache fill과 replacement는 hardware가 관리한다.

Software는 access pattern, prefetch instruction과 cache policy를 통해 간접적으로 영향을 줄 수 있다.

## 19.3 Cache Coherence가 Thread Synchronization을 해결한다

Coherence는 cache copy의 일관성을 위한 기반이다.

Program의 data race를 없애거나 필요한 memory ordering을 자동으로 표현하지 않는다.

언어의 atomic과 lock 같은 synchronization이 별도로 필요하다.

## 19.4 Cache의 최종 목적지는 항상 DRAM이다

일반 cacheable memory에서는 보통 DRAM이 backing store다.

하지만 MMIO와 persistent memory처럼 memory type과 목적지가 다른 access도 있으므로 address space의 모든 영역을 일반 DRAM처럼 취급하면 안 된다.

# 정리

CPU는 register에서 연산하고 load와 store를 통해 memory에 접근한다.

CPU와 DRAM의 속도 차이를 줄이기 위해 여러 단계의 cache hierarchy를 사용한다.

```text
Register
  → L1 Cache
  → L2 Cache
  → Last-Level Cache
  → DRAM
```

Cache는 locality를 이용하고 cache line 단위로 data를 관리한다.

Multi-core 환경에서는 cache line의 사본과 write ownership을 cache coherence protocol이 조정한다.

다음 글에서는 load가 사용하는 virtual address가 physical memory의 위치로 어떻게 변환되는지 살펴본다.
