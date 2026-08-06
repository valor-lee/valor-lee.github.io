---
title: '[Memory] Virtual Memory와 TLB'
date: 2026-08-06 00:20:00 +09:00
categories: [computer, memory]
published: false
tags:
  [
    virtual memory,
    address translation,
    page table,
    TLB,
    page fault
  ]
---

# 개요

Application에서 pointer가 가리키는 주소는 일반적으로 physical memory의 위치가 아니다.

```c
int value = *pointer;
```

Process는 virtual address를 사용하고, CPU와 OS는 이를 physical address로 변환한다.

```text
Virtual Address
    ↓ Address Translation
Physical Address
    ↓ Cache / Memory
Data
```

Page table은 변환 관계를 저장하고 TLB는 자주 사용하는 변환 결과를 cache한다.

이 글에서는 virtual memory, page table, page fault와 TLB의 관계를 살펴본다.

# 1. Physical Address만 사용한다면

모든 process가 physical address를 직접 사용한다고 가정해보자.

다음 문제가 발생한다.

- Process가 다른 process의 memory를 쉽게 읽거나 변경할 수 있다.
- Program이 어느 physical 위치에 배치될지 미리 알아야 한다.
- 연속된 큰 physical memory를 확보하기 어렵다.
- 같은 library code를 여러 process가 효율적으로 공유하기 어렵다.
- Memory보다 큰 address space를 제공하기 어렵다.

Virtual memory는 process가 보는 address space와 실제 physical memory 배치를 분리한다.

# 2. Virtual Address Space

각 process는 자신만의 virtual address space를 가진다.

```text
Process A Virtual Address 0x1000
    → Physical Frame 10

Process B Virtual Address 0x1000
    → Physical Frame 42
```

두 process가 같은 virtual address를 사용하더라도 서로 다른 physical memory에 mapping될 수 있다.

반대로 shared memory처럼 서로 다른 virtual address가 같은 physical frame을 가리킬 수도 있다.

# 3. Page와 Frame

Virtual memory는 address space를 고정 크기의 page 단위로 관리한다.

Physical memory의 대응 단위를 page frame 또는 physical frame이라고 한다.

```text
Virtual Pages                     Physical Frames
+---------+                       +---------+
| Page 0  | --------------------> | Frame 7 |
+---------+                       +---------+
| Page 1  | -----------+          | Frame 8 |
+---------+            |          +---------+
| Page 2  | -----+     +--------> | Frame 9 |
+---------+      |                +---------+
                 +--------------> | Frame 3 |
                                  +---------+
```

Virtual address에서 page 내부 offset은 address translation 후에도 유지된다.

```text
Virtual Address
+---------------------+-------------+
| Virtual Page Number | Page Offset |
+---------------------+-------------+
            ↓ translation
Physical Address
+----------------------+-------------+
| Physical Frame Number| Page Offset |
+----------------------+-------------+
```

# 4. Page Table

Page table은 virtual page와 physical frame의 mapping을 저장한다.

Page table entry에는 physical frame 번호 외에도 다음 정보가 포함될 수 있다.

- Present 또는 valid 여부
- Read/write permission
- User/kernel permission
- Executable 여부
- Accessed 여부
- Dirty 여부
- Cache policy 관련 속성

CPU의 memory management unit은 현재 process의 page table을 기준으로 address를 변환하고 permission을 검사한다.

# 5. Multi-Level Page Table

큰 virtual address space의 모든 page에 entry를 미리 만들면 page table 자체가 너무 커진다.

Multi-level page table은 address space를 계층적으로 나누고 실제 사용하는 영역에 필요한 하위 table만 생성한다.

```text
Virtual Address
  ├─ Level 1 Index
  ├─ Level 2 Index
  ├─ Level 3 Index
  ├─ Level 4 Index
  └─ Page Offset
```

정확한 level 수와 address bit 구성은 architecture와 page size에 따라 다르다.

# 6. Address Translation 과정

TLB miss가 발생했다고 가정하면 load는 개념적으로 다음 과정을 거친다.

