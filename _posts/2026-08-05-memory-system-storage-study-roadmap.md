---
title: '[Memory System] 삼성전자 메모리사업부 지원을 위한 학습 로드맵'
date: 2026-08-05 00:10:00 +09:00
categories: [computer, storage]
published: false
tags:
  [
    memory system,
    device I/O,
    PCIe,
    NVMe,
    NAND,
    FTL
  ]
---

# 개요

삼성전자 메모리사업부의 다음 직무를 준비하기 위해 필요한 지식을 정리한다.

| 세부 트랙 | 현재 적합도 | 경력과의 연결점 |
| --- | ---: | --- |
| Firmware Simulator·Test Platform | 8/10 | 코드, 테스트, 자동화 및 재현 경험 |
| Storage 성능 분석·Modeling | 8/10 | DBMS 성능 최적화와 병목 분석 경험 |
| System Level 검증·호환성 분석 | 7/10 | Linux, 디버깅, 장애 재현 경험 |

목표는 용어를 따로 암기하는 것이 아니다.

하나의 I/O 요청이 application에서 출발해 NAND에 도달하고 다시 완료되는 과정을 계층별로 설명할 수 있어야 한다.

```text
Application / DBMS
        ↓
File system / Block layer
        ↓
NVMe driver
        ↓
PCIe: MMIO, DMA, MSI-X
        ↓
NVMe controller / Firmware
        ↓
FTL
        ↓
NAND Flash
```

# 1. 학습 방법

하나의 주제를 처음부터 깊게 파기보다 전체 구조의 이론을 먼저 같은 깊이로 학습한다.

그다음 동일한 I/O 경로를 코드, 측정, 장애 분석의 관점으로 반복해서 살펴본다.

```text
이론 지도
  → 동작 흐름
  → 실제 코드
  → 성능 측정
  → 장애 재현과 원인 분리
```

각 주제는 다음 질문에 답할 수 있을 때 다음 깊이로 넘어간다.

1. 이것은 무엇인가?
2. 왜 필요한가?
3. 전체 I/O 경로의 어디에 있는가?
4. 입력과 출력은 무엇인가?
5. 어떤 구성 요소와 상호작용하는가?
6. 정상 동작 순서는 어떻게 되는가?
7. 어떤 오류와 병목이 발생할 수 있는가?
8. 코드와 측정 결과에서는 어떻게 관찰할 수 있는가?

# 2. 이미 학습한 기반 지식

다음 주제는 처음부터 다시 공부하지 않고 기존 노트를 복습한 뒤 실험으로 확인한다.

- CPU와 memory
- Cache hierarchy
- Cache coherence
- Memory ordering
- Virtual memory와 TLB
- NUMA
- Atomic operation
- Memory barrier
- Endianness와 alignment

이 지식은 독립된 선행 과목이 아니라 device I/O를 이해하기 위한 기반이다.

예를 들어 descriptor를 memory에 쓴 뒤 doorbell을 울리는 과정에는 cache, memory ordering, barrier가 다시 등장한다.

IOMMU를 이해할 때는 virtual address와 address translation 지식이 필요하다.

Multi-queue NVMe의 CPU affinity를 분석할 때는 cache locality와 NUMA 지식이 필요하다.

# 3. 첫 번째 영역: CPU와 Device의 경계

먼저 CPU가 device를 어떻게 제어하는지 이해한다.

- Bus와 interconnect
- Device controller
- Device register
- MMIO
- Interrupt와 polling
- Doorbell

핵심 질문은 다음과 같다.

```text
CPU는 device에 어떻게 명령을 전달하는가?
Device는 완료 사실을 CPU에 어떻게 알리는가?
일반 memory와 device register는 무엇이 다른가?
```

# 4. 두 번째 영역: Data 이동

다음으로 큰 payload가 CPU와 device 사이에서 이동하는 구조를 학습한다.

- DMA
- DMA mapping과 buffer lifetime
- CPU virtual address, physical address, device-visible address
- Coherent DMA와 non-coherent DMA
- IOMMU
- Descriptor와 ring buffer
- Scatter-Gather I/O
- Zero-copy
- Memory barrier와 ownership

