---
title: '[Storage] Application의 I/O는 NAND까지 어떻게 전달되는가'
date: 2026-08-05 00:40:00 +09:00
categories: [computer, storage]
published: false
tags:
  [
    storage stack,
    NVMe,
    PCIe,
    FTL,
    NAND
  ]
---

# 개요

Application이 file을 읽으면 SSD가 즉시 NAND page를 읽는 것처럼 보일 수 있다.

실제로는 여러 software와 hardware 계층을 통과한다.

```text
Application / DBMS
        ↓
System Call
        ↓
File System / Page Cache
        ↓
Block Layer
        ↓
NVMe Driver
        ↓
PCIe
        ↓
NVMe Controller
        ↓
FTL
        ↓
NAND Flash
```

각 계층은 서로 다른 주소, queue와 완료 조건을 사용한다.

Storage 성능과 오류를 분석하려면 “SSD가 느리다”는 하나의 결론보다 어느 계층에서 시간이 소비되었는지 분리해야 한다.

# 1. Application과 System Call

Application은 `read()`, `write()`, `pread()`, `fsync()` 또는 `io_uring` 같은 interface로 OS에 I/O를 요청한다.

DBMS에서는 다음 동작이 storage I/O로 연결될 수 있다.

- Data page read
- Dirty page writeback
- WAL write
- Transaction commit의 `fsync`
- Checkpoint

Application의 요청 단위와 storage device의 처리 단위는 같지 않을 수 있다.

DBMS page, file system block, block request, NVMe command와 NAND page를 구분해야 한다.

# 2. File System과 Page Cache

일반적인 buffered I/O는 page cache를 거칠 수 있다.

Read 대상이 page cache에 있다면 device I/O 없이 memory에서 결과를 반환할 수 있다.

```text
Read Request
    ├─ Page Cache Hit → Memory에서 반환
    └─ Page Cache Miss → Block I/O 생성
```

Write도 application의 `write()`가 반환되는 시점과 data가 SSD에 영구적으로 기록되는 시점이 다를 수 있다.

따라서 latency를 분석할 때 다음을 구분해야 한다.

- Application이 system call을 완료한 시점
- Kernel이 dirty page를 device에 제출한 시점
- Device가 command를 완료한 시점
- Durability가 보장되는 시점

# 3. Block Layer

Block layer는 file system의 I/O를 block device request로 변환하고 driver에 전달한다.

주요 역할은 다음과 같다.

- Request queue 관리
- 인접 request의 merge
- I/O scheduling
- Queue depth와 backpressure 관리
- Multi-queue를 통한 CPU별 request 분산
- Completion을 상위 계층으로 전달

NVMe 같은 multi-queue device에서는 Linux `blk-mq`와 hardware queue의 mapping이 성능에 영향을 줄 수 있다.

# 4. NVMe Driver

NVMe driver는 block request를 NVMe command로 만든다.

```text
Block Request
    ↓
NVMe Command 생성
    ↓
Submission Queue Entry 작성
    ↓
Data buffer를 PRP 또는 SGL로 표현
    ↓
Doorbell Write
```

Submission Queue와 Completion Queue는 host memory에 존재한다.

Controller는 DMA를 이용해 command를 읽고 completion을 쓴다.

# 5. PCIe

PCIe는 host와 NVMe controller 사이의 통신 경로를 제공한다.

이 경로에서는 서로 다른 종류의 transaction이 사용된다.

- MMIO: Controller register와 doorbell 접근
- DMA read: Controller가 command 또는 write payload를 host memory에서 읽음
- DMA write: Controller가 read payload 또는 completion을 host memory에 기록
- MSI-X: Completion event를 CPU에 알림

PCIe link의 bandwidth만으로 SSD 성능이 결정되지는 않는다.

Queueing, controller scheduling, NAND parallelism과 workload 특성도 함께 작용한다.

# 6. NVMe Controller와 Firmware

Controller는 제출된 command를 해석하고 내부 resource에 배치한다.

```text
NVMe Command Fetch
    ↓
Command Parsing
    ↓
FTL Mapping
    ↓
NAND Scheduling
    ↓
Data Transfer
    ↓
Completion 생성
```

여러 command가 동시에 들어오면 firmware는 channel, die와 queue 상태를 고려하여 처리 순서를 정할 수 있다.

Host의 queue depth가 증가하면 device 내부 parallelism을 더 활용할 수 있지만 queueing delay도 증가할 수 있다.

# 7. FTL

Host는 LBA(Logical Block Address)로 storage에 접근한다.

NAND는 page program과 block erase라는 제약을 가진다.

FTL(Flash Translation Layer)은 host의 logical address를 NAND의 physical location으로 변환한다.

```text
Host LBA
    ↓
FTL Mapping Table
    ↓
Channel / Die / Block / Page
```

FTL은 단순 주소 변환 외에도 다음 작업을 수행한다.

- Out-of-place update
- Garbage collection
- Wear leveling
- Bad block 관리
- Over-provisioning 관리
- TRIM 처리
- Mapping recovery

