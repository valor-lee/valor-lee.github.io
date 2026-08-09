---
title: '[Low Latency Trading] Latency를 올바르게 측정하는 방법'
date: 2026-08-07 01:10:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    latency measurement,
    benchmarking,
    performance engineering,
    tail latency
  ]
---

# 개요

Low-Latency System에서 숫자를 얻는 일과 올바른 latency를 측정하는 일은 다르다.
Timer 앞뒤의 값을 빼면 시간은 나오지만 다음 질문에 답하지 못하면 결과를 해석할 수 없다.

- 어느 두 지점 사이를 측정했는가?
- 두 Timestamp가 같은 Clock Domain에 속하는가?
- Clock을 읽는 비용이 대상 연산보다 크지 않은가?
- 부하 생성기가 느린 구간에 요청을 덜 보내지 않았는가?
- 평균이 숨긴 Tail과 최대값을 보존했는가?
- Batching과 Queue 대기 시간을 포함했는가?

```text
좋은 Benchmark
  = 명확한 경계
  + 적절한 Clock
  + 독립적인 부하 모델
  + 전체 Distribution
  + 재현 가능한 환경
```

이 글에서는 Trading System의 단계별 latency budget부터 TSC, PTP와 Coordinated Omission까지 하나의 측정 절차로 연결한다.

# 선수 글과 BFS 위치

엄격한 선수 글은 [[Low Latency Trading] 초저지연 트레이딩 시스템 전체 구조](/posts/low-latency-trading-system-overview/)다.
다음 글은 같은 Level에서 함께 읽는 보강 자료다.

- [[Low Latency Trading] Hot Path를 위한 Modern C++ 설계 원칙](/posts/low-latency-cpp-hot-path/)
- [[Low Latency Trading] Linux 실행 환경과 CPU 격리](/posts/low-latency-linux-execution/)
- [[Low Latency Trading] Market Data가 NIC에서 Application까지 오는 길](/posts/low-latency-network-path/)

이 글의 BFS 위치는 **Level 1-T: 시간과 측정의 공통 기반**이다.

# 1. 먼저 Latency의 정의를 고정한다

“우리 시스템 latency는 3us다”라는 문장은 경계가 없으면 의미가 불완전하다.
Trading System에는 여러 latency가 있다.

| 이름 | 시작 | 끝 |
| --- | --- | --- |
| Feed decode | User-space가 Packet을 관찰 | Domain Event 생성 |
| Book update | 정상 Sequence Event 입력 | Order Book 상태 반영 |
| Decision | Book Event 입력 | Strategy Decision 생성 |
| Risk | Order Intent 입력 | 승인 또는 거부 |
| Software tick-to-trade | Application receive | Application send |
| Wire tick-to-trade | NIC RX | NIC TX |
| Order round trip | Order 전송 | Exchange ACK 수신 |

같은 이름을 사용해도 Timestamp 위치가 다르면 다른 값이다.
측정 문서에는 시작 Event, 종료 Event, Clock과 포함·제외 구간을 함께 적는다.

# 2. Timestamp Point를 Data Path에 표시한다

한 Packet이 주문으로 변하는 경로에 Timestamp를 배치해보자.

```text
t0  NIC hardware RX
 │
t1  Application receive
 │  decode
t2  Domain event ready
 │  book update
t3  Book state ready
 │  strategy + risk
t4  Order approved
 │  encode + send
t5  Application send return
 │
t6  NIC hardware TX
 │
t7  Exchange ACK의 NIC hardware RX
```

같은 Clock Domain이라는 조건이 만족되면 다음처럼 계산할 수 있다.

```text
Decode latency          = t2 - t1
Book latency            = t3 - t2
Decision + Risk latency = t4 - t3
Application TX latency  = t5 - t4
Software tick-to-trade  = t5 - t1
Wire tick-to-trade      = t6 - t0
Order round trip        = t7 - t6
```

NIC PHC Timestamp와 System Clock Timestamp를 바로 빼서는 안 된다.
Clock 사이의 offset과 변환 관계를 알고 있거나 같은 Domain으로 변환해야 한다.

# 3. Latency Budget은 위치별 책임을 만든다

