---
title: '[Memory] NUMA 구조와 Local·Remote Memory'
date: 2026-08-06 00:40:00 +09:00
categories: [computer, memory]
published: false
tags:
  [
    NUMA,
    memory locality,
    CPU affinity,
    memory bandwidth,
    PCIe
  ]
---

# 개요

Multi-core system에서 모든 CPU가 모든 memory에 같은 비용으로 접근한다고 생각하기 쉽다.

하지만 여러 CPU socket이나 memory node로 구성된 system에서는 memory의 위치에 따라 latency와 bandwidth가 달라질 수 있다.

이를 NUMA(Non-Uniform Memory Access)라고 한다.

```text
CPU가 가까운 Memory 접근  → Local Memory Access
CPU가 다른 Node의 Memory 접근 → Remote Memory Access
```

NUMA는 database, high-performance server와 multi-queue storage의 성능을 분석할 때 중요하다.

# 1. UMA

UMA(Uniform Memory Access) model에서는 모든 CPU가 memory에 접근하는 비용이 대체로 동일하다고 본다.

```text
CPU 0 ─┐
CPU 1 ─┼─ Shared Memory Controller ─ Memory
CPU 2 ─┤
CPU 3 ─┘
```

CPU 수가 증가하면 하나의 memory controller와 interconnect가 병목이 될 수 있다.

# 2. NUMA

NUMA system은 CPU와 memory resource를 여러 node로 나눈다.

```text
+----------------------+        +----------------------+
| NUMA Node 0          |        | NUMA Node 1          |
| CPU 0, CPU 1         |        | CPU 2, CPU 3         |
| Memory Controller 0  |<------>| Memory Controller 1  |
| Local Memory 0       |        | Local Memory 1       |
+----------------------+        +----------------------+
```

CPU 0이 Memory 0에 접근하면 local access다.

CPU 0이 Memory 1에 접근하면 node 사이의 interconnect를 거치는 remote access다.

일반적으로 remote access는 추가 latency와 interconnect traffic을 발생시킬 수 있다.

# 3. NUMA Node

NUMA node는 가까운 CPU와 memory resource의 집합이다.

System에 따라 node에는 다음 resource가 연결될 수 있다.

- CPU core
- Memory controller와 memory
- PCIe Root Complex
- Network 또는 storage device

정확한 topology는 hardware와 firmware, OS 설정에 따라 달라진다.

# 4. Local과 Remote Memory Access

Thread가 실행 중인 CPU와 memory page가 같은 node에 있으면 local access다.

다른 node에 있으면 remote access다.

```text
Thread on CPU 0
  ├─ Node 0 Memory → Local
  └─ Node 1 Memory → Remote interconnect 경유
```

Remote access의 영향은 workload에 따라 다르다.

- Pointer chasing처럼 dependency가 강한 workload: latency 영향이 큼
- Sequential streaming workload: bandwidth와 interconnect 경쟁이 중요
- Cache hit가 많은 workload: memory node 차이가 덜 드러날 수 있음
- Shared writable data: cache coherence traffic도 함께 발생

# 5. CPU Affinity

OS scheduler는 thread를 다른 CPU로 이동시킬 수 있다.

Thread가 이동해도 이미 할당된 memory page의 node가 자동으로 함께 이동하는 것은 아니다.

```text
처음: Thread CPU 0 + Memory Node 0
이동: Thread CPU 3 + Memory Node 0
                      ↑ remote access 가능
```

CPU affinity를 사용하면 thread가 실행할 CPU 집합을 제한할 수 있다.

하지만 CPU만 고정하고 memory placement를 고려하지 않으면 NUMA locality가 보장되지 않는다.

# 6. Memory Placement

OS는 physical page를 어느 NUMA node에 할당할지 policy를 사용한다.

대표적인 개념은 다음과 같다.

- Local allocation: 요청한 CPU와 가까운 node에서 할당
- Preferred node: 특정 node를 우선 사용
- Bind: 지정한 node 집합에서만 할당
- Interleave: 여러 node에 page를 분산

구체적인 policy 이름과 동작은 OS를 확인해야 한다.

# 7. First-Touch Policy

Linux 같은 system에서는 anonymous memory의 physical page가 처음 실제로 접근되는 시점에 할당되는 first-touch 정책이 사용될 수 있다.

```c
void *buffer = allocate_large_buffer();

// 어느 CPU의 thread가 page를 처음 write하는지에 따라
// physical page의 NUMA node가 결정될 수 있다.
initialize_buffer(buffer);
```

Main thread 하나가 전체 buffer를 초기화한 뒤 여러 worker가 다른 node에서 사용하면 page가 한 node에 몰릴 수 있다.

각 worker가 자신이 사용할 영역을 자신의 CPU에서 먼저 초기화하면 local allocation을 유도할 수 있다.

# 8. Cache Coherence와 NUMA

NUMA와 cache coherence는 다른 문제지만 함께 작동한다.

- NUMA: Physical memory와 CPU의 거리
- Cache coherence: 여러 cache에 존재하는 같은 line의 사본과 write ownership

서로 다른 node의 thread가 같은 cache line을 자주 변경하면 remote memory access뿐 아니라 node 사이의 coherence traffic도 증가할 수 있다.

```text
Node 0 Core가 Line X write
    ↔ Interconnect를 통한 ownership 이동
Node 1 Core가 Line X write
```

False sharing이 NUMA node 사이에서 발생하면 비용이 더 크게 나타날 수 있다.

# 9. NUMA와 Database

Database는 큰 buffer pool과 많은 worker thread를 사용하므로 NUMA의 영향을 받을 수 있다.

