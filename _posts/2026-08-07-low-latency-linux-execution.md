---
title: '[Low Latency Trading] Linux 실행 환경과 CPU 격리'
date: 2026-08-07 00:50:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    Linux,
    CPU isolation,
    CPU affinity,
    NUMA,
    real-time scheduling
  ]
---

# 개요

초저지연 trading application의 thread를 특정 CPU에 고정하면 latency 문제가 모두 해결될 것처럼 생각하기 쉽다. 하지만 CPU affinity는 thread가 실행될 수 있는 CPU 집합을 제한할 뿐이다. 같은 CPU에는 다른 task, interrupt, kernel thread와 timer가 여전히 들어올 수 있다. Memory page가 다른 NUMA node에 있거나 CPU가 깊은 idle state에서 깨어나야 한다면 application thread를 고정해도 긴 지연이 발생할 수 있다.
```text
Trading Thread
  ├─ Scheduler와 다른 runnable task
  ├─ NIC interrupt와 NAPI
  ├─ Timer와 RCU callback
  ├─ Page fault와 memory reclaim
  ├─ NUMA remote access
  ├─ CPU frequency와 idle state
  └─ SMT sibling의 resource 경쟁
```
따라서 목표는 Linux를 완전히 없애는 것이 아니다. Hot path가 어떤 CPU, memory와 kernel event를 사용하는지 통제하고, 그 결과를 latency 분포와 system counter로 검증하는 것이다. 이 글의 모든 명령과 설정 이름은 예시다. Kernel version, distribution, boot loader, cgroup mode, NIC driver와 운영 정책에 따라 지원 여부와 동작이 다르므로 대상 system의 문서와 실제 상태를 반드시 확인해야 한다.

# 학습 위치

| 항목 | 내용 |
| --- | --- |
| BFS Level | Level 1-O — Operating System 실행 환경 |
| 선수 글 | [[CPU] Memory Access와 Cache Hierarchy](/posts/cpu-memory-cache-hierarchy/) |
| 선수 글 | [[Memory] NUMA 구조와 Local·Remote Memory](/posts/numa-memory-architecture/) |
| 함께 읽기 | [[Device I/O] Interrupt와 Polling, DMA는 어떻게 연결되는가](/posts/interrupt-polling-dma/) |
| 다음 연결 | Level 1-N의 NIC queue, NAPI와 application receive loop |

완료 기준은 다음 한 문장에 답할 수 있는 것이다.

> Thread affinity를 설정했는데도 latency spike가 남는 이유를 scheduler, IRQ, memory, power와 SMT 관점에서 나누어 설명할 수 있는가?

# 1. 평균보다 실행 중단을 본다

초저지연 환경에서는 CPU 사용률이나 평균 처리 시간만으로 실행 환경을 판단할 수 없다. 드물게 발생하는 scheduler migration, interrupt, page fault 또는 idle state exit가 p99.9와 최대 latency를 크게 만들 수 있으므로 다음 시간을 구분한다.

| 시간 | 질문 |
| --- | --- |
| Runnable delay | 실행할 준비가 되었는데 CPU를 언제 받았는가? |
| On-CPU time | CPU에서 실제로 얼마나 실행했는가? |
| Off-CPU time | I/O, lock, page fault 또는 sleep으로 왜 기다렸는가? |

CPU isolation은 주로 runnable delay와 비의도적 preemption을 줄이는 도구이며 algorithm, lock contention와 cache miss까지 자동으로 해결하지는 않는다.

# 2. Scheduler와 CPU Affinity

Linux scheduler는 runnable thread를 어느 CPU에서 언제 실행할지 결정한다. Thread의 CPU affinity mask는 그 thread가 실행될 수 있는 CPU의 집합이다.
```text
Allowed CPUs: 2-3

CPU 0: 실행 불가
CPU 1: 실행 불가
CPU 2: 실행 가능
CPU 3: 실행 가능
```
`sched_setaffinity()`를 사용하거나 `taskset`으로 mask를 지정할 수 있다.
```bash
taskset -pc 2 12345
```
위 명령은 PID 또는 TID `12345`의 현재 affinity를 CPU 2로 바꾸는 예시다. 실제 program이 여러 thread를 만들면 thread별 TID와 affinity를 따로 확인해야 한다.
```bash
ps -L -p 12345 -o pid,tid,psr,cls,rtprio,comm
grep -E 'Cpus_allowed_list|Mems_allowed_list' /proc/12345/status
```
Affinity의 장점은 thread migration을 줄이고 warm cache와 branch predictor state를 유지할 가능성을 높인다는 것이다. 그러나 affinity에는 다음 보장이 없다.