End-to-end 목표만 있으면 어느 Component가 Budget을 사용했는지 알기 어렵다.

```text
End-to-end budget
  ├─ Receive + Decode
  ├─ Book Update
  ├─ Strategy
  ├─ Risk
  ├─ Encode + Queue
  └─ Transmit
```

Budget은 평균만으로 작성하지 않는다.
각 구간에 다음 항목을 둔다.

- 정상 부하의 p50
- 목표 부하의 p99와 p99.9
- Burst에서의 Queue Depth와 Drop
- 허용 가능한 최대값 또는 Timeout
- Budget 초과 시 정책

Component Budget의 합은 설계상의 상한을 배분하는 데 유용하다.
하지만 각 Component의 p99를 더해 End-to-end p99라고 부르면 안 된다.

# 4. Resolution, Precision과 Accuracy

세 용어는 서로 다른 질문에 답한다.

## 4.1 Resolution

Clock이나 기록 형식이 구분해 표현할 수 있는 가장 작은 간격이다.

`clock_getres()`가 보고한 Resolution이 1ns라고 해서 실제 측정이 1ns까지 정확하다는 뜻은 아니다.

## 4.2 Precision

같은 조건에서 반복 측정한 값이 얼마나 일관되게 모이는지를 뜻한다.
이 글에서는 반복 측정의 산포가 작을수록 Precision이 높다고 표현한다.

## 4.3 Accuracy

Timestamp가 기준이 되는 실제 시간에 얼마나 가까운지를 뜻한다.
Clock이 매우 세밀한 숫자를 반환해도 기준 Clock과 10us 어긋나 있으면 Resolution은 높고 Accuracy는 낮을 수 있다.

```text
Resolution: 얼마나 잘게 표현하는가?
Precision : 반복 값이 얼마나 모이는가?
Accuracy  : 기준 시간에 얼마나 가까운가?
```

여기에 Clock Read Cost와 Clock Stability도 별도로 기록해야 한다.

# 5. Linux Clock을 목적에 맞게 선택한다

| Clock | 주요 성질 | 적합한 용도 |
| --- | --- | --- |
| `CLOCK_REALTIME` | Wall Clock이며 설정 변경으로 불연속 이동 가능 | Audit와 외부 시간 대응 |
| `CLOCK_MONOTONIC` | 불연속 Wall Clock 변경의 영향을 받지 않지만 점진적 보정 영향은 받음 | 한 Host 안의 일반 Duration |
| `CLOCK_MONOTONIC_RAW` | NTP·`adjtime`의 점진적 보정을 받지 않는 Linux Raw Clock | 짧은 구간과 Clock 진단 |
| TSC | x86 CPU의 Counter | 매우 낮은 Overhead의 Cycle 측정 |
| NIC PHC | NIC가 제공하는 PTP Hardware Clock | Wire에 가까운 Hardware Timestamp |

`CLOCK_REALTIME`은 사람이 읽는 시각과 여러 Host의 Log를 연결하는 데 필요하다.

그러나 관리자가 시간을 변경하거나 동기화 과정에서 불연속 변화가 생기면 Duration이 음수가 되거나 튈 수 있다.
한 Process 안의 경과 시간은 보통 monotonic 계열 Clock을 사용한다.

`CLOCK_MONOTONIC_RAW`는 Wall Clock에 맞추기 위한 주파수 보정을 받지 않으므로 장기간에는 기준 시간과 Drift가 생길 수 있다.

# 6. `clock_gettime()`도 비용을 측정한다

Linux에서는 지원되는 Clock의 `clock_gettime()`이 vDSO를 통해 System Call 없이 실행될 수 있다.
하지만 Architecture, Kernel, Clock 종류와 Runtime 조건에 따라 경로가 달라질 수 있다.
다음 Wrapper는 Linux Raw Monotonic Clock을 nanosecond로 바꾼다.

```cpp
#include <cstdint>
#include <cstdlib>
#include <ctime>

std::uint64_t raw_now_ns() noexcept {
    timespec ts{};
    if (::clock_gettime(CLOCK_MONOTONIC_RAW, &ts) != 0) {
        std::abort();
    }
    return static_cast<std::uint64_t>(ts.tv_sec) * 1'000'000'000ULL
         + static_cast<std::uint64_t>(ts.tv_nsec);
}
```