```text
1. CPU가 virtual address 생성
2. Virtual page number와 offset 분리
3. Page table walk 수행
4. Page table entry의 valid와 permission 확인
5. Physical frame number 획득
6. Physical address 구성
7. Cache hierarchy에서 data 검색
```

매 memory access마다 여러 단계의 page table을 읽으면 큰 비용이 발생한다.

이를 줄이기 위해 TLB를 사용한다.

# 7. TLB

TLB(Translation Lookaside Buffer)는 최근 사용한 virtual page에서 physical frame으로의 변환을 저장하는 cache다.

```text
Virtual Page Number
       ↓
      TLB
  ├─ Hit  → Physical Frame Number
  └─ Miss → Page Table Walk
                ↓
             TLB에 결과 저장
```

Cache가 data를 저장한다면 TLB는 address translation 결과를 저장한다.

둘은 서로 다른 역할을 한다.

# 8. TLB Hit와 TLB Miss

## 8.1 TLB Hit

필요한 translation이 TLB에 있으면 page table walk 없이 physical address를 얻을 수 있다.

## 8.2 TLB Miss

Translation이 TLB에 없으면 page table을 조회해야 한다.

Mapping이 유효하다면 page table walk 후 TLB를 채우고 instruction을 계속 실행한다.

TLB miss가 곧 page fault라는 뜻은 아니다.

```text
TLB Miss
  → Page Table Entry Valid
      → Translation을 TLB에 채움

TLB Miss
  → Page Table Entry Invalid 또는 특별한 처리 필요
      → Page Fault
```

# 9. Page Fault

CPU가 현재 page table mapping으로 처리할 수 없는 memory access를 만나면 exception을 발생시킨다.

OS의 page fault handler가 원인을 확인한다.

## 9.1 Demand Paging

Virtual address space는 예약되어 있지만 physical page가 아직 할당되지 않은 경우다.

OS가 physical frame을 할당하고 page table을 갱신한 뒤 instruction을 다시 실행할 수 있다.

## 9.2 File-Backed Page

Executable, shared library 또는 memory-mapped file의 page가 아직 memory에 없을 수 있다.

OS가 storage에서 page를 읽어온다.

## 9.3 Copy-on-Write

여러 process가 read-only로 공유하던 page를 한 process가 수정하려 할 수 있다.

OS가 새 physical page를 만들고 내용을 복사한 뒤 해당 process의 mapping을 변경한다.

## 9.4 Invalid Access

Mapping되지 않은 주소 또는 permission을 위반한 접근이라면 OS가 process에 segmentation fault 같은 오류를 전달할 수 있다.

# 10. Minor Fault와 Major Fault

일반적인 OS 통계에서는 page fault를 다음처럼 구분할 수 있다.

- Minor fault: 필요한 page가 memory에 있어 storage I/O 없이 mapping을 구성할 수 있음
- Major fault: 필요한 page를 storage에서 읽어야 함

Major fault는 storage latency를 포함할 수 있어 훨씬 큰 비용이 발생한다.

하지만 정확한 분류와 명칭은 OS 구현과 관찰 도구의 정의를 확인해야 한다.

# 11. Page Size와 TLB Reach

TLB가 cover할 수 있는 memory 범위를 TLB reach라고 생각할 수 있다.

```text
TLB Reach ≈ TLB Entry 수 × Page Size
```

예를 들어 같은 TLB entry 수에서 page size가 크면 더 넓은 memory 영역의 translation을 저장할 수 있다.

큰 working set을 random하게 접근하면 TLB entry가 자주 교체되어 TLB miss가 증가할 수 있다.

# 12. Huge Page

일반 page보다 큰 page를 사용하면 같은 memory 범위에 필요한 TLB entry 수를 줄일 수 있다.

장점:

- TLB miss 감소 가능
- Page table 크기와 walk 횟수 감소 가능

비용과 주의점:

- 작은 object에 사용하면 internal fragmentation 증가
- 큰 연속 physical memory 확보가 어려울 수 있음
- Allocation과 compaction 비용 발생 가능
- NUMA placement가 잘못되면 더 큰 범위가 잘못된 node에 배치될 수 있음
- 모든 workload에서 성능이 향상되는 것은 아님