- 같은 CPU에서 다른 process가 실행되지 않는다는 보장
- Interrupt가 그 CPU에 오지 않는다는 보장
- Kernel workqueue와 timer가 실행되지 않는다는 보장
- Memory가 같은 NUMA node에 있다는 보장
- SMT sibling이 같은 execution resource를 사용하지 않는다는 보장

즉, affinity는 배치 조건이고 isolation은 경쟁자를 줄이는 정책이다.

# 3. CPU Affinity와 Isolation은 다르다

CPU 4에 trading thread를 고정했다고 가정해보자.
```text
CPU 4
  ├─ Trading thread
  ├─ Monitoring agent
  ├─ Kernel worker
  ├─ NIC IRQ
  └─ Scheduler tick
```
Thread는 이동하지 않지만 실행은 계속 중단될 수 있다. Isolation은 특정 CPU를 일반적인 scheduler load balancing과 일부 kernel work에서 분리해 전용 실행 환경에 가깝게 만드는 과정이다. Linux에는 하나의 `low_latency=true` switch가 없다. 다음 축을 각각 확인해야 한다.
```text
Task placement
  + Scheduler domain isolation
  + IRQ placement
  + Timer와 RCU offload
  + Memory placement
  + Power와 topology 정책
```
# 4. cpuset과 Scheduler Domain

현재 Linux kernel 문서는 cgroup v2의 cpuset isolated partition을 scheduler domain isolation의 권장 interface로 설명한다. Cpuset은 task가 사용할 CPU와 memory node를 cgroup 단위로 제한할 수 있다. Isolated partition은 해당 CPU를 일반 load balancing에서 제외할 수 있고 runtime에 구성을 조정할 수 있다. 확인할 file의 예시는 다음과 같다.
```text
/sys/fs/cgroup/cgroup.controllers
/sys/fs/cgroup/cpuset.cpus
/sys/fs/cgroup/cpuset.cpus.effective
/sys/fs/cgroup/cpuset.mems
/sys/fs/cgroup/cpuset.mems.effective
/sys/fs/cgroup/cpuset.cpus.partition
```
실제 설정 순서는 systemd가 cgroup을 관리하는지, delegation이 허용되는지와 cgroup v2가 활성화되어 있는지에 따라 달라진다. 단순히 sysfs file에 값을 쓰는 예제를 production server에 그대로 실행하면 service manager의 정책과 충돌할 수 있다. 격리 후에는 housekeeping CPU를 반드시 남겨야 한다.
```text
Housekeeping CPU
  ├─ 일반 process
  ├─ Timer
  ├─ RCU callback
  ├─ 관리용 interrupt
  └─ Monitoring과 운영 작업

Isolated CPU
  └─ Latency-sensitive thread
```
모든 CPU를 격리하면 운영 작업을 수행할 곳이 사라지거나 noise를 다른 형태로 다시 만들 수 있다.

## 4.1 isolcpus Boot Parameter

`isolcpus=`는 boot 시 CPU를 scheduler domain 등에서 분리하는 오래된 interface다. Kernel 문서는 `domain` 격리에는 runtime에 조정 가능한 cpuset을 더 유연한 방법으로 안내한다. `domain`, `nohz`, `managed_irq` 같은 flag의 의미와 지원 여부는 kernel version을 확인해야 한다.
```bash
cat /proc/cmdline
cat /sys/devices/system/cpu/isolated
```
Boot parameter는 즉시 되돌리기 어렵다. Runtime affinity와 cpuset으로 가설을 먼저 검증하고, housekeeping CPU와 복구 접속 경로를 남긴다.

# 5. IRQ도 CPU를 사용한다