대상 Host에서 Back-to-back Read를 반복해 Clock Read 자체의 Distribution을 구한다.

```cpp
for (std::size_t i = 0; i < samples; ++i) {
    const auto t0 = raw_now_ns();
    const auto t1 = raw_now_ns();
    clock_read_histogram.record(t1 - t0);
}
```

한 번 구한 평균을 모든 결과에서 기계적으로 빼면 안 된다.
Clock 비용도 Distribution이며 Cache, Core Migration과 간섭의 영향을 받을 수 있다.
빈 구간 측정값과 실제 측정값을 함께 보고해 Measurement Floor를 드러낸다.

# 7. TSC를 사용할 때 확인할 것

TSC는 x86의 Time-Stamp Counter다.

`RDTSC` 또는 `RDTSCP`로 읽을 수 있지만 단순히 두 instruction을 넣는 것만으로 정확한 경계가 만들어지지는 않는다.

`RDTSC`는 serializing instruction이 아니다. `RDTSCP`도 이전 instruction과 load에는 더 강한 ordering을 제공하지만 이전 store가 globally visible해질 때까지 기다리거나 뒤 instruction의 실행 시작을 막는 완전한 양방향 barrier는 아니다. 측정 경계가 load와 store 중 무엇을 포함하는지에 따라 Intel 문서가 요구하는 `LFENCE`·`MFENCE` 배치를 선택하고, intrinsic 또는 compiler barrier로 compiler의 재배치도 막아야 한다.

- CPU가 Timestamp Read 주변 instruction을 재배치해 실행할 수 있다.
- Counter가 어떤 Frequency로 증가하는지 확인해야 한다.
- Core 사이 TSC 동기화와 Process Migration 영향을 확인해야 한다.
- Virtual Machine에서는 Hypervisor 정책을 확인해야 한다.
- Cycle을 ns로 바꾸는 Calibration과 Drift 정책이 필요하다.

Invariant TSC가 있으면 CPU Core Frequency가 변해도 TSC가 일정한 기준 속도로 증가할 수 있다.
따라서 “현재 CPU가 4GHz이므로 1 cycle은 0.25ns”처럼 Turbo Frequency를 그대로 변환식에 넣으면 안 된다.
Intel 문서가 설명하는 instruction ordering과 fence 조건을 따르고, 대상 Microarchitecture에서 검증된 Timestamp Utility를 사용한다.
Thread Pinning은 Migration 변수를 줄이지만 TSC의 모든 문제를 자동으로 해결하지 않는다.

# 8. Measurement가 대상 경로를 바꿀 수 있다

모든 단계마다 Clock을 읽고 문자열 Log를 남기면 측정 때문에 Hot Path가 달라진다.
이를 Observer Effect로 볼 수 있다.

```text
원래 경로
  Decode → Book → Risk → Encode
계측 경로
  Clock → Decode → Clock → Log → Book → Clock → ...
```

다음 방법을 조합한다.

- Sampling Rate를 명시하고 일부 Event만 상세 측정한다.
- Timestamp를 고정 크기 Record에 저장하고 다른 Thread가 집계한다.
- 문자열 Formatting과 File I/O를 Hot Path 밖으로 옮긴다.
- 전체 계측 On/Off Build를 모두 측정한다.
- Stage Microbenchmark와 End-to-end Hardware Timestamp를 교차 검증한다.

계측을 줄인 결과가 원래 경로와 동일한 상태 전이를 만드는지도 확인해야 한다.

# 9. 실험 환경을 기록한다

Latency는 Source Code만으로 결정되지 않는다.
최소한 다음 환경을 보고서에 남긴다.

- CPU Model, Microcode, Socket과 NUMA Topology
- SMT, Turbo와 Power Governor 상태
- Kernel Version과 Boot Parameter
- CPU Affinity, Isolation과 IRQ Affinity
- NIC Model, Firmware, Driver와 Queue 설정
- Compiler, Standard Library, Build Flag와 Commit
- Huge Page, Memory Locking과 Page Placement
- Background Process와 Monitoring Agent
- 실험 중 온도와 Frequency Throttling 여부