이 영역의 중심 질문은 “현재 buffer와 descriptor의 소유자가 누구인가?”이다.

CPU와 device가 같은 memory를 비동기적으로 사용하기 때문에 주소 변환, cache visibility, ordering, buffer 재사용 시점을 함께 이해해야 한다.

# 5. 세 번째 영역: PCIe와 NVMe

Device I/O의 공통 원리를 storage protocol에 연결한다.

PCIe에서는 다음 항목을 학습한다.

- Root Complex, endpoint, switch
- Configuration space와 BAR
- Link와 lane
- Transaction과 completion
- MSI와 MSI-X
- AER와 오류 처리

NVMe에서는 다음 항목을 학습한다.

- Controller와 namespace
- Admin Queue와 I/O Queue
- Submission Queue와 Completion Queue
- Doorbell과 phase tag
- PRP와 SGL
- Multi-queue와 interrupt affinity
- Timeout, reset, error recovery

# 6. 네 번째 영역: NAND와 FTL

Host interface 아래에서 실제 media가 갖는 제약을 학습한다.

NAND의 핵심 주제는 다음과 같다.

- Cell, page, block
- SLC, MLC, TLC, QLC
- Page read/program과 block erase
- Erase-before-write
- ECC, bad block, retention, disturb
- Channel, die, plane parallelism

FTL의 핵심 주제는 다음과 같다.

- Logical address와 physical address mapping
- Out-of-place update
- Garbage collection
- Wear leveling
- Over-provisioning
- Write amplification
- TRIM
- Mapping recovery

# 7. 이론 이후의 실습 방향

전체 이론 지도를 완성한 뒤 다음 순서로 실습한다.

## 7.1 Descriptor Ring Simulator

Host thread가 descriptor를 생성하고 device thread가 소비하도록 구현한다.

- Producer/consumer index
- Ring wrap-around
- Queue full과 backpressure
- Descriptor ownership
- Memory ordering
- Polling과 notification 비교

## 7.2 NVMe 관찰과 성능 분석

Linux와 QEMU 또는 실제 NVMe 장비에서 다음 도구를 사용한다.

- `lspci`
- `nvme-cli`
- `fio`
- `iostat`
- `perf`
- Block tracepoint

Block size, queue depth, read/write ratio와 access pattern을 바꾸면서 IOPS, bandwidth, average latency와 tail latency를 분석한다.

## 7.3 FTL Simulator

간단한 page-level mapping FTL을 구현한다.

- Logical-to-physical mapping
- Free block 관리
- Garbage collection
- Over-provisioning
- Write amplification 측정
- Power loss와 consistency test

# 8. 직무별 최종 연결

## Firmware Simulator·Test Platform

정상 경로만 구현하는 것이 아니라 timeout, invalid descriptor, queue full, media error와 power loss를 주입한다.

실패한 test를 random seed와 event log로 다시 재현할 수 있어야 한다.

## Storage 성능 분석·Modeling

DBMS에서 경험한 page, WAL, `fsync`, buffer와 concurrency 문제를 storage queue와 FTL의 동작에 연결한다.

평균값만 보지 않고 p95, p99, p99.9 latency와 steady state를 함께 분석한다.

## System Level 검증·호환성 분석

오류를 다음 계층으로 나누어 조사한다.

```text
Application
  → File system / Block layer
  → Driver / PCIe
  → Controller / Firmware
  → FTL / NAND
```

각 실험에는 가설, 통제 변수, 재현 절차, 실제 결과, 증거와 다음 실험을 기록한다.

# 정리

학습의 기준은 읽은 문서의 수가 아니다.

Application에서 NAND까지의 경로를 그리고, 각 경계에서 전달되는 명령, 주소, 데이터, 소유권과 완료 정보를 설명할 수 있어야 한다.

전체 이론을 확립한 뒤 코드를 읽고 측정하면 개별 API와 수치가 어느 계층의 현상인지 판단할 수 있다.