# 13. Context Switch와 TLB

Process가 바뀌면 같은 virtual address가 다른 physical frame을 의미할 수 있다.

따라서 TLB entry가 어느 address space의 translation인지 구분해야 한다.

Architecture는 address-space identifier 같은 tag를 사용해 여러 process의 entry를 구분할 수 있다.

그렇지 않거나 mapping이 변경된 경우 TLB entry를 invalidate해야 한다.

# 14. TLB Shootdown

한 process의 page table을 여러 CPU core가 사용하고 있을 때 mapping을 변경하면 다른 core의 오래된 TLB entry도 무효화해야 한다.

```text
Core A가 Page Table 변경
  → Core B와 Core C에 invalidate 요청
  → 각 Core가 해당 TLB entry 제거
  → 완료 동기화
```

이 과정을 TLB shootdown이라고 한다.

Memory mapping을 자주 변경하는 workload에서는 core 간 coordination 비용이 발생할 수 있다.

# 15. Cache와 TLB의 관계

CPU가 cache를 검색하려면 address의 일부 또는 전체가 필요하다.

Address translation과 cache lookup을 어떤 순서와 방식으로 겹치는지는 cache 설계에 따라 다르다.

Software 관점에서는 다음 두 working set을 구분하는 것이 중요하다.

- Cache working set: 자주 사용하는 data의 cache line 집합
- TLB working set: 자주 사용하는 virtual page의 translation 집합

Data가 cache에 잘 맞더라도 많은 page에 흩어져 있으면 TLB miss가 발생할 수 있다.

# 16. Device I/O와 Virtual Memory

Application은 virtual address의 buffer를 사용하지만 device가 process의 일반 virtual address를 그대로 이해하는 것은 아니다.

Driver와 OS는 buffer를 DMA에 사용할 수 있도록 mapping하고 device-visible address를 준비한다.

```text
Application Virtual Address
    ↓ OS가 page 확인 및 고정
Physical Pages
    ↓ DMA Mapping / IOMMU
Device-visible DMA Address
```

IOMMU는 device의 DMA address를 physical memory로 translation하고 접근 권한을 제한할 수 있다.

CPU의 page table과 IOMMU의 translation table은 목적과 사용 주체가 다르다.

# 17. 성능을 분석할 때 확인할 것

- Working set의 전체 크기
- 접근하는 page 수
- Page size
- Sequential 또는 random access
- Page fault 수
- Minor/major fault 구분
- TLB miss
- Page table walk 비용
- Memory mapping 변경 빈도
- TLB shootdown
- NUMA placement

# 18. 자주 생기는 오해

## 18.1 Virtual Memory는 Memory보다 큰 공간을 쓰기 위한 기능뿐이다

Virtual memory는 isolation, permission, flexible placement와 sharing도 제공한다.

## 18.2 TLB Miss는 Page Fault다

Page table에 valid mapping이 있다면 TLB만 채우고 계속 실행할 수 있다.

## 18.3 Page Fault는 항상 오류다

Demand paging과 copy-on-write를 처리하기 위한 정상적인 page fault도 있다.

## 18.4 Virtual Address와 Physical Address는 항상 일대일이다

서로 다른 virtual page가 같은 physical frame을 공유할 수 있고, mapping은 시간에 따라 변경될 수 있다.

## 18.5 Huge Page는 항상 빠르다

TLB miss를 줄일 수 있지만 memory 낭비, allocation, NUMA와 workload 특성을 함께 고려해야 한다.

# 정리

Virtual memory는 process의 address space와 physical memory 배치를 분리한다.

Page table은 virtual page를 physical frame에 mapping하고 permission을 관리한다.

TLB는 자주 사용하는 address translation을 cache해 page table walk 비용을 줄인다.

```text
Virtual Address
  → TLB
      ├─ Hit  → Physical Address
      └─ Miss → Page Table Walk
                    ├─ Valid → TLB Fill
                    └─ 처리 필요 → Page Fault
```

다음 글에서는 memory에 저장된 여러 byte를 어떤 순서로 해석하는지, data가 특정 address 경계에 배치되는 이유를 살펴본다.