Noise를 줄인 Lab 환경과 실제 운영에 가까운 환경은 목적이 다르다.
Lab 결과만으로 Production Tail을 약속하지 말고 두 환경의 차이를 문서화한다.

# 10. Warm-up은 결과를 버리는 기술이 아니다

처음 몇 번의 요청은 다음 비용을 포함할 수 있다.

- Code와 Data Cache Fill
- Branch Predictor 학습
- Page Fault와 First Touch
- Dynamic Loader와 Lazy Binding
- Allocator 초기화
- CPU Frequency와 Thermal State 변화
- NIC Queue와 Connection 상태 준비

Warm-up은 “느린 값이 보기 싫어서 제거”하는 과정이 아니다.
Production에서 Cold Start가 발생한다면 별도의 Cold-Start Distribution으로 측정해야 한다.
Steady State Benchmark에서는 Warm-up 시간만 적지 말고 안정 상태 기준을 정한다.

```text
Page Fault가 더 발생하지 않는다.
Offered Load와 Achieved Throughput이 안정적이다.
CPU Frequency와 Temperature가 허용 범위다.
Queue Depth와 Latency Histogram이 반복 구간에서 안정적이다.
```

# 11. 평균 대신 Distribution을 보존한다

평균이 2us여도 일부 주문이 200us 걸릴 수 있다.
최소한 다음 값을 함께 본다.

| 지표 | 해석 |
| --- | --- |
| p50 | 전형적인 중앙값 |
| p90 또는 p95 | 일반적인 느린 구간의 시작 |
| p99 | 100개 중 약 1개의 Tail 경계 |
| p99.9 | 1,000개 중 약 1개의 Tail 경계 |
| p99.99 | 더 드문 Jitter와 Stall 관찰 |
| max | 측정 구간에서 실제 관찰한 최악값 |

p99.9를 신뢰하려면 Tail에 충분한 Sample이 있어야 한다.
Sample 1,000개에서 p99.9를 말하면 최상위 극소수 값에 결과가 의존한다.
높은 Percentile일수록 더 긴 실행, 반복 Run과 안정성 분석이 필요하다.
Max는 실행 길이에 민감하지만 버려야 할 값은 아니다.
원인을 Packet Capture, Scheduler Event, Page Fault와 Queue Depth로 추적한다.

# 12. Histogram의 범위와 정밀도를 기록한다

Histogram은 개별 Sample을 Bucket에 집계한다.
설정이 잘못되면 긴 Tail이 잘리거나 서로 다른 값이 같은 Bucket에 과도하게 합쳐질 수 있다.
HdrHistogram을 사용할 때도 다음을 명시한다.

- Lowest Discernible Value
- Highest Trackable Value
- Significant Digits
- 단위가 cycle, ns 또는 us인지
- Overflow 또는 범위 초과 Sample 처리
- Interval Histogram을 언제 Rotate하는지

서로 다른 Run의 p99를 평균내지 않는다.
동일한 측정 정의, 단위, bucket schema와 부하라면 Histogram의 Count를 합친 뒤 Percentile을 다시 계산할 수 있다.
Version, 부하 또는 환경이 다르면 섞지 않고 별도 Distribution으로 비교한다.

# 13. Percentile은 단계별로 더할 수 없다

다음 식은 일반적으로 성립하지 않는다.

```text
p99(Decode + Book + Risk)
    ≠ p99(Decode) + p99(Book) + p99(Risk)
```

각 Stage의 느린 Sample이 같은 Event에서 발생한다는 보장이 없고 Stage 사이 상관관계도 있기 때문이다.
Stage Histogram은 병목 진단에 사용한다.
SLO를 판단할 End-to-end Percentile은 같은 Event의 시작과 끝을 직접 측정해 구한다.

# 14. Coordinated Omission

Closed-loop Generator는 이전 요청이 끝난 뒤 다음 요청을 보낸다.

```text
Send A → Wait A → Send B → Wait B → Send C
```