NIC가 packet arrival을 알리거나 storage device가 completion을 알릴 때 interrupt가 발생할 수 있다. Application을 고정해도 그 CPU에 IRQ가 들어오면 현재 user thread의 실행이 영향을 받을 수 있다. IRQ 분포는 다음처럼 확인한다.
```bash
cat /proc/interrupts
cat /proc/irq/IRQ_NUMBER/smp_affinity_list
cat /proc/irq/IRQ_NUMBER/effective_affinity_list
```
`smp_affinity_list`는 허용 mask이고 `effective_affinity_list`는 실제 적용 결과를 보여줄 수 있다. IRQ controller와 managed interrupt의 제약 때문에 요청한 mask가 그대로 적용되지 않을 수도 있다. `irqbalance` 같은 service가 affinity를 다시 조정하는지도 확인해야 한다. NIC receive path에서는 RX queue, MSI-X vector, NAPI 처리 CPU와 application thread의 배치가 함께 중요하다.
```text
NIC RX Queue
  → MSI-X IRQ CPU
  → NAPI / Network Stack CPU
  → Application CPU
```
IRQ와 application을 같은 CPU에 두면 cache locality에 유리할 수 있지만 application이 interrupt에 의해 중단될 수 있다. 서로 다른 CPU에 두면 preemption은 줄어들 수 있지만 packet data와 queue state가 cache 사이를 이동할 수 있다. 정답은 workload, driver와 queue 구성에 따라 다르므로 두 배치를 모두 측정해야 한다.

# 6. nohz_full과 Scheduler Tick

일반적인 CPU에는 주기적인 scheduler tick이 발생할 수 있다. `nohz_full=`은 조건이 맞을 때 user space에서 하나의 task를 실행하는 CPU의 periodic tick을 멈추는 full dynticks 기능이다.
```text
Periodic tick 감소
  → 실행 중단 가능성 감소
  → 무조건 tick 0회라는 뜻은 아님
```
다음 상황에서는 kernel 진입과 event가 여전히 발생할 수 있다.

- System call
- Exception과 page fault
- Device interrupt
- 여러 runnable task
- POSIX CPU timer
- Kernel이 요구하는 per-CPU 작업

Full dynticks는 kernel 경계를 자주 드나드는 workload보다 대부분 user space에서 한 thread가 실행되는 workload에 더 잘 맞는다. 설정 상태의 예시는 다음처럼 확인한다.
```bash
cat /sys/devices/system/cpu/nohz_full
cat /proc/cmdline
```
기능이 kernel config에 포함되어 있는지와 boot CPU가 대상에서 제외되었는지도 확인해야 한다.

# 7. RCU Callback과 Housekeeping

RCU(Read-Copy Update)는 Linux kernel의 read-mostly data structure에서 사용하는 synchronization mechanism이다. RCU callback 처리가 latency-sensitive CPU에서 실행되면 noise가 될 수 있다. `rcu_nocbs=`는 지정 CPU의 callback 처리를 offload하기 위한 boot parameter다. 현재 kernel 문서에서는 `nohz_full` CPU가 RCU callback offload 대상이 되는 동작도 설명하므로 별도 parameter가 항상 필요한 것은 아니다. Kernel version과 실제 boot configuration을 기준으로 확인해야 한다. Offload는 일을 없애지 않는다.
```text
Isolated CPU에서 제거된 RCU 작업
  → Housekeeping CPU와 RCU kthread가 대신 처리
```
Housekeeping CPU가 과부하되면 callback 지연과 system 전체 문제로 돌아올 수 있다. 따라서 isolated CPU의 noise뿐 아니라 housekeeping CPU의 사용률과 run queue도 함께 측정한다.

# 8. NUMA와 First-Touch

CPU affinity를 설정해도 기존 physical page가 자동으로 해당 CPU의 NUMA node로 이동하지는 않는다. Anonymous memory는 일반적으로 page를 처음 실제로 접근할 때 allocation되는 first-touch 특성을 보일 수 있다.
```text
Main thread on Node 0
  → 전체 buffer 초기화
  → Page가 Node 0에 배치될 수 있음

Worker on Node 1
  → 같은 buffer 사용
  → Remote access 가능
```
Latency-sensitive worker가 사용할 buffer는 thread를 원하는 CPU에 배치한 뒤 해당 worker가 직접 초기화하는 방법을 검토할 수 있다. 그러나 first-touch 하나만으로 다음 요소가 모두 맞는 것은 아니다.