# 8. NAND Flash

NAND는 일반적으로 page 단위로 read/program하고 block 단위로 erase한다.

```text
Read     : Page 단위
Program  : Page 단위
Erase    : Block 단위
```

이미 program된 위치를 DRAM처럼 바로 덮어쓸 수 없다.

새 위치에 data를 기록하고 기존 page를 invalid 처리한 뒤, 나중에 garbage collection으로 block을 정리한다.

이 과정에서 host가 요청한 write보다 실제 NAND write가 많아질 수 있다.

이를 write amplification이라고 한다.

# 9. NVMe Read의 전체 흐름

Page cache miss가 발생한 read를 예로 들면 다음과 같다.

```text
1. Application이 read 요청
2. File system이 block I/O 생성
3. Block layer가 NVMe driver에 request 전달
4. Driver가 SQ entry와 PRP/SGL 작성
5. Driver가 MMIO doorbell write
6. Controller가 SQ entry를 DMA read
7. FTL이 LBA를 NAND 위치로 mapping
8. NAND page read
9. Controller가 host buffer에 data를 DMA write
10. Controller가 CQ entry를 DMA write
11. MSI-X interrupt 또는 polling으로 completion 확인
12. Driver와 block layer가 request 완료
13. Application에 data 반환
```

# 10. NVMe Write의 전체 흐름

Write에서는 data 방향과 durability 조건을 주의해야 한다.

```text
1. Application 또는 writeback이 write request 생성
2. Driver가 SQ entry와 data buffer 정보 작성
3. Controller가 host buffer의 data를 DMA read
4. Firmware가 logical address의 새 physical location 결정
5. NAND에 data program
6. Mapping과 metadata 갱신
7. Controller가 completion 기록
8. Driver가 request 완료
```

Device cache, FUA, flush와 power-loss protection에 따라 command completion이 의미하는 durability 수준은 달라질 수 있다.

“Write가 완료되었다”는 표현을 사용할 때 어느 계층의 완료인지 명확히 해야 한다.

# 11. 주소의 종류

전체 경로에는 여러 address space가 등장한다.

| 주소 | 사용하는 주체 | 의미 |
| --- | --- | --- |
| Application virtual address | Process | Application buffer 위치 |
| Kernel virtual address | Kernel | Kernel이 접근하는 mapping |
| CPU physical address | CPU/Memory system | Physical memory 위치 |
| DMA address | Device/IOMMU | Device가 DMA에 사용하는 주소 |
| LBA | Host/Storage protocol | Logical block 위치 |
| NAND physical location | Controller/FTL | Channel, die, block, page 위치 |

이 주소들은 같은 숫자일 수도 있지만 같은 개념은 아니다.

특히 DMA address를 CPU physical address와 항상 같다고 가정하면 IOMMU가 있는 환경을 이해할 수 없다.

# 12. 성능 분석 관점

End-to-end latency는 여러 구간의 합으로 볼 수 있다.

```text
Application/System Call
  + File System/Page Cache
  + Block Queueing
  + Driver/PCIe
  + Controller Queueing
  + FTL/GC
  + NAND Media
  + Completion 전달
```

Queue depth를 늘리면 device parallelism을 활용해 throughput이 증가할 수 있다.

하지만 요청이 queue에서 대기하는 시간이 늘어 tail latency가 악화될 수도 있다.

따라서 다음 지표를 함께 확인해야 한다.

- IOPS
- Bandwidth
- Average latency
- p50, p95, p99, p99.9 latency
- Queue depth
- CPU 사용량
- Read/write ratio
- Sequential/random access pattern
- Garbage collection과 write amplification

# 13. 장애 분석 관점

오류도 계층별로 분리한다.

| 계층 | 확인할 문제 |
| --- | --- |
| Application/DBMS | 요청 pattern, timeout, durability 요구 |
| File system/Block | Queueing, writeback, scheduling |
| Driver | Descriptor, timeout, reset, buffer lifetime |
| PCIe | Link, transaction, AER, MSI-X |
| Controller/Firmware | Scheduling, command state, recovery |
| FTL/NAND | Mapping, GC, bad block, media error |

하나의 결과만으로 원인을 단정하지 않는다.

가설을 세우고 관찰 지점을 추가하면서 문제가 시작되는 경계를 좁혀야 한다.

# 정리

Storage I/O는 하나의 함수 호출이 아니라 여러 software와 hardware 계층을 통과하는 비동기 작업이다.

각 계층은 서로 다른 queue, address와 완료 조건을 사용한다.

```text
Application 요청
  → OS가 block request 생성
  → NVMe driver가 command 제출
  → PCIe를 통해 controller와 통신
  → FTL이 logical address 변환
  → NAND가 실제 data 처리
  → Completion이 반대 방향으로 전달
```

이 전체 경로를 먼저 이해하면 DMA, NVMe queue, FTL garbage collection과 tail latency를 서로 분리된 용어가 아니라 하나의 흐름으로 연결할 수 있다.