System이 100ms 멈추면 Generator도 같이 멈추므로 그동안 도착했어야 할 요청을 생성하지 않는다.
나쁜 구간에서 Sample 수가 줄어 결과가 실제 Offered Load보다 좋아 보이는 현상이 Coordinated Omission이다.
Open-loop Generator는 응답 완료와 독립적인 Schedule을 만든다.

```text
Intended send: 0us, 10us, 20us, 30us, ...
System stall :          [----------]
```

각 요청에 대해 세 시간을 구분한다.

```text
Scheduled Time
Actual Dispatch Time
Completion Time
```

```cpp
for (std::uint64_t i = 0; i < count; ++i) {
    const auto scheduled = start + i * interval_ns;
    wait_until(scheduled);
    const auto dispatched = raw_now_ns();
    dispatch_lag.record(dispatched - scheduled);

    if (!submit_async(Request{i, scheduled, dispatched})) {
        dropped_requests.increment();
    }
}

void on_completion(const Request& request) {
    const auto completed = raw_now_ns();
    service_time.record(completed - request.dispatched);
    response_time.record(completed - request.scheduled);
}
```

Generator가 Schedule을 따라가지 못하면 대상 System이 아니라 Generator가 병목일 수 있다.
동기 API라면 여러 Connection이나 Worker, 또는 별도 Traffic Generator로 응답과 발행 Schedule을 분리해야 한다.
Generator CPU, Dispatch Lag, 실제 Offered Rate를 함께 측정하고 필요하면 별도 Host나 Core를 사용한다.
HdrHistogram의 Coordinated Omission 보정 기능은 예상 간격이 알려진 경우 분석을 도울 수 있다.
그러나 실제 Open-loop 부하 생성과 Queue 대기 측정을 대체하지는 않는다.

# 15. Throughput, Queue와 Batching

Latency는 Offered Load와 함께 제시해야 한다.

```text
낮은 부하      → Queue가 거의 없음
포화점 근처    → Queue 대기와 Tail 증가
포화점 초과    → Backlog, Drop 또는 Timeout
```

다음 수치를 분리한다.

- Offered Messages per Second
- Accepted Messages per Second
- Completed Messages per Second
- Rejected 또는 Dropped Messages
- Queue Depth와 Queueing Time

Batching은 한 번의 syscall, doorbell 또는 queue operation으로 여러 Message를 처리해 Throughput을 높일 수 있다.
하지만 첫 Message는 Batch가 찰 때까지 기다릴 수 있다.
Batch 전체 시간만 재서 Message 수로 나누면 각 Message의 Queueing Distribution을 숨긴다.
Batch Size, Batch 안의 Position과 Flush Timeout을 기록하고 Message 기준 End-to-end latency도 측정한다.
Steady Rate뿐 아니라 실제 Market Data처럼 짧은 Burst를 포함한 부하도 사용한다.

# 16. PTP, PHC와 Hardware Timestamp

서로 다른 Host 사이의 One-way Latency는 Clock 동기화가 없으면 직접 계산할 수 없다.
PTP는 Network를 통해 Clock을 동기화하는 Protocol이고, PHC는 NIC 같은 hardware device의 clock을 Linux가 노출하는 PTP Hardware Clock interface다.
Linux 환경에서는 일반적으로 다음 관계를 볼 수 있다.

```text
Grandmaster
    ↕ PTP
NIC PHC ← ptp4l
    ↕
System Clock ← phc2sys
```

`ptp4l`은 PTP를 구현하고 Hardware Timestamp Mode에서 PHC를 사용한다.

`phc2sys`는 보통 동기화된 PHC와 System Clock 사이를 맞추는 데 사용한다.

linuxptp의 hardware timestamp mode에서는 PHC가 PTP time scale을, `CLOCK_REALTIME`이 UTC를 따를 수 있다. `phc2sys`가 관리하는 UTC offset까지 포함해 변환해야 하며 두 clock의 숫자를 같은 epoch라고 가정하면 안 된다.