- CPU와 memory node
- NIC가 연결된 PCIe Root Complex
- NIC RX queue와 IRQ CPU
- Shared state를 갱신하는 다른 thread

Topology는 다음 명령으로 관찰할 수 있다.
```bash
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE
numactl --hardware
cat /sys/class/net/DEVICE/device/numa_node
```
`numactl`의 `--cpunodebind`, `--membind`와 `--preferred` 같은 option은 실험 도구가 될 수 있다. 특정 node에 memory를 강제로 묶으면 locality는 좋아질 수 있지만 capacity 부족 시 allocation failure 또는 다른 병목을 만들 수 있다. Automatic NUMA balancing을 무조건 끄기보다 page migration이 실제 spike 원인인지 먼저 확인해야 한다.

# 9. Page Fault와 Memory Lock

Virtual address를 확보한 시점에 모든 physical page가 준비된 것은 아닐 수 있다. Hot path에서 처음 code나 data page를 접근하면 minor page fault가 발생할 수 있다. Disk I/O가 필요한 major fault는 더 큰 지연을 만들 수 있다.
```text
Virtual memory 확보
  → 첫 접근
  → Page fault
  → Physical page 연결 또는 data 읽기
  → Instruction 재실행
```
`mlock()`과 `mlockall()`은 page가 swap으로 밀려나는 것을 막는 데 사용할 수 있다. 하지만 memory lock만 호출했다고 모든 fault 원인이 사라지는 것은 아니다.

- 아직 접근하지 않은 새 allocation
- Copy-on-write
- Thread stack의 추가 성장
- Lazy symbol binding
- Memory-mapped file의 page 준비
- Application이 runtime에 만드는 새로운 object

Hot path에서는 allocation을 미리 끝내고, 필요한 page를 실제로 touch하고, thread stack을 미리 사용해 보는 warm-up이 필요할 수 있다.
```c
for (size_t offset = 0; offset < buffer_size; offset += page_size) {
    buffer[offset] = 0;
}
```
위 코드는 개념 예시이며 compiler 최적화, huge page, NUMA placement와 page size를 함께 고려해야 한다. Memory lock에는 `RLIMIT_MEMLOCK`과 capability 제한이 있다. `MCL_FUTURE`를 사용한 뒤 limit을 초과하면 이후 allocation이 실패할 수 있으므로 반환값과 `errno`를 반드시 처리해야 한다. Page fault는 다음처럼 측정할 수 있다.
```bash
perf stat -e page-faults,minor-faults,major-faults -p 12345
```
# 10. CPU Idle State와 Frequency

CPU가 사용되지 않을 때 깊은 idle state에 들어가면 전력을 절약할 수 있다. 깊은 state는 일반적으로 wake-up 시 exit latency를 가질 수 있다. CPU frequency scaling도 workload와 platform에 따라 실행 시간의 변동에 영향을 줄 수 있다.
```text
전력 절약 강화
  ↔ Wake-up latency와 성능 변동 가능성
```
확인할 항목의 예시는 다음과 같다.
```bash
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_driver
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_min_freq
cat /sys/devices/system/cpu/cpu*/cpuidle/state*/latency
```
`performance` governor는 허용 범위 안에서 높은 frequency를 요청하지만 모든 hardware와 driver에서 동일한 결과를 보장하지 않는다. Turbo, thermal throttling, package power limit와 shared frequency domain도 남아 있다. Idle state를 모두 비활성화하거나 최저 frequency를 최고값으로 올리는 변경은 전력, 발열과 다른 core의 성능에 영향을 준다. PM QoS 또는 platform별 정책도 포함해 변경 전후를 측정해야 한다.

# 11. SMT Sibling