확인할 항목:

- Buffer pool page가 어느 node에 배치되는가
- Query worker와 data의 node가 일치하는가
- Lock과 shared counter가 node 사이에서 경쟁하는가
- Background thread가 어느 CPU에서 동작하는가
- Memory bandwidth가 특정 node에 몰리는가
- Huge page와 first-touch가 어떻게 결합되는가

모든 memory를 한 node에 두면 일부 access는 local해지지만 해당 node의 capacity와 bandwidth가 병목이 될 수 있다.

Interleave는 bandwidth를 분산할 수 있지만 모든 access를 local로 만들지는 않는다.

# 10. NUMA와 PCIe Device

PCIe device도 특정 CPU socket 또는 NUMA node의 Root Complex에 가까울 수 있다.

```text
Node 0 CPU / Memory / PCIe Root Complex
                    │
                    └─ NVMe SSD

Node 1 CPU / Memory
```

Node 1의 thread가 Node 0에 연결된 NVMe를 사용하고 buffer도 Node 1에 있다면 command 처리, DMA와 completion 과정에서 node 간 traffic이 발생할 수 있다.

# 11. NVMe Multi-Queue와 NUMA

NVMe는 여러 submission/completion queue와 MSI-X vector를 사용할 수 있다.

이들을 같은 locality domain에 배치하는 것이 중요하다.

```text
Application Thread
    ↓
Block/NVMe Queue
    ↓
MSI-X 처리 CPU
    ↓
I/O Buffer Memory
    ↓
PCIe NVMe Device
```

이 구성 요소가 서로 다른 node에 흩어지면 다음 비용이 발생할 수 있다.

- Remote memory access
- Cache line 이동
- Interrupt 처리 후 다른 CPU로 작업 전달
- Inter-socket bandwidth 사용
- Tail latency 증가

# 12. DMA와 NUMA

Device는 DMA address를 통해 host memory에 접근한다.

DMA 대상 buffer가 device와 가까운 node에 있는지에 따라 data path가 달라질 수 있다.

IOMMU가 address translation을 제공하더라도 물리적인 topology와 interconnect 비용이 사라지는 것은 아니다.

```text
Address를 접근할 수 있다
    ≠
가장 가까운 경로로 접근한다
```

# 13. NUMA Balancing

OS는 memory access pattern을 관찰해 page를 thread와 가까운 node로 이동시키는 automatic NUMA balancing을 제공할 수 있다.

장기적으로 locality를 개선할 수 있지만 page migration과 sampling 비용이 발생한다.

짧게 실행되는 workload나 thread가 자주 이동하는 workload에서는 안정적인 placement를 만들기 어려울 수 있다.

# 14. NUMA 성능 측정

측정할 때 CPU placement와 memory placement를 독립적으로 제어해야 한다.

```text
Case 1: CPU Node 0 + Memory Node 0 → Local
Case 2: CPU Node 0 + Memory Node 1 → Remote
Case 3: CPU 분산 + Memory Interleave
```

확인할 지표:

- Load latency
- Memory bandwidth
- Local/remote access 비율
- Cache miss
- Interconnect traffic
- Application throughput
- Average와 tail latency

한 번의 실행 결과보다 workload warm-up, 반복 실행과 분산을 기록해야 한다.

# 15. NUMA 최적화 순서

1. Hardware topology를 확인한다.
2. Thread와 process의 CPU placement를 확인한다.
3. Memory page의 placement를 확인한다.
4. Device와 interrupt의 topology를 확인한다.
5. Local/remote access와 성능을 측정한다.
6. CPU affinity와 memory policy를 하나씩 변경한다.
7. Throughput뿐 아니라 tail latency와 전체 system 영향을 확인한다.

Topology를 확인하지 않고 무조건 thread를 고정하면 scheduler의 load balancing 기회를 잃고 오히려 성능이 나빠질 수 있다.

# 16. 자주 생기는 오해

## 16.1 NUMA에서는 Remote Memory에 접근할 수 없다

접근할 수 있지만 local memory와 비용이 다를 수 있다.

## 16.2 CPU Affinity만 설정하면 NUMA 문제가 해결된다

Memory placement와 device/interrupt locality도 함께 봐야 한다.

## 16.3 모든 Memory를 Interleave하면 항상 빠르다

Bandwidth 분산에는 도움이 될 수 있지만 latency-sensitive workload와 locality에는 불리할 수 있다.

## 16.4 IOMMU가 있으면 Physical Topology는 중요하지 않다

IOMMU는 address translation과 isolation을 제공하지만 node 사이의 물리적 거리와 bandwidth 비용을 없애지 않는다.

## 16.5 NUMA는 Multi-Socket Server에서만 생각하면 된다

엄밀한 topology는 system마다 다르다. OS가 NUMA node로 노출하는 memory와 device locality가 있다면 socket 수만으로 판단하지 않고 실제 topology를 확인해야 한다.

# 정리

NUMA system에서는 memory access 비용이 CPU와 physical memory의 위치에 따라 달라질 수 있다.

```text
CPU Placement
  + Memory Placement
  + Cache Coherence
  + Device Topology
  + Interrupt Affinity
  = 실제 Locality와 성능
```

Database와 NVMe workload에서는 application thread, I/O queue, interrupt CPU, DMA buffer와 PCIe device의 위치를 하나의 data path로 분석해야 한다.

이제 CPU와 memory의 기본 구조를 바탕으로 atomic operation, cache coherence, memory ordering과 memory barrier를 학습할 수 있다.