Hardware Timestamp는 Application Timestamp보다 Wire에 가까운 지점을 관찰할 수 있다.
Linux `SO_TIMESTAMPING` API는 Software와 Hardware RX/TX Timestamp를 전달한다.
`SOF_TIMESTAMPING_RAW_HARDWARE` 값은 system-clock으로 변환된 시간이 아니라 device hardware clock의 raw timestamp이므로 연관된 PHC와 clock domain을 확인해야 한다.
TX Hardware Timestamp는 비동기로 도착할 수 있으므로 Packet과 Timestamp의 상관 관계와 Error Queue 처리를 구현해야 한다.
Hardware Timestamp를 사용해도 다음 오차는 남는다.

- Grandmaster와 PHC Offset
- PTP Path Delay와 비대칭
- PHC와 System Clock 변환 오차
- Hardware Timestamp가 device의 MAC·PHY pipeline 중 어디에서 생성됐는지, Software Timestamp가 driver·kernel stack의 어느 hook에서 생성됐는지의 차이
- Clock Servo가 Lock을 잃은 구간

결과에는 PTP Lock 상태, Offset의 RMS·Max, 측정 Clock과 Timestamp 위치를 함께 남긴다.

# 17. 단계별 Benchmark Ladder

한 번에 전체 시스템만 측정하면 원인을 찾기 어렵다.
다음 순서로 범위를 넓힌다.

```text
1. Clock Read와 Empty Bracket
        ↓
2. Pure Function Microbenchmark
        ↓
3. Component Benchmark
        ↓
4. In-Process Pipeline
        ↓
5. Socket 또는 NIC Loopback
        ↓
6. Exchange Emulator End-to-end
        ↓
7. Production-like Network와 Hardware Timestamp
```

Microbenchmark는 Mechanism을 격리하지만 실제 Queue, Network와 Scheduler를 생략한다.
End-to-end Benchmark는 실제 효과를 보여주지만 원인 분리가 어렵다.
두 종류의 결과가 같은 방향을 가리킬 때 최적화 근거가 강해진다.

# 18. 재현 가능한 Benchmark Protocol

## Step 1. 질문과 경계

“새 자료구조가 빠른가?”보다 “목표 500k msg/s에서 Book Update p99.9가 줄어드는가?”처럼 쓴다.
시작·종료 Timestamp, 포함 구간, 단위와 SLO를 고정한다.

## Step 2. Correctness Baseline

동일 Input에서 이전 구현과 같은 상태를 만드는지 Replay와 Invariant Test로 확인한다.
잘못된 결과를 낸 빠른 Run은 폐기한다.

## Step 3. 환경 고정과 기록

Build, CPU, NUMA, IRQ, NIC와 Kernel 설정을 기록한다.
통제할 수 없는 변수도 숨기지 않고 적는다.

## Step 4. Clock 검증

Clock 종류, Resolution, Read Cost, Core Migration과 Clock Domain 변환을 확인한다.

## Step 5. Precondition과 Warm-up

Memory를 Pre-Touch하고 대표 Input으로 경로를 준비한다.
Cold Start는 별도 Scenario로 보존한다.

## Step 6. Load Matrix

Message Size, Offered Rate, Burst, Batch Size와 Queue Depth를 바꾼다.
포화점 아래, 근처와 위를 모두 측정한다.

## Step 7. Open-loop 실행

Intended Schedule을 기준으로 부하를 만들고 Dispatch Lag를 기록한다.
Generator가 목표 부하를 실제로 전달했는지 확인한다.

## Step 8. Distribution과 Counter 수집

p50, p99, p99.9, p99.99, max와 Histogram을 보존한다.
CPU, Context Switch, Page Fault, Cache/TLB Miss, Queue와 Drop Counter를 함께 수집한다.

## Step 9. 반복과 순서 교차

여러 Run을 수행하고 A/B 순서를 바꿔 Thermal State와 시간 경과의 영향을 줄인다.
Median Run만 고르지 말고 Run 간 변동을 공개한다.

## Step 10. End-to-end 검증

Component 개선이 Wire 또는 Emulator End-to-end에서도 유지되는지 확인한다.

# 19. 결과 보고서 Template