SMT가 활성화된 CPU에서는 하나의 physical core가 여러 logical CPU로 보일 수 있다. Logical CPU가 다르다고 execution resource까지 완전히 독립적인 것은 아니다.
```text
Physical Core 3
  ├─ Logical CPU 6: Trading thread
  └─ Logical CPU 7: Logging thread

공유 가능 자원
  → Front-end, execution unit, cache와 bandwidth 일부
```
Sibling에서 다른 workload가 실행되면 latency가 변할 수 있다. 따라서 CPU 번호를 고를 때 logical CPU 목록만 보지 않고 core와 sibling topology를 확인한다.
```bash
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE
cat /sys/devices/system/cpu/cpu6/topology/thread_siblings_list
```
선택지는 SMT 전체 비활성화, 대상 core의 sibling을 비워 두기, 또는 두 thread의 간섭을 측정해 함께 배치하기다. SMT를 끄면 사용 가능한 logical CPU 수와 throughput이 줄어들 수 있다.

# 12. SCHED_FIFO는 마지막 수단이다

`SCHED_FIFO`는 real-time scheduling policy다. Runnable 상태의 높은 priority `SCHED_FIFO` thread는 일반 `SCHED_OTHER` thread보다 우선하며 같은 priority 안에서 time slice 없이 실행될 수 있다. 이 특성은 잘못 사용하면 매우 위험하다.
```text
무한 loop의 높은 priority thread
  → 같은 CPU의 일반 task 실행 기회 상실
  → Network, logging, monitoring 또는 운영 shell 지연
```
Lock을 가진 낮은 priority thread가 실행되지 못하면 높은 priority thread도 기다리는 priority inversion 문제가 나타날 수 있다. Real-time policy는 code를 빠르게 만들지 않는다. 누가 먼저 CPU를 받는지를 바꾼다. 적용 전에는 다음이 필요하다.

- Bounded execution과 명확한 blocking point
- Watchdog와 외부 복구 경로
- Priority 관계와 lock 분석
- RT runtime throttling 정책 확인
- `CAP_SYS_NICE`와 resource limit 검토
- Housekeeping 및 critical kernel thread의 실행 가능성 확인

상태 확인 예시는 다음과 같다.
```bash
chrt -p 12345
ps -L -p 12345 -o tid,psr,cls,rtprio,pri,comm
```
Production에서 `chrt`로 policy를 변경하는 명령은 영향 범위를 검토하지 않고 실행하면 안 된다.

# 13. 검증 순서

여러 설정을 한 번에 바꾸면 무엇이 latency를 개선하거나 악화시켰는지 알 수 없다. 다음 순서로 한 단계씩 검증한다.

## 13.1 Baseline을 고정한다

- 같은 binary와 configuration을 사용한다.
- Traffic rate, message size와 burst pattern을 고정한다.
- Warm-up 구간과 측정 구간을 나눈다.
- p50, p95, p99, p99.9와 maximum을 함께 기록한다.
- CPU, kernel, firmware와 BIOS version을 기록한다.

## 13.2 Topology를 그린다
```text
Application Thread CPU
  ↔ SMT Sibling
  ↔ NUMA Memory Node
  ↔ NIC PCIe NUMA Node
  ↔ RX Queue / IRQ CPU
```
Topology를 모르면 CPU 번호 선택은 추측이 된다.

## 13.3 Affinity만 적용한다

먼저 thread migration이 원인인지 확인한다.
```bash
perf stat -e context-switches,cpu-migrations,page-faults -p 12345
```
## 13.4 IRQ와 Kernel Noise를 관찰한다

`/proc/interrupts`, scheduler trace와 softirq 통계를 latency spike 시점과 비교한다.

## 13.5 Memory를 준비한다

Allocation, first-touch, warm-up과 memory lock을 각각 분리해 측정한다.

## 13.6 Scheduler Isolation을 적용한다

Cpuset isolated partition 또는 대상 환경의 지원 방법을 사용하고 housekeeping CPU 상태를 함께 본다.

## 13.7 Tick, RCU와 Power를 조정한다

Trace에서 해당 원인이 보일 때 `nohz_full`, RCU offload, idle 또는 frequency policy를 검토한다.

## 13.8 Real-Time Policy를 마지막에 검토한다

일반 policy와 isolation만으로 목표를 달성할 수 있는지 먼저 확인한다.