```text
Hypothesis
  무엇이 왜 줄어들 것으로 예상했는가?
Measurement Contract
  Start, End, Clock, Unit, 포함 구간은 무엇인가?
Environment
  Hardware, Kernel, Build, Pinning, NIC 설정은 무엇인가?
Workload
  Offered Rate, Burst, Message, Batch와 실행 시간은 무엇인가?
Correctness
  동일 상태와 Invariant를 어떻게 검증했는가?
Results
  Histogram, Percentile, Max, Throughput와 Drop은 무엇인가?
Mechanism
  PMU, Queue와 Trace가 설명하는 원인은 무엇인가?
Limitations
  생략한 Hardware와 Production 조건은 무엇인가?
Reproduction
  Commit, Command, Config와 Raw Result 위치는 어디인가?
```

# 완료 기준

다음을 만족하면 이 단계의 학습을 완료한 것으로 본다.

1. Software tick-to-trade와 Wire tick-to-trade의 Timestamp 경계를 그린다.
2. Resolution, Precision, Accuracy와 Clock Read Cost를 구분해 설명한다.
3. `REALTIME`, `MONOTONIC`, `MONOTONIC_RAW`, TSC와 PHC의 용도를 구분한다.
4. 대상 Host에서 Clock Read와 Empty Bracket Distribution을 측정한다.
5. Warm-up과 Cold Start 결과를 분리한다.
6. p50, p99, p99.9, p99.99와 max를 Histogram에서 계산한다.
7. Stage Percentile을 합하지 않고 End-to-end를 직접 측정한다.
8. Closed-loop와 Open-loop 결과에서 Coordinated Omission을 재현한다.
9. Offered·Completed·Dropped Throughput과 Queue Depth를 함께 보고한다.
10. PTP Offset과 Hardware Timestamp 위치를 포함한 보고서를 작성한다.

# 자주 생기는 오해

## Nanosecond 단위가 나오면 Nanosecond까지 정확하다

표시 단위와 Resolution, Precision, Accuracy는 다르다.
Clock Source와 동기화 오차를 확인해야 한다.

## 평균을 여러 번 재면 Tail도 알 수 있다

평균은 Distribution 모양과 드문 Stall을 보존하지 않는다.
Histogram과 충분한 Sample이 필요하다.

## 각 단계 p99를 더하면 전체 p99다

Percentile은 일반적으로 선형적으로 더할 수 없다.
같은 Event의 End-to-end Timestamp가 필요하다.

## 가장 느린 값은 Noise이므로 지운다

Max는 Interrupt, Page Fault, Packet Loss와 실제 장애를 보여줄 수 있다.
제거하려면 원인을 분류하고 제거 전후 결과를 모두 남긴다.

## Hardware Timestamp면 오차가 없다

Wire에 가까운 Timestamp일 뿐 Clock Offset, Path Asymmetry와 Timestamp 지점의 불확실성은 남는다.

# 정리

Latency 측정은 Timer API를 고르는 문제보다 Measurement Contract를 세우는 문제다.

```text
Boundary를 정의한다.
  → Clock Domain과 Read Cost를 검증한다.
  → Warm-up과 환경을 기록한다.
  → Open-loop로 목표 부하를 제공한다.
  → Histogram으로 Tail을 보존한다.
  → Throughput, Queue와 Drop을 함께 본다.
  → Hardware Timestamp로 End-to-end를 교차 검증한다.
```

좋은 결과는 가장 작은 숫자가 아니라 다른 사람이 같은 조건에서 다시 얻고 반박할 수 있는 숫자다.

# 참고 자료

- [Linux `clock_gettime(2)` Manual](https://man7.org/linux/man-pages/man2/clock_gettime.2.html)
- [Linux `vdso(7)` Manual](https://man7.org/linux/man-pages/man7/vdso.7.html)
- [Linux Kernel Timestamping Documentation](https://docs.kernel.org/networking/timestamping.html)
- [Linux PTP Hardware Clock Infrastructure](https://docs.kernel.org/driver-api/ptp.html)
- [Intel 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [linuxptp `ptp4l(8)`](https://www.linuxptp.org/documentation/ptp4l/)
- [linuxptp `phc2sys(8)`](https://www.linuxptp.org/documentation/phc2sys/)
- [HdrHistogram](https://hdrhistogram.github.io/HdrHistogram/)