# 14. 자주 생기는 오해

## 14.1 taskset을 사용하면 CPU가 전용이 된다

Affinity는 대상 thread의 허용 CPU만 제한한다. 다른 task와 IRQ를 제거하려면 별도 정책이 필요하다.

## 14.2 isolcpus 하나면 모든 OS Noise가 사라진다

Scheduler domain, IRQ, tick, RCU, workqueue와 timer는 서로 다른 경로다. 각 경로의 실제 상태를 확인해야 한다.

## 14.3 mlockall을 호출하면 Page Fault가 0이 된다

Memory lock은 swap-out 방지와 residency에 도움을 주지만 새 allocation, copy-on-write와 code path의 첫 접근까지 자동으로 없애지 않는다.

## 14.4 performance Governor면 Frequency가 항상 고정된다

Scaling driver, turbo, thermal limit와 hardware policy에 따라 실제 frequency는 달라질 수 있다.

## 14.5 SCHED_FIFO가 가장 빠르다

Scheduling priority를 높일 뿐 instruction 수, cache miss와 lock contention을 줄이지 않는다. 잘못 적용하면 system 전체의 progress를 막을 수 있다.

# 15. 완료 기준

다음 질문에 자신의 문장으로 답하고 측정 결과를 제시할 수 있어야 한다.

1. CPU affinity와 scheduler isolation은 무엇이 다른가?
2. Housekeeping CPU를 남겨야 하는 이유는 무엇인가?
3. NIC IRQ와 application thread를 같은 CPU에 둘 때와 나눌 때의 trade-off는 무엇인가?
4. `nohz_full`이 모든 interrupt와 kernel 진입을 제거하지 못하는 이유는 무엇인가?
5. RCU callback offload는 일을 어디로 이동시키는가?
6. Thread를 고정한 뒤에도 remote NUMA access가 발생할 수 있는 이유는 무엇인가?
7. `mlockall()` 이후에도 준비해야 할 page fault 원인은 무엇인가?
8. Deep idle state, frequency scaling과 SMT sibling이 tail latency에 어떤 영향을 줄 수 있는가?
9. `SCHED_FIFO`가 system을 멈추게 할 수 있는 시나리오는 무엇인가?
10. 한 번에 하나의 설정만 바꾸고 효과를 입증할 실험을 설계할 수 있는가?

# 정리

Linux의 low-latency 실행 환경은 thread 하나를 특정 CPU에 고정하는 작업으로 끝나지 않는다.
```text
CPU Affinity
  → Thread가 실행될 CPU를 제한

CPU Isolation
  → 일반 load balancing과 일부 kernel noise를 분리

IRQ / NAPI Placement
  → Device event가 처리될 CPU를 결정

NUMA / First-Touch / mlock
  → Hot data가 준비되고 접근되는 위치를 통제

Power / SMT / Scheduler Policy
  → 실행 시간의 변동과 경쟁 조건을 관리
```
가장 중요한 원칙은 설정 이름을 많이 아는 것이 아니다. Latency spike 시점에 thread가 왜 실행되지 못했는지 증거로 분류하고, 가장 작은 변경으로 가설을 검증하는 것이다.

# 참고 자료

- [Linux Kernel — CPU Isolation](https://docs.kernel.org/admin-guide/cpu-isolation.html)
- [Linux Kernel — The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html)
- [Linux Kernel — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [Linux Kernel — SMP IRQ affinity](https://docs.kernel.org/core-api/irq/irq-affinity.html)
- [Linux Kernel — Affinity managed interrupts](https://docs.kernel.org/core-api/irq/managed_irq.html)
- [Linux Kernel — CPU Idle Time Management](https://docs.kernel.org/admin-guide/pm/cpuidle.html)
- [Linux Kernel — CPU Performance Scaling](https://docs.kernel.org/admin-guide/pm/cpufreq.html)
- [Linux man-pages — sched_setaffinity(2)](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)
- [Linux man-pages — sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)
- [Linux man-pages — mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html)
- [Linux man-pages — set_mempolicy(2)](https://man7.org/linux/man-pages/man2/set_mempolicy.2.html)
