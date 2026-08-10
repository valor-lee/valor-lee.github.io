---
title: '[Low Latency Trading] 면접 심화 질문과 답변 — CPU·C++·동시성·Linux·Network·Trading'
date: 2026-08-11 00:10:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    interview,
    computer architecture,
    modern cpp,
    concurrency,
    Linux,
    networking,
    performance
  ]
---

# 개요

초저지연 트레이딩 개발 면접은 용어의 정의에서 끝나지 않는다.

면접관은 대개 한 질문을 다음 방향으로 계속 좁힌다.

```text
정의
  → 실제 내부 동작
  → 언어·하드웨어가 보장하는 경계
  → 성능 비용과 tail latency
  → corner case와 실패 시나리오
  → 구현·측정·검증 방법
```

예를 들어 "atomic이 무엇인가"라는 질문은 `std::atomic`의 사용법보다 다음을 확인하려는 질문일 수 있다.

- Atomicity와 ordering을 구분하는가?
- `happens-before`를 코드에서 증명할 수 있는가?
- x86에서 우연히 동작하는 코드와 portable C++를 구분하는가?
- Cache coherence와 C++ memory model을 같은 것으로 오해하지 않는가?
- Lock-free 자료구조에서 object lifetime과 reclamation까지 고려하는가?

이 글은 암기용 한 줄 답안이 아니다. 각 질문에 대해 먼저 30초에서 1분 안에 말할 수 있는 **핵심 답변**을 제시하고, 이어서 내부 메커니즘, 자주 나오는 꼬리 질문, 저지연 시스템에서의 설계 판단을 정리한다.

전체 구성은 CPU 15문항, Modern C++·동시성 18문항, Linux·Network·측정·Trading 23문항으로 총 56문항이다.

하드웨어 구현은 microarchitecture와 제품 세대에 따라 달라지고, 거래소의 주문·복구 규칙도 venue specification에 따라 달라진다. 따라서 구현 예시는 보편 법칙처럼 말하지 않고, 언어 표준의 보장과 특정 플랫폼에서 흔한 구현을 구분한다.

# 1. 면접 답변을 구성하는 방법

## 1.1 먼저 결론을 말한다

좋은 답변은 핵심 결론을 먼저 말한 뒤 조건과 예외를 붙인다.

```text
질문: relaxed atomic은 무엇을 보장합니까?

결론:
해당 atomic 접근의 원자성은 보장하고, store·RMW에는 객체별
modification order를 부여하지만,
주변 non-atomic 데이터의 publication 순서는 만들지 않습니다.

조건:
주변 데이터를 전달하려면 일반적으로 release/acquire처럼
synchronizes-with 관계를 만드는 수단이 추가로 필요합니다.

검증:
어떤 store 값을 어떤 load가 관측하는지와
happens-before edge를 코드 위에 그려 설명하겠습니다.
```

## 1.2 다섯 층으로 답한다

| 층 | 답해야 할 내용 |
| --- | --- |
| 의미 | 이 개념이 해결하는 문제는 무엇인가? |
| 메커니즘 | CPU, compiler, kernel 또는 protocol 내부에서 무엇이 일어나는가? |
| 보장 | 반드시 참인 것과 구현별로 달라지는 것은 무엇인가? |
| 비용 | 평균뿐 아니라 contention, queueing과 tail에 어떤 비용이 생기는가? |
| 검증 | Test, invariant, counter와 benchmark로 어떻게 확인하는가? |

## 1.3 모르면 경계를 정확히 말한다

정확한 register 수, LLC 정책이나 거래소 recovery 규칙이 기억나지 않을 때 임의로 단정하면 안 된다.

```text
"정확한 LLC inclusion 정책은 CPU 세대별 문서를 확인해야 합니다.
다만 이 질문의 correctness에 필요한 보장은 coherence가 동일 주소의
write serialization과 최신 값 전달을 유지한다는 점입니다."
```

이런 답변은 회피가 아니라 architecture guarantee와 implementation detail을 구분한다는 신호다.

# 2. 이 글의 선수 학습

전체 시스템 순서는 먼저 [[Low Latency Trading] 초저지연 트레이딩 시스템 BFS 학습 로드맵](/posts/low-latency-trading-bfs-roadmap/)에서 확인한다.

| 분야 | 먼저 읽을 글 |
| --- | --- |
| 전체 경로 | [[Low Latency Trading] 초저지연 트레이딩 시스템 전체 구조](/posts/low-latency-trading-system-overview/) |
| CPU 실행 | [[Low Latency Trading] CPU Pipeline·Out-of-Order·Branch Prediction과 ABI](/posts/low-latency-cpu-pipeline-branch-abi/) |
| Cache와 allocation | [[Low Latency Trading] Cache-Friendly Data Layout과 Memory Pool](/posts/low-latency-cache-layout-memory-pool/) |
| CPU 연결 경로 | [[Low Latency Trading] CPU Interconnect와 Uncore — Core·Memory·PCIe 연결 경로](/posts/low-latency-cpu-interconnect-uncore/) |
| Atomic 기초 | [[C언어] Atomic Operation 3. Atomic은 실제로 어떻게 동작하는가](/posts/c-atomic-operation-3/) |
| Memory order | [[C언어] Atomic Operation 4. Memory Order 이해하기](/posts/c-atomic-operation-4/) |
| 동시성 | [[Low Latency Trading] SPSC Ring Buffer와 Thread-per-Core 설계](/posts/low-latency-spsc-thread-per-core/) |
| Linux | [[Low Latency Trading] Linux 실행 환경과 CPU 격리](/posts/low-latency-linux-execution/) |
| Network | [[Low Latency Trading] Market Data가 NIC에서 Application까지 오는 길](/posts/low-latency-network-path/) |
| 측정 | [[Low Latency Trading] Latency를 올바르게 측정하는 방법](/posts/low-latency-measurement/) |
| Trading state | [[Low Latency Trading] Order Gateway와 FIX·Binary Session Recovery](/posts/low-latency-order-gateway-session/) |
| Numerics와 risk | [[Low Latency Trading] Fixed-Point Trading Numerics와 O(1) Pre-Trade Risk](/posts/low-latency-trading-numerics-risk/) |
| 복구와 검증 | [[Low Latency Trading] Binary Event Log·Deterministic Replay와 Fault Injection](/posts/low-latency-event-log-replay/) |

# 3. CPU·Cache·Memory·Compiler와 ABI

## Q1. 현대적인 out-of-order CPU는 명령을 어떻게 실행하는가?

### 면접용 핵심 답변

CPU는 명령을 fetch하고 분기를 예측한 뒤 decode하여 내부 uop으로 만든다. Architectural register를 physical register로 rename해 WAR·WAW 거짓 의존성을 없애고, operand와 실행 unit이 준비된 uop부터 순서와 무관하게 실행한다. 하지만 예외와 architectural state는 프로그램 순서대로 보여야 하므로 reorder buffer에서 in-order로 retire한다. 따라서 **실행 순서와 commit 순서는 다를 수 있다.**

### 내부 동작 상세

- Front-end는 instruction cache/uop cache, branch predictor, decoder를 통해 uop을 공급한다.
- Rename은 거짓 의존성을 없애지만 앞 결과를 실제로 쓰는 RAW dependency는 없애지 못한다.
- Scheduler/reservation station은 준비된 uop을 ALU, load/store, branch, vector port에 issue한다.
- ROB는 정확한 exception과 잘못된 speculation 복구를 위해 결과를 프로그램 순서로 retire한다.
- 오래 걸리는 cache miss가 ROB head를 막거나 load buffer·ROB가 가득 차면 독립 작업이 있어도 정체된다.

### 꼬리 질문과 짧은 답

- **Out-of-order면 프로그램 결과 순서도 바뀌는가?** 단일 thread의 정의된 observable behavior는 유지된다. 다른 thread의 관찰은 ISA·C++ memory model로 판단한다.
- **Rename이 없애지 못하는 의존성은?** 실제 data를 전달하는 RAW dependency다.
- **IPC가 낮으면 CPU가 나쁜가?** 아니다. dependency 또는 memory latency가 원인일 수 있어 assembly와 PMU를 함께 봐야 한다.

### 저지연 실무 연결

Pointer chain은 load 병렬성을 막고, 예측하기 어려운 branch는 younger work를 폐기시킨다. 명령 수만 줄이지 말고 front-end, bad speculation, execution-port pressure, memory bound 중 어디가 병목인지 `perf`와 annotated assembly로 확인한다.

---

## Q2. Store가 retire되었다는 것과 다른 코어에 보인다는 것은 같은가?

### 면접용 핵심 답변

같지 않다. Retire는 store가 이전 exception이나 잘못된 speculation 때문에 취소되지 않고 architectural execution에 포함되었다는 뜻이다. Store의 주소와 data가 준비되고 retire되어도 store buffer에서 cache-line write ownership이나 cache port를 기다릴 수 있다. 다른 코어가 값을 관찰하는 시점은 coherence, ISA ordering, fence와 C++ synchronization을 함께 봐야 한다.

### 내부 동작 상세

- Store queue/buffer는 in-flight store를 보관해 ownership 획득을 기다리는 동안 CPU가 진행하게 한다.
- 같은 코어의 뒤 load는 cache 반영 전에도 store-to-load forwarding으로 값을 받을 수 있다.
- 따라서 “내 코어가 새 값을 읽음”은 “다른 코어도 이미 관찰 가능함”을 뜻하지 않는다.
- Fence는 요구된 memory ordering을 만족하도록 진행을 제한하지만 보통 dirty cache를 DRAM까지 flush하는 명령은 아니다.
- 구체적인 store queue와 buffer 분리는 CPU별 구현 사항이며 답은 architectural effect 중심으로 해야 한다.

### 꼬리 질문과 짧은 답

- **Store buffer가 비어야 retire하는가?** 보통 아니다. Retire와 visibility를 분리하는 것이 buffer 목적 중 하나다.
- **x86이면 plain variable로 thread 통신해도 되는가?** 아니다. C++ data race는 undefined behavior다.
- **`mfence`는 persistent storage까지 기록하는가?** 아니다. Ordering fence와 cache write-back·persistence protocol은 별개다.

### 저지연 실무 연결

Payload를 쓰고 ready flag를 세우는 publication은 release store와 acquire load로 happens-before를 만들어야 한다. 임의 fence를 많이 넣으면 store 진행과 speculation을 제한해 tail latency를 키우므로, ownership protocol을 단순화하고 필요한 최소 ordering을 증명한다.

---

## Q3. L1·L2·LLC와 inclusive/exclusive/non-inclusive cache는 무엇인가?

### 면접용 핵심 답변

일반적으로 L1은 core별 instruction/data cache로 작고 빠르며, L2는 더 크고 대개 private, LLC는 core들이 공유하거나 여러 slice로 분산된다. Inclusive는 CPU에 가까운 cache level의 line이 더 아래 level에도 포함되는 관계, exclusive는 level 간 중복을 줄이는 정책, non-inclusive/non-exclusive는 어느 관계도 보장하지 않는 정책이다. 실제 hierarchy와 정책은 CPU 제품별로 다르다.

### 내부 동작 상세

- Cache는 대개 line 단위로 data를 옮기며 set-associative 구조에서 index로 set을 고르고 tag를 비교한다.
- Entry에는 data뿐 아니라 tag, valid/dirty, coherence metadata가 있다.
- Inclusive LLC는 private-cache 존재 추적에 이점이 있지만 LLC eviction이 private line invalidation을 부를 수 있다.
- Non-inclusive LLC라도 별도 directory/snoop filter로 sharer를 추적할 수 있다.
- Shared LLC도 하나의 균일한 배열이 아니라 address hashing된 slice와 interconnect로 구성될 수 있다.

### 꼬리 질문과 짧은 답

- **Cache line은 항상 64 byte인가?** 흔한 x86 서버에서는 그렇지만 CPU별 차이가 있고 C++ 표준 보장은 아니다.
- **LLC hit latency는 고정인가?** 아니다. Slice 위치, coherence 상태와 interconnect queueing에 따라 달라질 수 있다.
- **LLC에 없으면 어느 L1에도 없는가?** Inclusive라고 확인된 설계에서만 가능한 추론이다.

### 저지연 실무 연결

“자료구조가 L1 크기보다 작다”만으로 L1 resident를 보장하지 않는다. Set conflict, code/data working set, SMT 경쟁과 replacement를 고려해야 한다. 대상 CPU의 topology를 확인하고 cache latency를 단일 숫자가 아닌 분포로 측정한다.

---

## Q4. Cache에 없던 주소를 load하면 cache line은 어떻게 들어오는가?

### 면접용 핵심 답변

CPU는 virtual address를 translation하면서 L1D tag를 조회한다. L1 miss면 MSHR 계열 miss-tracking entry를 할당하고 lower cache로 요청을 보낸다. 요청은 L2, LLC/home agent·directory를 거쳐 peer cache의 최신 사본 또는 memory controller의 DRAM에서 응답받는다. Data는 보통 line 단위로 hierarchy를 채우고 기다리던 load를 깨운다. 구체적인 병렬 lookup과 fill 경로는 구현마다 다르다.

### 내부 동작 상세

- VIPT L1 cache는 page-offset index를 사용해 TLB lookup과 cache set 접근 일부를 겹칠 수 있다.
- 같은 line miss가 이미 진행 중이면 새 요청을 보내지 않고 기존 miss에 합칠 수 있다.
- 다른 core가 Modified data를 가지면 그 core/coherence agent가 최신 data를 공급할 수 있다.
- Miss-tracking, load-buffer, fill-buffer entry는 유한하므로 많은 miss가 생기면 memory-level parallelism이 포화된다.
- Unaligned access가 두 line 또는 두 page를 가로지르면 요청과 translation이 각각 둘 필요할 수 있다.

### 꼬리 질문과 짧은 답

- **L1 miss면 L2를 다 기다린 뒤 LLC를 보는가?** 논리적 hierarchy와 실제 pipeline은 다르다. CPU는 lookup과 요청을 겹칠 수 있다.
- **다른 core의 L1에서 직접 오는가?** 가능하지만 source와 route는 coherence 상태·directory·cache 정책에 달렸다.
- **Miss는 무한히 병렬 처리되는가?** 아니다. MSHR와 queue entry 수에 제한된다.

### 저지연 실무 연결

Linked node의 dependent pointer chase는 다음 주소를 앞 load 완료 뒤에야 알 수 있다. Order book·risk table을 compact contiguous layout과 index 기반 참조로 바꾸면 line 수와 dependency를 함께 줄일 수 있다. Warm-cache microbenchmark와 production burst를 분리해 측정한다.

---

## Q5. 다른 코어가 보유한 cache line에 store하면 어떤 일이 일어나는가?

### 면접용 핵심 답변

Core A와 B가 line을 Shared로 가진 상황에서 B가 store하려면 write ownership을 얻어야 한다. B는 흔히 RFO라고 부르는 ownership 요청을 보내고 directory 또는 snooping이 A 등 sharer에 invalidation을 전달한다. 필요한 acknowledgement 후 B가 독점 권한을 얻으면 store를 반영하고 line은 논리적으로 Modified 계열 상태가 된다. A가 다시 읽으면 coherence miss로 최신 data를 공급받는다. MESI/MOESI는 설명 모델이며 실제 메시지·transient state·data source는 구현마다 다르다.

### 내부 동작 상세

- B가 clean copy를 가졌어도 S→M 전환에는 다른 sharer를 무효화하는 upgrade가 필요하다.
- A가 Modified owner라면 memory는 오래된 값일 수 있어 A가 data를 forward/write-back하며 ownership을 넘긴다.
- MOESI의 Owned는 dirty data를 memory에 즉시 쓰지 않고 owner가 shared copy에 공급할 수 있게 한다.
- Directory 방식은 sharer/owner metadata로 대상을 찾고, snoop 방식은 fabric의 요청을 cache들이 관찰한다.
- Coherence는 주로 한 location의 write를 직렬화하며 서로 다른 line의 publication 순서를 자동 보장하지 않는다.

### 꼬리 질문과 짧은 답

- **RFO는 반드시 DRAM read인가?** 아니다. Data는 local/peer cache나 LLC에서 올 수 있다.
- **E와 M 차이는?** E는 memory와 같은 clean data, M은 memory보다 최신인 dirty data다.
- **Line이 물리적으로 하나만 이동하는가?** Data transfer와 metadata 전환이다. Clean shared copy는 여러 개 존재할 수 있다.

### 저지연 실무 연결

여러 worker가 같은 counter·queue metadata를 갱신하면 ownership이 core/socket 사이를 왕복한다. Sharded counter, single-writer, batch publish, read-mostly snapshot으로 sharing 자체를 줄이는 것이 fence 미세 조정보다 효과적인 경우가 많다.

---

## Q6. False sharing은 무엇이고 atomic 변수에서도 왜 발생하는가?

### 면접용 핵심 답변

False sharing은 논리적으로 독립된 변수가 같은 coherence line에 놓여 서로 다른 core가 각각 쓸 때 line 전체 ownership이 ping-pong하는 현상이다. 변수 간 data dependency가 없어도 invalidation과 cache-to-cache traffic이 생긴다. Atomic은 data race와 원자성을 다루지만 coherence granularity를 줄이지 않으므로 false sharing을 해결하지 않는다.

### 내부 동작 상세

```cpp
struct alignas(64) PaddedCounter { std::atomic<std::uint64_t> value{0}; };
struct Counters { PaddedCounter producer; PaddedCounter consumer; };
```

- `alignas` 시작 주소뿐 아니라 type 크기, 배열 stride와 allocator placement를 확인해야 한다.
- 64 byte는 흔한 x86 값일 뿐 대상 CPU에서 확인해야 하며, 지원되면 `std::hardware_destructive_interference_size`도 검토한다.
- 무조건 padding하면 working set과 TLB pressure가 증가하므로 write frequency가 높은 field에 선택적으로 적용한다.
- 여러 core가 읽기만 하는 clean shared copy는 보통 false-sharing ping-pong을 만들지 않는다.

### 꼬리 질문과 짧은 답

- **Non-atomic이면 사라지는가?** 아니다. 서로 다른 object라 data race가 없어도 같은 line write는 ping-pong한다.
- **`alignas(64)`면 무조건 해결되는가?** 아니다. Field의 끝과 인접 object 배치까지 봐야 한다.
- **어떻게 확인하는가?** Affinity를 고정하고 CPU별 cache-to-cache/HITM 계열 counter와 padding A/B test를 본다.

### 저지연 실무 연결

SPSC ring의 producer-owned tail과 consumer-owned head를 서로 다른 line에 두고, 상대 index는 local copy에 cache해 공유 line read 빈도도 줄인다. Padding 전후에는 throughput뿐 아니라 p99.9와 전체 working-set 증가를 함께 측정한다.

---

## Q7. Store buffer, store-to-load forwarding, load/store disambiguation은 어떻게 연결되는가?

### 면접용 핵심 답변

Store buffer는 store가 cache ownership을 기다리는 동안 뒤 명령이 진행되도록 한다. Younger load가 아직 cache에 반영되지 않은 older store와 겹치면 buffer에서 값을 직접 받는 것이 store-to-load forwarding이다. 한편 older store의 주소가 아직 불명확할 때 CPU는 younger load와 alias하지 않는다고 예측해 먼저 실행할 수 있으며, 나중에 alias가 밝혀지면 load와 dependent uop을 replay한다. 이 추적을 load/store queue와 memory disambiguator가 담당한다.

### 내부 동작 상세

- 주소·크기·정렬이 forwarding 조건에 맞지 않거나 partial overlap이면 stall/replay가 생길 수 있다.
- Store-buffer, load-buffer와 miss-tracking entry가 가득 차면 dispatch 또는 retirement가 정체된다.
- 보수적 alias 판단은 불필요한 대기를, 공격적 예측 실패는 memory-order violation replay를 만든다.
- Virtual address의 같은 page offset 때문에 physical address 확정 전 잠정 alias하는 4K-alias 계열 penalty도 가능하다.
- 정확한 forwarding 조건, predictor와 PMU event는 microarchitecture마다 다르다.

### 꼬리 질문과 짧은 답

- **Store buffer는 write-back cache인가?** 아니다. In-flight store를 추적하는 core 내부 자원이다.
- **내 store 직후 같은 주소 load는 L1에서 읽는가?** Buffer forwarding으로 받을 수 있다.
- **`__restrict`가 hardware disambiguation을 없애는가?** Compiler alias 최적화에는 도움을 주지만 runtime queue가 불필요해진다는 보장은 없다.

### 저지연 실무 연결

Overlapping field access, unaligned wide load, 긴 address-generation chain은 예상 밖 replay를 만든다. Assembly와 CPU별 PMU를 확인하고 구조체 layout·alignment를 바꿔 검증한다. 많은 shared store 뒤에 fence를 두기보다 single-writer와 batch notification으로 store pressure를 줄인다.

---

## Q8. Branch prediction과 speculative execution은 latency에 어떤 영향을 주는가?

### 면접용 핵심 답변

CPU는 direction predictor, branch-target buffer, return-stack 계열 구조로 다음 경로를 예측하고 미리 실행한다. 맞으면 pipeline을 채운 채 진행하지만 틀리면 younger speculative work를 버리고 올바른 target에서 다시 fetch한다. Penalty는 고정 숫자가 아니라 branch condition이 준비되는 시점과 front-end 복구에 달렸다. Branchless code도 extra instruction과 긴 dependency 때문에 더 느릴 수 있다.

### 내부 동작 상세

- Conditional branch는 address와 history의 pattern을, indirect branch는 target도 예측해야 한다.
- Condition 계산이 cache miss에 의존하면 잘못된 경로에서 더 많은 uop·cache bandwidth를 소비할 수 있다.
- Misprediction 시 rename/checkpoint 상태를 복구하고 instruction stream을 다시 공급한다.
- Speculative work는 retire되지 않아도 cache/TLB/predictor 상태에 흔적을 남길 수 있어 Spectre 계열 문제의 배경이 된다.
- `cmov`는 mispredict를 없앨 수 있지만 condition과 data의 dependency를 유지하고 양쪽 값 계산이 필요할 수 있다.

### 꼬리 질문과 짧은 답

- **50:50 branch면 예측률도 50%인가?** 아니다. 전체 빈도보다 history와 상관관계가 중요하다.
- **Virtual call은 예측 불가능한가?** Target이 안정적이면 예측될 수 있지만 devirtualization·inline 기회를 잃을 수 있다.
- **Mispredict 수가 줄면 반드시 빨라지는가?** 아니다. Instruction 수, code footprint와 dependency가 이득을 상쇄할 수 있다.

### 저지연 실무 연결

시장 regime에 따라 branch 분포가 급변하므로 균일 random이나 고정 입력 benchmark는 위험하다. Rare error path를 hot code에서 분리하고 production profile로 PGO를 검토하되, I-cache/uop-cache와 실제 latency 분포로 결과를 확인한다.

---

## Q9. TLB miss, page walk, page fault와 huge page는 어떻게 연결되는가?

### 면접용 핵심 답변

TLB miss는 address translation이 TLB에 없어 page table을 조회해야 한다는 뜻이며 page fault와 같지 않다. Hardware page walk로 valid PTE를 찾으면 kernel exception 없이 해결될 수 있고, not-present·권한·COW 처리 등이 필요할 때 page fault가 난다. Huge page는 한 TLB entry의 coverage를 늘리고 page-walk level을 줄일 수 있지만 allocation·단편화·NUMA placement·promotion/split 비용 때문에 항상 빠르지는 않다.

### 내부 동작 상세

- L1 TLB, higher/shared TLB에 miss하면 page-table root부터 여러 level의 entry를 따라간다.
- PTE는 일반 cache와 page-walk cache에 있을 수 있어 모든 walk가 DRAM 여러 번을 뜻하지 않는다.
- Mapping 변경은 다른 core의 stale translation을 없애는 TLB shootdown과 IPI를 유발할 수 있다.
- 2 MiB page는 같은 범위에 필요한 entry 수를 줄이지만 huge-page 전용 TLB entry 수가 적을 수 있다.
- THP의 background collapse/compaction은 편의성과 맞바꿔 latency jitter를 만들 수 있다.

### 꼬리 질문과 짧은 답

- **Minor fault면 hot path에서 괜찮은가?** Kernel 진입, allocation·zeroing과 lock 때문에 큰 jitter가 가능하다.
- **`mlockall`이면 fault가 전부 사라지는가?** 향후 allocation, stack growth와 새 mapping까지 자동 해결하지는 않는다.
- **Huge page면 data-cache miss도 줄어드는가?** 직접 효과는 translation이다. Data locality가 나쁘면 cache miss는 그대로다.

### 저지연 실무 연결

기동 시 pool과 stack을 확보·pre-touch하고 hot path의 mapping 변경을 피한다. 큰 dense ring은 explicit huge page 후보지만 모든 allocation에 쓰지는 않는다. dTLB/page-walk counter와 minor/major fault를 분리하고 THP 상태와 실제 mapping page size를 확인한다.

---

## Q10. NUMA에서 local·remote memory access는 왜 차이가 나는가?

### 면접용 핵심 답변

NUMA 시스템은 socket/node마다 가까운 memory controller와 DRAM이 있고, 다른 node의 memory 접근은 CPU interconnect를 거친다. Remote access는 보통 더 높은 latency와 fabric queue 경쟁을 가진다. Thread를 CPU에 pinning해도 이미 다른 node에 배치된 page가 자동 이동하지 않으므로 **CPU, memory, NIC queue locality를 함께 맞춰야 한다.** 정확한 home-agent·routing 구조는 제품별로 다르다.

### 내부 동작 상세

- Linux first-touch에서는 대개 page를 처음 실제 touch한 CPU의 node에 physical page가 배치된다.
- Remote DRAM 요청은 inter-socket link와 remote memory controller를 사용한다.
- 최신 data가 peer cache에 있으면 remote cache-to-cache/coherence 경로를 타므로 단순 DRAM 모델과 다르다.
- Shared line의 home/directory 위치와 fabric congestion도 latency에 영향을 준다.
- Automatic NUMA balancing의 sampling fault와 page migration이 latency SLA에 맞는지는 별도로 판단한다.

### 꼬리 질문과 짧은 답

- **Affinity만 설정하면 해결되는가?** 아니다. Page와 IRQ/RX queue 위치도 확인해야 한다.
- **`malloc`을 호출한 node에 배치되는가?** Lazy allocation이면 실제 first touch가 더 중요할 수 있다.
- **Interleave 정책은 항상 좋은가?** Bandwidth 분산에는 유리할 수 있지만 latency-critical thread의 remote access를 늘릴 수 있다.

### 저지연 실무 연결

NIC RX queue는 node 0인데 polling thread와 buffer pool은 node 1이면 DMA descriptor와 payload가 fabric을 오간다. IRQ/RSS queue, polling core, huge-page pool과 strategy thread를 topology에 맞추고 `/proc/interrupts`, sysfs와 `numa_maps`로 실제 배치를 검증한다.

---

## Q11. NIC DMA와 CPU cache는 어떻게 일관성과 순서를 유지하는가?

### 면접용 핵심 답변

NIC는 descriptor가 가리키는 host memory에 DMA하고 completion/ownership 정보를 갱신한다. 적절한 coherent DMA mapping과 device ownership protocol을 지키는 플랫폼에서는 coherence fabric이 CPU cache 복사본과 device transaction의 일관성을 유지한다. Non-coherent mapping에서는 driver가 DMA sync API로 cache clean/invalidate와 ownership 전환을 명시해야 한다. 어느 경우든 TX의 data·descriptor write → doorbell 순서와 RX의 completion 관측 → payload read 순서를 **device용 memory primitive**로 보장해야 하며 C++ `volatile`만으로는 부족하다.

### 내부 동작 상세

- RX: CPU가 buffer descriptor 게시 → NIC가 payload DMA → status/completion 기록 → CPU가 barrier/sync 후 payload 처리.
- TX: CPU가 payload·descriptor 작성 → write ordering 보장 → MMIO doorbell → NIC 처리 → 별도 completion 확인.
- PCIe MMIO write는 posted일 수 있어 doorbell store의 실행을 packet 전송 완료로 해석할 수 없다.
- IOMMU 사용 시 device I/O virtual address translation과 IOTLB miss·mapping 비용이 추가될 수 있다.
- Intel DDIO처럼 DMA를 LLC와 결합하는 기능은 vendor·제품·설정 의존적이며 일반화하면 안 된다.

### 꼬리 질문과 짧은 답

- **DMA 뒤 CPU가 stale L1을 읽을 수 있는가?** 올바른 coherent mapping·completion protocol에서는 fabric이 private-cache 복사본을 조정한다. Non-coherent mapping에서는 정해진 DMA sync API가 필요하다.
- **MMIO를 `volatile` pointer로 쓰면 충분한가?** 아니다. Compiler access 보존과 CPU/device ordering은 별개다.
- **Doorbell 뒤 fence면 TX 완료인가?** 아니다. Completion descriptor나 device-specific status가 필요하다.

### 저지연 실무 연결

Kernel bypass도 ordering을 없애지 않는다. Framework의 descriptor ownership, DMA barrier, doorbell batching과 buffer recycling 규칙을 지킨다. RX queue·polling core·pool의 NUMA locality와 IOMMU/DDIO 설정을 median과 tail 양쪽에서 측정한다.

---

## Q12. Cache coherence, hardware memory consistency, C++ memory model은 어떻게 다른가?

### 면접용 핵심 답변

Cache coherence는 같은 memory location의 write가 하나의 coherence order로 관찰되게 하고 cache-line 복사본과 ownership을 조정한다. 이것이 모든 core가 매 순간 같은 최신 값을 읽거나 서로 다른 location의 접근 순서까지 자동 보장한다는 뜻은 아니다. Hardware memory consistency는 서로 다른 location을 포함한 load/store가 observer에게 어떤 순서로 보일 수 있는지 정의한다. C++ memory model은 compiler와 모든 표적 하드웨어 위의 language 규약으로 data race, modification order, synchronizes-with와 happens-before를 정의한다. 따라서 coherent cache가 plain non-atomic publication을 안전하게 만들지는 않는다.

### 내부 동작 상세

```cpp
int payload;
std::atomic<bool> ready{false};
// producer: payload = 42; ready.store(true, std::memory_order_release);
// consumer: if (ready.load(std::memory_order_acquire)) use(payload);
```

- Acquire가 release가 쓴 값을 관찰하면 synchronizes-with가 생겨 payload write가 consumer read보다 happens-before가 된다.
- `relaxed`도 atomic object의 atomicity와 modification order는 보장하지만 payload publication ordering은 만들지 않는다.
- Compiler는 source-level 보장을 x86·ARM 등 ISA에 맞는 reordering 제한과 instruction으로 mapping한다.
- Coherence granule 전송은 여러 field에 대한 language-level atomic snapshot을 보장하지 않는다.

### 꼬리 질문과 짧은 답

- **x86에서 relaxed flag도 우연히 동작하지 않는가?** 특정 codegen에서는 그럴 수 있어도 C++ correctness 근거가 아니다.
- **`seq_cst`면 algorithm이 자동으로 맞는가?** 아니다. Lifetime, ABA와 invariant 오류는 남는다.
- **Coherence와 consistency는 같은가?** 아니다. 전자는 주로 per-location, 후자는 여러 access의 관찰 순서를 다룬다.

### 저지연 실무 연결

SPSC queue는 “cache가 최신 값을 준다”가 아니라 payload write → release index → acquire index → payload read의 happens-before를 그려 증명한다. 그 뒤 별도로 false sharing과 ownership 이동 비용을 분석한다.

---

## Q13. `volatile`, compiler barrier, CPU memory barrier는 무엇이 다른가?

### 면접용 핵심 답변

C++ `volatile`은 volatile object access를 compiler가 일반 access처럼 제거·합치지 못하게 하지만 thread 간 atomicity나 happens-before를 만들지 않는다. Compiler barrier는 compiler 재배치를 제한하되 CPU ordering을 바꾸지 않을 수 있다. CPU barrier는 ISA 차원에서 load/store ordering을 강제하지만 올바른 compiler constraint가 함께 필요하다. Thread 통신에는 C++ atomic/mutex, MMIO에는 OS·driver accessor와 device barrier를 쓴다.

### 내부 동작 상세

- As-if rule에 따라 compiler는 정의된 observable behavior가 같으면 명령을 제거·재배치할 수 있다.
- `volatile` access의 의미는 구현 규칙을 따르지만 주변 non-volatile payload publication을 보장하지 않는다.
- Inline assembly의 memory clobber 같은 compiler barrier는 hardware fence instruction을 생성하지 않을 수 있다.
- `dmb`, `mfence` 같은 CPU barrier도 compiler intrinsic/atomic API를 통해 compiler semantics와 함께 써야 한다.
- Normal cacheable memory와 device memory 사이 ordering은 generic thread fence와 다를 수 있다.

### 꼬리 질문과 짧은 답

- **`volatile bool ready`로 spin하면?** Atomicity·ordering·data-race freedom이 없어 C++에서는 잘못된 통신이다.
- **Benchmark 결과를 volatile에 저장하면 충분한가?** Dead-code 제거 일부만 막을 뿐 측정 ordering과 codegen 왜곡이 남는다.
- **MMIO에 `std::atomic`을 쓰면 되는가?** Device register의 width·side effect·ordering은 일반 atomic과 다르므로 플랫폼 API를 따른다.

### 저지연 실무 연결

직접 fence를 지우고 volatile로 바꾸는 최적화는 간헐적 corruption을 만든다. 먼저 shared ownership과 publication 횟수를 줄이고, correctness를 증명할 수 있는 가장 약한 C++ atomic order를 사용한다. Device path는 framework가 제공하는 barrier를 쓴다.

---

## Q14. Undefined behavior는 왜 최적화된 저지연 코드에서 특히 위험한가?

### 면접용 핵심 답변

Undefined behavior는 단순히 해당 instruction이 이상한 값을 낼 수 있다는 뜻이 아니다. Compiler는 정의된 program에서는 그 상황이 없다고 가정해 주변 code를 제거·재배치할 수 있다. 저지연 C++에서 흔한 예는 signed overflow, data race, lifetime 종료 뒤 access, out-of-bounds, strict-aliasing 위반과 misaligned pointer dereference다. x86 instruction이 우연히 실행 가능해도 C++ program이 정의되는 것은 아니다.

### 내부 동작 상세

- Signed overflow가 없다는 가정으로 조건·loop가 변형될 수 있다. Wrap이 의도라면 적절한 unsigned/checked arithmetic을 쓴다.
- Conflicting non-atomic access에 happens-before가 없으면 data race이며 cache coherence가 고쳐주지 않는다.
- Type punning은 `reinterpret_cast`만으로 합법화되지 않는다. 조건에 맞는 `std::memcpy`/`std::bit_cast`를 쓴다.
- Storage 존재와 object lifetime 시작은 다르며 placement construction/destruction 규칙을 지켜야 한다.
- Misaligned dereference는 language UB일 수 있고, line/page crossing과 atomicity penalty도 별개로 존재한다.

### 꼬리 질문과 짧은 답

- **`-fno-strict-aliasing`이면 해결되는가?** Alias assumption 일부만 완화하며 lifetime·alignment·race UB는 남는다.
- **Unsigned overflow도 UB인가?** 정수 unsigned arithmetic은 modulo로 정의되지만 algorithm상 의도 여부는 별개다.
- **Sanitizer면 전부 찾는가?** 아니다. 매우 유용하지만 coverage 한계가 있고 timing/layout도 바꾼다.

### 저지연 실무 연결

Wire bytes를 packed struct pointer로 바로 cast하면 alignment, lifetime, aliasing과 endian 문제가 겹친다. 경계에서 안전하게 decode해 host representation으로 바꾸고 hot data를 정렬된 구조에 둔다. “복사 한 번 제거”보다 UB와 cross-line access의 tail 위험이 더 클 수 있다.

---

## Q15. ABI와 calling convention은 저지연 C++ 성능에 어떤 영향을 주는가?

### 면접용 핵심 답변

ABI는 argument와 return value의 register/stack 배치, caller/callee-saved register, stack alignment, object layout, name mangling과 exception unwinding을 정한다. 예를 들어 System V AMD64는 일부 정수·pointer argument를 register로 넘기지만 aggregate는 크기와 분류에 따라 register 또는 memory를 쓴다. 따라서 “값 전달은 항상 copy라 느리고 reference는 항상 빠르다”는 답은 틀리며 실제 ABI와 inlining 후 codegen을 봐야 한다.

### 내부 동작 상세

- Caller는 필요한 caller-saved 값을 보존하고, callee는 사용하는 callee-saved register를 저장·복구한다.
- 작은 trivially-copyable aggregate는 register로 분해될 수 있고 큰 return은 hidden result pointer를 쓸 수 있다.
- Inlining은 call과 ABI 이동을 없애고 constant propagation을 돕지만 code size와 I-cache pressure를 키울 수 있다.
- Virtual dispatch는 흔히 vptr를 통한 indirect call이지만 compiler가 dynamic type을 알면 devirtualize할 수 있다.
- Table 기반 exception은 정상 경로 비용을 낮출 수 있으나 실제 throw/unwind는 비싸고 구체적 비용은 구현별이다.

### 꼬리 질문과 짧은 답

- **작은 struct도 reference가 빠른가?** 아니다. 값은 register로 전달될 수 있고 reference는 memory load·alias 가능성을 더할 수 있다.
- **Red zone은 항상 쓸 수 있는가?** 특정 user-space ABI의 규칙이며 Windows ABI나 kernel에는 그대로 적용되지 않는다.
- **강제 inline이면 빨라지는가?** Call overhead는 줄지만 code bloat가 I-cache/uop-cache를 악화시킬 수 있다.

### 저지연 실무 연결

작은 parser/state-transition 함수는 inlining 이득이 있을 수 있지만 거대한 template specialization은 hot code footprint를 폭증시킨다. `objdump`, optimization report, symbol size와 I-cache/ITLB counter를 함께 보고 API는 추상적인 copy 금지가 아니라 실제 register passing과 ownership 명확성으로 설계한다.

# 4. Modern C++·Object Lifetime·Memory Model과 동시성

## Q1. RAII란 정확히 무엇이며 자동 메모리 해제와 무엇이 다른가?

### 면접용 핵심 답변

RAII는 **자원의 소유권을 객체 수명에 결합**하는 설계 기법이다. 생성이 성공하면 클래스 불변식과 자원 소유권이 성립하고, 정상적인 scope 종료나 exception unwinding으로 객체 수명이 끝나면 소멸자가 자원을 확정적으로 반납한다. 자원은 힙 메모리뿐 아니라 mutex, file descriptor, socket, mapping, thread, 임시 상태 변경도 포함한다.

GC와 달리 automatic 객체는 scope 종료나 stack unwinding 시점에 파괴되므로 비메모리 자원을 즉시 반납할 수 있다. 객체가 반드시 stack에 있어야 하는 것은 아니며, 본질은 storage 위치가 아니라 lifetime과 ownership의 결합이다. `std::terminate`, `_Exit`, process 강제 종료처럼 stack unwinding이 일어나지 않는 경로에는 automatic 객체의 소멸을 기대할 수 없으므로 영속 상태와 crash recovery는 별도로 설계한다.

### 내부 동작 상세

```cpp
class Fd {
    int fd_;
public:
    explicit Fd(int fd) noexcept : fd_(fd) {}
    ~Fd() { if (fd_ >= 0) ::close(fd_); }
    Fd(const Fd&) = delete;
    Fd& operator=(const Fd&) = delete;
    Fd(Fd&& x) noexcept : fd_(std::exchange(x.fd_, -1)) {}
};
```

### 꼬리 질문과 짧은 답

- **예외를 쓰지 않아도 유용한가?** 조기 반환, 다중 종료 경로, 소유권 표현에도 유용하다.
- **RAII면 비용이 없는가?** 추상화 자체는 사라질 수 있지만 소멸자가 수행하는 `close`, free, lock 해제 비용은 실제다.
- **`lock_guard`가 RAII인 이유는?** 생성 시 잠그고 소멸 시 반드시 풀어 lock 수명을 scope와 결합한다.

### 저지연 실무 연결

RAII를 제거할 게 아니라 hot path에서 어떤 소멸자가 실행되는지 확인해야 한다. 마지막 `shared_ptr` 해제, blocking I/O, 큰 deallocation은 제어 경로나 전용 reclaim thread로 분리한다.

---

## Q2. 생성자 실행 중 예외가 나면 무엇이 파괴되며, 소멸자가 예외를 던지면 어떻게 되는가?

### 면접용 핵심 답변

일반적인 non-delegating 생성자가 실패하면 가장 바깥 객체가 완전히 생성되지 않았으므로 **그 객체 자신의 소멸자는 호출되지 않는다**. 이미 생성이 끝난 base와 member만 생성의 역순으로 파괴되고, 생성자 body의 지역 RAII 객체도 unwinding으로 파괴된다. 선언 순서가 member 생성 순서를 정하며 initializer list 표기 순서는 이를 바꾸지 않는다. 예외적으로 delegating 생성자는 target 생성자가 완료된 뒤 자신의 body에서 예외가 나면 완성된 객체의 소멸자가 호출된다.

일반적인 소멸자는 암시적으로 `noexcept(true)`다. 정확히는 base/member 소멸자의 exception specification에 따라 조건부로 결정된다. non-throwing 소멸자 밖으로 예외가 나오거나, 다른 예외의 unwinding 중 두 번째 예외가 빠져나오면 `std::terminate`가 호출된다.

### 내부 동작 상세

`Socket`, `Buffer`, `Parser` 순으로 선언된 객체에서 `Parser` 생성이 실패하면 `Buffer`, `Socket`만 역순 파괴된다. 생성자에서 raw handle을 얻고 RAII member에 넘기기 전에 실패하면 누수될 수 있으므로 획득 즉시 소유 객체에 담아야 한다.

### 꼬리 질문과 짧은 답

- **중요한 `close` 실패는 어떻게 보고하나?** 명시적인 `flush/close` API로 반환하고 소멸자는 best effort cleanup만 한다.
- **`noexcept` 함수에서 예외가 나면?** 함수 밖으로 전파되려는 순간 종료된다.
- **배열 member 중 일부만 생성됐다면?** 완성된 element만 역순 파괴된다.

### 저지연 실무 연결

초기 NIC queue, huge-page mapping, shared memory 구성은 member별 RAII로 부분 실패를 안전하게 정리한다. hot path에서는 사전 할당과 명시적 status 반환으로 실패 가능 경로를 통제한다.

---

## Q3. Basic, strong, no-throw exception guarantee를 설명하라.

### 면접용 핵심 답변

- **Basic**: 실패 후에도 자원 누수가 없고 invariant는 유지되지만 값은 일부 바뀔 수 있다.
- **Strong**: 실패하면 연산 전과 같은 관찰 가능한 상태다. commit-or-rollback 의미다.
- **No-throw**: 연산이 실패를 보고하는 예외를 외부로 던지지 않는다는 exception-safety 계약이다. 단순히 함수에 `noexcept`를 붙였다는 사실만으로 정상 완료가 보장되는 것은 아니며, 실제로 예외가 빠져나오면 `std::terminate`가 호출된다.

Strong guarantee는 보통 비공개 임시 상태에 실패 가능 작업을 끝낸 뒤 `noexcept` swap/commit으로 공개한다. 다만 이미 NIC에 주문을 보낸 것 같은 외부 side effect는 메모리 rollback만으로 취소할 수 없다.

### 내부 동작 상세

```cpp
Book next = current;        // 실패 가능
next.apply(update);         // 실패 가능
current.swap(next);         // noexcept commit
```

전체 book copy는 hot path에 비싸므로 실제로는 prevalidation, bounded storage, undo log, double buffering, immutable snapshot 등으로 같은 불변식을 더 싸게 구현한다. 예외 대신 `expected`를 써도 실패 시 상태 계약은 여전히 필요하다.

### 꼬리 질문과 짧은 답

- **모든 함수에 strong guarantee가 최선인가?** 비용과 시스템 의미에 따라 basic+recovery가 더 적합할 수 있다.
- **`swap`이 왜 `noexcept`여야 하나?** commit 단계가 실패하면 원상 복구를 보장하기 어렵다.
- **외부 side effect는?** journal, idempotency key, 상태 머신, reconciliation로 복구한다.

### 저지연 실무 연결

주문 흐름은 `risk reservation → journal/publication → send → acknowledgement` 단계별로 crash와 중복을 정의해야 한다. “메모리 상태만 strong guarantee”라는 답으로는 부족하다.

---

## Q4. Rule of Zero/Five와 `noexcept` move는 왜 중요한가?

### 면접용 핵심 답변

Rule of Zero는 자원을 표준 RAII member에 맡겨 destructor/copy/move를 직접 쓰지 않는 원칙이다. 직접 소유권을 구현한다면 destructor, copy constructor/assignment, move constructor/assignment 다섯 연산을 함께 검토해야 한다.

이동 연산의 `noexcept`는 container의 exception safety와 성능에 영향을 준다. `vector` 재할당에서 move가 던질 수 있고 copy가 가능하다면 구현은 strong guarantee를 유지하려고 copy를 선택할 수 있다. non-throwing move면 안전하게 소유권을 이동할 수 있다.

### 내부 동작 상세

`std::move`는 실제 이동이 아니라 대체로 rvalue로 cast할 뿐이다. 선택된 move constructor가 소유권을 옮긴다. 이동된 원본은 보통 **valid but unspecified** 상태로, 소멸과 재대입 등 타입 계약이 허용하는 연산은 가능해야 한다. Move가 항상 O(1)인 것도 아니다. inline array의 move는 element 수에 비례할 수 있다.

### 꼬리 질문과 짧은 답

- **`const T&&`에서 왜 대개 move가 안 되나?** move는 원본을 수정하므로 보통 `T&&`를 받는다.
- **거짓 `noexcept`를 쓰면?** 예외가 빠져나오는 순간 `std::terminate`다.
- **Self-move는 반드시 원래 값을 보존하나?** 아니다. 타입 계약에 따르며 최소한 안전하게 파괴 가능한 상태를 유지하도록 설계하는 편이 좋다.

### 저지연 실무 연결

타입별 allocation 여부, move complexity, destructor 비용을 확인한다. 시작 단계 snapshot 교체의 deep copy는 줄이되, 주문 hot path에서는 애초에 container 성장을 허용하지 않는 것이 우선이다.

---

## Q5. `unique_ptr`와 `shared_ptr`의 비용과 thread-safety를 설명하라.

### 면접용 핵심 답변

`unique_ptr`는 단독 소유권을 표현하며 stateless deleter라면 보통 raw pointer와 같은 크기다. 복사는 금지되고 move로 소유권을 넘긴다. `shared_ptr`는 control block에 strong/weak count, deleter 등을 두며 복사·파괴 시 reference count를 원자적으로 갱신하는 것이 일반적이다.

서로 다른 `shared_ptr` 복사본이 같은 control block을 조작하는 것은 안전하지만, **가리키는 객체가 자동으로 thread-safe해지지는 않는다**. 같은 `shared_ptr` 변수 하나를 여러 thread가 갱신하려면 별도 동기화나 `atomic<shared_ptr<T>>`가 필요하다.

### 내부 동작 상세

`make_shared`는 보통 control block과 객체를 한 번에 할당한다. locality와 allocation 횟수는 좋지만 weak reference가 오래 남으면 객체 파괴 후에도 합쳐진 allocation이 유지될 수 있다. 마지막 strong count를 낮춘 thread에서 객체 소멸과 deallocation이 실행될 수 있다. 공유 refcount cache line은 core 사이를 왕복할 수 있다.

### 꼬리 질문과 짧은 답

- **Refcount ordering은?** 증가는 relaxed, 마지막 감소는 더 강한 ordering이 흔하지만 정확한 opcode는 구현 세부다.
- **순환 참조는?** strong count가 0이 되지 않으므로 `weak_ptr`로 ownership cycle을 끊는다.
- **Custom deleter 비용은?** `unique_ptr` 크기에 영향을 줄 수 있고 `shared_ptr`에서는 보통 control block에 저장된다.

### 저지연 실무 연결

fan-out 경로의 `shared_ptr` 복사는 원자적 refcount와 예측 불가능한 마지막 소멸 위치를 만든다. immutable snapshot은 RCU, epoch, double buffering, 명시적 ownership handoff와 비교한다.

---

## Q6. Storage duration과 object lifetime은 어떻게 다르며 placement new가 왜 어려운가?

### 면접용 핵심 답변

Storage는 바이트 공간이고 object lifetime은 그 공간에 특정 타입 객체가 존재해 타입 규칙으로 접근 가능한 기간이다. 메모리를 할당했다고 모든 타입의 lifetime이 자동으로 시작되는 것은 아니고, 객체를 파괴해도 storage는 남을 수 있다.

Placement new나 `construct_at`는 기존 storage에 객체를 생성해 lifetime을 시작한다. 재사용 시 기존 객체 파괴, alignment, 새 lifetime, 이전 pointer/reference의 유효성을 함께 다뤄야 한다. 일부 implicit-lifetime type 규칙을 모든 타입에 일반화하면 안 된다.

### 내부 동작 상세

```cpp
alignas(Order) std::byte storage[sizeof(Order)];
Order* p = std::construct_at(reinterpret_cast<Order*>(storage), args...);
use(*p);
std::destroy_at(p);
```

`std::launder`는 같은 storage 재사용의 제한된 pointer provenance 문제를 다루며 임의의 strict-aliasing 위반을 고쳐 주지 않는다. 객체 lifetime 밖에서 member를 읽으면 UB가 될 수 있다. 바이트 복사는 기본적으로 trivially copyable 조건과 `memcpy`/`bit_cast` 계약을 확인한다.

### 꼬리 질문과 짧은 답

- **Placement new가 allocation도 하나?** 아니다. 제공된 주소에 객체만 생성한다.
- **소멸자를 직접 호출하면 storage도 해제되나?** 아니다.
- **`memcpy`로 모든 객체를 복사할 수 있나?** 아니다. 일반적으로 trivially copyable 타입에서만 의존한다.

### 저지연 실무 연결

Pool과 ring slot은 storage와 lifetime을 분리한다. 정확히 한 번 생성·파괴되고 consumer가 끝낸 뒤에만 재사용된다는 invariant를 memory-order proof와 함께 검토한다.

---

## Q7. 일반 allocator가 tail latency를 만드는 이유와 대안은?

### 면접용 핵심 답변

범용 allocator는 평균이 빨라도 경쟁, thread-cache refill, arena 확장, page fault, coalescing, kernel 진입 때문에 드문 긴 지연을 만들 수 있다. 객체별 `new/delete`는 metadata와 cache/TLB locality도 악화시킨다.

대안은 최대 용량을 정해 시작 시 사전 할당하고 fixed-size pool, arena, `pmr`, intrusive free list, bounded container를 쓰는 것이다. 반드시 exhaustion 정책, alignment, NUMA placement, reclamation 시점을 명시해야 한다.

### 내부 동작 상세

`monotonic_buffer_resource`는 개별 free 없이 resource 전체를 일괄 반납하므로 batch 수명에 적합하다. Thread-local pool은 경쟁을 줄이나 remote free와 thread별 메모리 불균형이 생긴다. Lock-free free list는 allocator lock 대신 ABA와 reclamation 문제를 가져온다.

사전 allocation만으로 page fault가 없어지지는 않는다. 실제 page pre-touch, 필요 시 memory locking, huge page와 NUMA binding을 별도로 고려한다.

### 꼬리 질문과 짧은 답

- **`reserve`와 `resize` 차이는?** 전자는 capacity만 확보하고 후자는 element lifetime도 시작·종료한다.
- **Pool 고갈 시에는?** block, drop, reject, fail-fast 중 시스템 의미에 맞는 명시적 정책이 필요하다.
- **Cache-line alignment는 항상 좋은가?** 공유 쓰기는 줄이나 footprint와 TLB 압력을 늘릴 수 있다.

### 저지연 실무 연결

주문 pool 고갈 시 조용한 heap fallback은 장애 순간 p99.9를 폭발시킨다. 고갈을 telemetry와 risk 상태 전환에 연결하고 bounded 동작을 유지한다.

---

## Q8. Data race, `sequenced-before`, `synchronizes-with`, `happens-before`를 구분하라.

### 면접용 핵심 답변

한 thread의 평가 순서는 `sequenced-before`다. Release가 쓴 값을 acquire가 읽는 등의 조건이 성립하면 thread 사이 `synchronizes-with`가 생긴다. `happens-before`는 이 관계들을 전이적으로 연결해 한 접근의 효과를 다른 접근이 안전하게 관찰할 근거를 만든다.

서로 다른 thread의 conflicting access 중 하나 이상이 write이고, 적어도 하나가 atomic 연산이 아니며 happens-before로 정렬되지 않으면 data race이고 C++에서는 UB다. 단순 stale read 문제가 아니다.

### 내부 동작 상세

```cpp
payload = 42;                                  // A
ready.store(true, std::memory_order_release);  // B

if (ready.load(std::memory_order_acquire)) {   // C: B의 값을 읽을 때
    use(payload);                              // D
}
```

`A → B`, `B synchronizes-with C`, `C → D`이므로 A happens-before D다. Acquire가 이전 `false`를 읽은 실행에는 그 release와의 edge가 없다.

### 꼬리 질문과 짧은 답

- **동시 read 두 개도 race인가?** 둘 다 read면 conflicting access가 아니다.
- **Aligned plain 64-bit 값이 안 찢어지면 안전한가?** 아니다. non-atomic data race는 여전히 UB다.
- **`volatile`이면?** 원자성이나 thread synchronization을 제공하지 않는다.

### 저지연 실무 연결

각 plain payload access를 보호하는 happens-before graph를 그려 리뷰한다. ThreadSanitizer는 보조 도구일 뿐 모든 schedule과 lock-free correctness를 증명하지 않는다.

---

## Q9. Atomicity, cache coherence, memory ordering은 무엇이 다른가?

### 면접용 핵심 답변

Atomicity는 atomic 객체의 연산이 중간 상태로 관찰되지 않는다는 언어 보장이다. 각 atomic 객체에 대한 store와 RMW 같은 **modification**은 하나의 modification order를 이루고, atomic load는 memory-order 규칙상 허용된 modification의 값을 관찰한다. Load 자체가 그 modification order의 구성원인 것은 아니다. Cache coherence는 하드웨어가 주로 한 cache line의 복사본과 write ownership을 조정하는 메커니즘이다. Memory ordering은 여러 load/store 사이에서 어떤 순서를 관찰하도록 보장하는지 다룬다.

한 line이 MESI류 coherence에 참여해도 여러 변수 사이 C++ happens-before가 자동으로 생기지 않는다. 반대로 relaxed atomic도 coherence에는 참여하지만 주변 payload publication ordering은 제공하지 않는다.

### 내부 동작 상세

각 atomic 객체에는 모든 modification의 단일 modification order가 있지만, 서로 다른 atomic 객체들 사이 하나의 전역 순서가 자동으로 생기지는 않는다. `seq_cst` 연산들은 추가적인 single total order 제약에 참여한다.

하드웨어 store는 store buffer에 먼저 들어가고 ownership 획득과 line 갱신이 뒤따를 수 있다. Compiler는 target instruction과 코드 재배치를 함께 제어해 C++ 계약을 구현한다.

### 꼬리 질문과 짧은 답

- **Atomic은 cache를 우회하나?** 보통 cache/coherence를 사용한다.
- **Atomic이면 항상 lock-free인가?** 타입·정렬·구현에 따라 hidden lock을 쓸 수 있다.
- **False sharing은 data race인가?** 아니다. 서로 다른 객체여도 같은 line write ownership 경쟁이 생기는 성능 문제다.

### 저지연 실무 연결

Padding은 cache-line traffic 최적화이고 acquire/release는 언어-level correctness다. 두 문제를 독립적으로 검증해야 한다.

---

## Q10. `relaxed`와 release-acquire publication을 비교하라.

### 면접용 핵심 답변

Relaxed도 해당 atomic 접근의 원자성을 유지하고 store·RMW modification에는 객체별 modification order를 부여하지만, 주변 plain 데이터에 happens-before를 만들지 않는다. 독립적인 통계 counter에는 적합하지만 ready flag를 통한 payload publication에는 보통 부족하다.

Release store 이전 연산들은, 그 release가 쓴 값 또는 표준상 연결된 값을 실제로 읽은 acquire load 이후 연산보다 happens-before가 된다. Acquire를 썼다는 사실만으로 충분하지 않고 해당 release와 read-from 관계가 필요하다.

### 내부 동작 상세

```cpp
payload = 42;
ready.store(true, std::memory_order_release);

if (ready.load(std::memory_order_acquire))
    use(payload);
```

이 패턴은 payload publication을 보호하지만 backing storage의 동시 재사용까지 자동으로 막지는 않는다. 별도의 ownership/lifetime protocol이 필요하다. `acq_rel`은 값을 읽어 선행 상태를 acquire하면서 자신의 변경을 release하는 RMW에 사용한다.

### 꼬리 질문과 짧은 답

- **Relaxed counter increment는 유실되지 않나?** atomic RMW라면 각 증가가 modification order에 들어가 유실되지 않는다.
- **x86에서 relaxed publication이 작동해 보이면?** hardware ordering 관찰일 뿐 compiler와 C++ portable guarantee가 아니다.
- **Acquire가 다른 값을 읽었다면?** 해당 release와 synchronization이 성립하지 않을 수 있다.

### 저지연 실무 연결

Metrics는 per-thread relaxed counter로 write sharing까지 줄이고, queue flag/snapshot pointer는 필요한 publication edge를 유지한다. Ordering 완화보다 ownership partitioning이 더 큰 효과일 수 있다.

---

## Q11. `seq_cst`, atomic fence, compiler barrier를 설명하고 x86의 강한 ordering과 구분하라.

### 면접용 핵심 답변

`seq_cst`는 acquire/release 계열 보장에 더해 모든 seq_cst 연산이 참여하는 single total order 제약을 둔다. 추론은 쉬워지지만 lifetime, ABA, 잘못된 invariant, non-atomic race를 고치지는 않는다.

`atomic_thread_fence`는 atomic read/write와 결합해 C++ memory model의 ordering을 만든다. Compiler barrier는 compiler 재배치만 제한하고, CPU fence는 architecture-level ordering을 제한한다. Inline assembly fence 하나로 C++ data race가 사라지지는 않는다.

### 내부 동작 상세

x86-64에서 acquire load와 release store가 plain load/store로 lowering될 수 있는 것은 source-level 계약이 싸게 구현된다는 뜻이다. Source에서 atomic을 생략해도 된다는 뜻이 아니다. Compiler 최적화와 C++ UB는 CPU coherence로 해결되지 않는다. ARM에서는 acquire/release instruction이나 barrier가 더 명시적으로 나타날 수 있다.

Device MMIO/DMA는 일반 thread atomic과 다른 platform 계약이 필요하므로 driver/DPDK의 device barrier API를 따라야 한다.

### 꼬리 질문과 짧은 답

- **`asm volatile("" ::: "memory")`는 thread fence인가?** compiler barrier일 뿐 C++ synchronization은 아니다.
- **x86 `mfence`면 race가 사라지나?** 아니다.
- **seq_cst는 항상 느린가?** architecture와 operation에 따라 다르며 측정해야 한다.

### 저지연 실무 연결

먼저 올바른 edge를 증명하고 generated assembly와 PMU로 비용을 확인한다. “x86 전용”이어도 표준 atomic으로 의미를 표현하고 opcode를 검증하는 편이 안전하다.

---

## Q12. Compare-and-exchange의 weak/strong과 성공·실패 ordering을 설명하라.

### 면접용 핵심 답변

CAS는 atomic 값이 `expected`와 같으면 `desired`로 바꾸고 성공한다. 다르면 실제 값을 `expected`에 기록하고 실패한다. `compare_exchange_weak`는 값이 같아도 spurious failure를 허용해 retry loop에 적합하고, `strong`은 허위 실패를 허용하지 않아 단발성 판단에 적합할 수 있다.

성공 ordering은 read-modify-write에 적용된다. 실패는 load뿐이므로 실패 ordering에는 release/acq_rel을 쓸 수 없고 성공 ordering보다 강해서도 안 된다.

### 내부 동작 상세

```cpp
Node* seen = head.load(std::memory_order_relaxed);
do {
    n->next = seen;
} while (!head.compare_exchange_weak(
    seen, n,
    std::memory_order_release,
    std::memory_order_relaxed));
```

실패하면 `seen`이 현재 head로 갱신되므로 `n->next`도 다시 설정해야 한다. 성공 release는 node 초기화를 acquire pop에 공개할 수 있지만 안전한 reclamation은 별도 문제다. 경쟁 시 CAS retry와 cache-line bouncing이 tail latency를 키운다.

### 꼬리 질문과 짧은 답

- **Weak CAS가 x86에서도 허위 실패하나?** 표준상 허용되지만 구현은 strong과 같은 instruction을 쓸 수 있다.
- **CAS loop면 전체가 lock-free인가?** primitive와 전체 progress proof를 모두 봐야 한다.
- **seq_cst CAS면 ABA도 해결되나?** 아니다.

### 저지연 실무 연결

고경쟁 MPMC CAS loop보다 shard와 SPSC로 ownership을 나눈다. 불가피한 loop에는 retry 수, backoff와 contention을 계측한다.

---

## Q13. Obstruction-free, lock-free, wait-free와 `is_lock_free()`를 구분하라.

### 면접용 핵심 답변

- **Obstruction-free**: thread가 충분히 혼자 실행되면 완료된다.
- **Lock-free**: 전체 실행이 계속 step을 수행하면 어떤 operation은 계속 완료되지만, 특정 thread의 완료나 operation별 고정 step 상한은 보장하지 않는다.
- **Wait-free**: 각 연산이 다른 thread와 무관하게 유한한 step 상한 안에 완료된다.

`atomic<T>::is_lock_free()`는 해당 atomic 연산이 내부 lock 없이 구현되는지 말할 뿐, 그것을 사용한 자료구조 전체의 progress를 증명하지 않는다.

### 내부 동작 상세

Lock-free 코드도 preemption, cache miss, page fault, CAS 경쟁 때문에 wall-clock 지연 상한이 없을 수 있다. 반대로 uncontended mutex는 user-space fast path가 짧아 복잡한 lock-free 구조보다 빠를 수 있다. 올바르게 정렬된 atomic도 타입 크기·ABI·구현에 따라 내부 lock을 쓸 수 있으므로 target에서 `is_lock_free()`를 확인한다. 반면 `atomic_ref`를 포함한 atomic API가 요구하는 정렬을 어기는 것은 portable lock fallback이 아니라 precondition 위반이나 undefined behavior가 될 수 있다.

### 꼬리 질문과 짧은 답

- **Lock-free면 context switch가 없나?** 아니다.
- **Lock-free면 starvation이 없나?** 아니다. wait-free가 더 강한 per-thread progress다.
- **Wait-free면 실시간 deadline을 보장하나?** step bound와 page fault·preemption을 포함한 시간 상한은 다르다.

### 저지연 실무 연결

라벨보다 최악 retry, preemption, overload, reclamation을 검토한다. Bounded SPSC와 core ownership이 실용적으로 더 예측 가능한 경우가 많다.

---

## Q14. ABA 문제를 실행 순서로 설명하라.

### 면접용 핵심 답변

CAS는 값만 비교하므로 thread가 A를 읽은 사이 다른 thread가 A→B→A로 바꾸면 변화가 없었다고 오인할 수 있다. Pointer 값이 같아도 node 세대와 연결 관계가 달라졌거나, 해제 후 같은 주소에 새 객체가 생성됐을 수 있다.

### 내부 동작 상세

Treiber stack 예:

1. T1이 `head=A`, `A->next=B`를 읽고 멈춘다.
2. T2가 A와 B를 차례로 pop한다.
3. T2가 A를 다시 push하거나 allocator가 같은 주소를 재사용한다.
4. T1의 `CAS(head, A, B)`가 성공한다.
5. B는 더 이상 유효한 successor가 아닐 수 있어 구조가 손상된다.

해법은 pointer+version/tag CAS, hazard pointer, epoch/RCU reclaim 지연, 또는 sequence 기반 bounded slot이다. Tag는 wrap할 수 있고 double-width CAS의 lock-free 지원도 확인해야 한다. Reclamation이 주소 재사용 문제를 해결해도 논리적 A→B→A까지 해결하는지는 알고리즘별 증명이 필요하다.

### 꼬리 질문과 짧은 답

- **같은 pointer면 같은 객체 아닌가?** 주소 재사용 시 lifetime이 다른 객체다.
- **GC면 ABA가 사라지나?** 주소 재사용형은 완화되지만 논리 상태 ABA는 남을 수 있다.
- **Reference count로 간단히 해결되나?** 안전하게 count를 올리기 전에 해제될 수 있어 단순하지 않다.

### 저지연 실무 연결

동적 node MPMC보다 고정 slot과 증가 sequence를 쓰면 reclamation과 ABA가 단순해진다. Sequence wrap은 처리율과 서비스 수명으로 계산한다.

---

## Q15. Hazard pointer, epoch reclamation, RCU를 비교하라.

### 면접용 핵심 답변

Node를 자료구조에서 unlink하는 것과 메모리를 재사용하는 것은 별개다. Reader가 raw pointer를 보유할 수 있어 즉시 `delete`하면 use-after-free가 된다.

- **Hazard pointer**: reader가 접근할 pointer를 공개하고 재검증한다. 어떤 hazard에도 없는 retired node만 해제한다.
- **Epoch/EBR/QSBR**: reader의 참여 epoch 또는 quiescent state를 추적해 제거 이전 reader가 모두 빠진 뒤 일괄 해제한다.
- **RCU**: reader-side를 매우 가볍게 두고 update 후 grace period 뒤 reclaim한다. 정확한 의미는 API 구현에 따른다.

### 내부 동작 상세

Hazard reader는 공유 pointer 읽기 → hazard slot 공개 → 공유 pointer 재읽기 → 같을 때만 dereference 순서가 필요하다. 재검증 전 reclaimer가 해제할 수 있기 때문이다. EBR은 read path가 싸지만 한 participant가 epoch를 떠나지 않으면 reclaim이 막혀 메모리가 늘 수 있다.

### 꼬리 질문과 짧은 답

- **Unlink 직후 destructor를 호출해도 되나?** reader가 내용을 쓰는 중일 수 있어 일반적으로 안 된다.
- **`shared_ptr`로 충분하지 않나?** 참조의 안전한 획득, refcount 경쟁과 마지막 소멸 위치를 고려해야 한다.
- **RCU reader는 atomic이 전혀 없나?** 구현별 quiescent-state 추적과 ordering이 있으며 일반화할 수 없다.

### 저지연 실무 연결

Read-heavy instrument/config snapshot은 RCU/epoch에 잘 맞는다. 멈춘 thread가 grace period를 막을 수 있으므로 retired memory 상한과 stall telemetry가 필요하다.

---

## Q16. SPSC ring buffer가 acquire/release로 안전한 이유를 증명하라.

### 면접용 핵심 답변

SPSC에서는 producer만 tail을 쓰고 consumer만 head를 쓴다. Producer는 빈 slot에 payload를 쓴 뒤 release tail로 공개하고, consumer는 acquire tail로 그 진행을 본 뒤 slot을 읽는다. Consumer는 읽기를 끝낸 뒤 release head로 재사용 가능 상태를 공개하고 producer는 acquire head 뒤에만 slot을 덮는다.

### 내부 동작 상세

```cpp
// 전제: N은 unsigned constexpr인 2의 거듭제곱, mask == N - 1,
// head와 tail은 충분히 넓은 unsigned counter다.
static_assert(N != 0 && (N & (N - 1)) == 0);

// producer
auto t = tail.load(std::memory_order_relaxed);
auto h = head.load(std::memory_order_acquire);
if (t - h == N) return false;
slot[t & mask] = value;                         // A
tail.store(t + 1, std::memory_order_release);   // B

// consumer
auto h = head.load(std::memory_order_relaxed);
auto t = tail.load(std::memory_order_acquire);  // C
if (h == t) return false;
out = slot[h & mask];                           // D
head.store(h + 1, std::memory_order_release);
```

C가 B의 진행을 읽으면 A happens-before D다. 반대 방향 edge는 consumer가 끝낸 slot만 producer가 재사용하게 한다. 자기 thread만 쓰는 index load는 relaxed일 수 있다. `t - h`의 modular subtraction이 안전하려면 producer-consumer 거리가 항상 `0..N`이고 counter wrap을 모호하지 않게 구분할 수 있다는 invariant가 필요하다. Raw storage를 쓰면 construction/destruction lifetime도 같은 방식으로 증명해야 한다.

### 꼬리 질문과 짧은 답

- **Head와 tail이 같은 line이면?** correctness는 보통 유지되지만 false sharing이 커진다.
- **Index wrap은?** unsigned modulo, capacity, 최대 거리 조건을 증명해야 한다.
- **Full에서 overwrite해도 되나?** 데이터 의미에 따른 명시적 drop/backpressure 정책 없이는 안 된다.

### 저지연 실무 연결

Feed→strategy, strategy→gateway를 SPSC로 나누면 CAS와 reclamation을 피한다. Index를 별도 cache line에 두고 local cache로 상대 index load 횟수를 줄일 수 있다.

---

## Q17. SPSC를 MPSC/MPMC로 확장하면 무엇이 어려워지는가?

### 면접용 핵심 답변

여러 producer에서는 tail **예약**과 slot payload **publication**이 분리된다. P1이 slot 10을 예약하고 멈춘 동안 P2가 slot 11을 완성해도 consumer가 tail 값만 보고 slot 10을 읽어서는 안 된다. 여러 consumer까지 있으면 slot ownership과 reclamation도 복잡해진다.

따라서 bounded MPMC는 보통 slot별 sequence/state로 특정 turn의 empty/ready/reusable 상태를 표현하거나, linked-node 알고리즘과 안전한 reclamation을 쓴다.

### 내부 동작 상세

단순 `fetch_add`는 unique index 예약만 해결한다. Full 판정, 미완성 slot publication, producer preemption, lap wrap을 해결하지 않는다. Slot별 sequence는 producer가 예약한 slot의 payload를 완성한 뒤에만 consumer-visible 상태로 바꾸고, consumer 완료 후 다음 lap producer에게 되돌린다.

### 꼬리 질문과 짧은 답

- **CAS tail 하나면 충분한가?** 예약 중복만 막고 publication 순서는 못 막는다.
- **MPMC lock-free면 한 producer 정지가 무관한가?** 예약한 slot이 진행을 막는 구현도 있어 progress proof가 필요하다.
- **Mutex는 항상 더 느린가?** 낮은 contention에서는 단순하고 더 빠를 수 있다.

### 저지연 실무 연결

범용 MPMC 하나보다 producer별 SPSC fan-in이 경쟁을 줄인다. 대신 poll 공정성, timestamp ordering, backlog 우선순위를 정의해야 한다.

---

## Q18. `std::mutex`, futex, spinlock은 내부적으로 어떻게 다르고 언제 선택하는가?

### 면접용 핵심 답변

일반 Linux pthread mutex는 uncontended fast path에서 user-space atomic으로 잠그고, 경쟁으로 기다려야 할 때 futex를 통해 kernel sleep/wake를 사용한다. Futex는 mutex 자체가 아니라 user-space word가 예상 값일 때 잠들고 깨우는 kernel primitive다.

Spinlock은 기다리면서 CPU를 쓰므로 context switch는 피하지만 core 시간과 shared line/interconnect를 소모한다. Critical section이 매우 짧고 owner가 실행 중일 때만 유리할 수 있으며, owner가 preempt되면 spinner는 진행 없이 CPU를 태운다.

### 내부 동작 상세

Test-and-set은 매번 RMW해 line bouncing이 크다. Test-test-and-set은 read로 대기하다 free일 때만 RMW한다. Ticket lock은 FIFO지만 공통 serving line을 공유하고, MCS lock은 waiter별 node에 spin해 높은 경쟁에서 확장성이 좋다. 구현한 lock은 성공 lock에 acquire, unlock에 release 의미가 필요하다.

Futex/condition wait는 spurious wakeup과 상태 변경을 고려해 predicate를 loop에서 재검사한다. `atomic::wait/notify`의 spin/futex 조합은 library/platform 구현에 달려 있다.

### 꼬리 질문과 짧은 답

- **Uncontended mutex도 syscall을 하나?** 일반적으로 user-space fast path로 끝난다.
- **`yield()`면 spin 문제가 해결되나?** scheduler hint일 뿐 지연·공정성 보장은 없다.
- **Priority inversion은?** 낮은 우선순위 owner 때문에 높은 우선순위 waiter가 막히는 현상이다.
- **Lock-free가 mutex보다 항상 빠른가?** Retry storm과 reclamation 때문에 더 나쁠 수 있다.

### 저지연 실무 연결

먼저 공유 상태를 core별로 소유하게 한다. 공유가 불가피하면 lock hold time, owner preemption, contention과 p99.9를 측정하고 bounded spin 뒤 sleep/fallback 정책을 둔다.

# 5. Linux·Network·측정·Trading Domain

아래 동작의 세부 사항은 CPU architecture, Linux kernel, NIC·driver와 거래소 protocol에 따라 달라질 수 있다. 면접에서는 일반 원리와 플랫폼별 계약을 구분해서 설명한다.

## 5.1 Linux 실행·메모리·스케줄링

### Q1. 시스템 콜과 컨텍스트 스위치는 같은 것인가?

#### 면접용 핵심 답변

아니다. **시스템 콜은 같은 스레드가 user mode에서 kernel mode로 권한 수준을 전환해 커널 서비스를 실행하는 것**이고, **컨텍스트 스위치는 실행 중인 태스크가 다른 태스크로 교체되는 것**이다. `read()`가 page cache에서 즉시 데이터를 찾으면 시스템 콜 진입과 복귀만 하고 같은 스레드가 계속 실행할 수 있다. 반대로 데이터가 없어 sleep하거나, 실행 중 선점되면 scheduler가 다른 태스크를 선택하면서 컨텍스트 스위치가 발생한다.

#### 내부 동작 상세

1. 애플리케이션이 시스템 콜 번호와 인자를 ABI가 정한 레지스터에 넣고 `syscall` 같은 진입 명령을 실행한다.
2. CPU는 커널 진입점으로 이동하고, 커널은 필요한 사용자 상태를 저장한 뒤 인자와 권한을 검증한다.
3. 요청이 즉시 끝나면 결과를 레지스터에 두고 user mode로 복귀한다.
4. I/O 대기, storage I/O를 기다리는 major page fault, futex wait 등으로 현재 태스크가 runnable하지 않게 되거나 선점되면 scheduler가 다른 태스크의 레지스터·주소 공간 관련 상태를 복원한다. 반면 minor fault처럼 커널이 즉시 처리할 수 있는 fault는 다른 태스크로 전환하지 않고 같은 태스크로 복귀할 수 있다.

모드 전환에는 진입/복귀, 보안 완화책, 커널 코드 실행 비용이 든다. 컨텍스트 스위치에는 그 비용 외에도 scheduler 실행, 레지스터 교체, cache·TLB·branch predictor locality 손실 가능성이 더해진다. 단, PCID/ASID 같은 기능은 주소 공간 교체 시 TLB 손실을 줄일 수 있으므로 “스위치마다 TLB 전체 flush”라고 단정하면 안 된다.

#### 꼬리 질문

- **`clock_gettime()`도 항상 시스템 콜인가?** 아니다. 지원되는 clock은 vDSO를 통해 user space에서 읽을 수 있다. 어떤 clock과 플랫폼이 지원되는지는 확인해야 한다.
- **시스템 콜을 없애면 항상 빨라지는가?** 호출 횟수만 줄이고 batching 때문에 대기 시간이 늘면 tail latency가 악화될 수 있다. 호출 비용과 queueing을 함께 봐야 한다.
- **프로세스 전환과 스레드 전환의 비용은 항상 다른가?** 같은 주소 공간을 공유하는 스레드 전환이 대체로 locality에 유리하지만, 실제 비용은 working set과 CPU 배치에 좌우된다.

#### 저지연 실무 연결

hot path에서는 매 packet마다 syscall하지 않도록 batching, `recvmmsg`, memory mapping, kernel bypass 등을 검토한다. 그러나 “syscall 수”만 대리 지표로 삼지 말고 `perf`, scheduler trace, 실제 p99.9를 함께 측정한다.

---

### Q2. CPU affinity를 설정하면 jitter가 사라지는가?

#### 면접용 핵심 답변

아니다. affinity는 **해당 태스크가 실행될 수 있는 CPU 집합을 제한**할 뿐 그 CPU를 독점시켜 주지 않는다. IRQ, softirq, kernel thread, RCU callback, 다른 허용된 태스크, SMT sibling의 경쟁, frequency·power state 변화가 여전히 영향을 줄 수 있다. 저지연 코어 구성은 application affinity, IRQ affinity, queue affinity, NUMA placement, housekeeping 분리를 하나의 문제로 다뤄야 한다.

#### 내부 동작 상세

- `sched_setaffinity`는 scheduler의 배치 가능 CPU를 제한한다.
- `isolcpus`는 일반 load balancing에서 CPU를 격리하는 부팅 옵션이지만, 모든 커널 작업과 interrupt를 자동으로 제거하는 만능 스위치는 아니다.
- `nohz_full`은 조건을 만족하는 CPU에서 periodic scheduler tick을 줄인다. runnable 태스크가 여러 개이거나 다른 커널 작업이 있으면 효과가 제한된다.
- `rcu_nocbs`는 RCU callback 처리를 지정 CPU 밖으로 넘기는 데 사용될 수 있다.
- `SCHED_FIFO`는 일반 태스크보다 높은 우선순위로 계속 실행될 수 있지만, 잘못 사용하면 housekeeping과 복구 작업까지 굶겨 시스템을 멈춘 것처럼 만들 수 있다.
- SMT sibling은 execution port, cache, TLB 등 일부 자원을 공유하므로 sibling에 다른 부하가 있으면 고정된 코어에서도 지연이 흔들릴 수 있다.

#### 꼬리 질문

- **`SCHED_FIFO`가 항상 가장 빠른가?** scheduler 지연은 줄일 수 있지만 우선순위 역전, starvation, runaway loop 위험이 있다. watchdog와 운영 절차가 필요하다.
- **한 NIC queue와 한 thread를 같은 CPU에 붙이면 끝인가?** 메모리와 PCIe device가 다른 NUMA node라면 원격 접근 비용이 남는다.
- **SMT를 반드시 꺼야 하는가?** workload에 따라 다르다. sibling을 비워 tail을 안정화하거나, 처리량 때문에 사용하기도 한다. 측정으로 결정한다.

#### 저지연 실무 연결

`NIC RX queue → MSI-X vector/polling thread → parser/strategy thread → memory pool`의 CPU와 NUMA node를 문서화한다. `/proc/interrupts`, task affinity, thread migration, C-state와 frequency를 실행 때마다 기록해야 재현 가능한 benchmark가 된다.

---

### Q3. `mlockall()`을 호출하면 page fault가 완전히 없어지는가?

#### 면접용 핵심 답변

그렇게 보장할 수 없다. `mlockall()`은 매핑된 페이지를 swap 대상에서 제외하는 수단이지, 앞으로 접근할 모든 주소에 대한 fault를 무조건 제거하는 선언이 아니다. 사용 flag, 이후 생성되는 mapping, stack 성장, copy-on-write, 파일 매핑, lazy allocation에 따라 fault가 생길 수 있다. 저지연 시작 단계에서는 메모리를 미리 할당하고 실제 접근 패턴으로 page를 pre-touch하며, 실행 중 fault counter를 확인해야 한다.

#### 내부 동작 상세

- minor fault는 backing data를 disk에서 읽을 필요는 없지만 page table 설치, zero-page 할당, COW 같은 커널 처리가 필요하다.
- major fault는 storage I/O가 필요할 수 있어 지연이 훨씬 크다.
- anonymous memory는 `malloc` 또는 `mmap` 시 주소 공간만 예약되고 첫 write 때 실제 physical page가 배정될 수 있다.
- fork 이후 write는 COW fault를 일으킬 수 있다.
- file-backed mapping은 page cache 상태에 따라 fault 비용이 달라진다.
- `MCL_CURRENT`, `MCL_FUTURE`, `MCL_ONFAULT`의 의미는 다르다. 특히 on-fault 방식은 이름 그대로 첫 접근 fault를 허용한다.

pre-touch는 각 page에 읽기만 하는 것으로 충분하지 않을 수 있다. 앞으로 write할 page라면 write해 실제 private writable page와 필요한 page table을 만들고, thread stack도 예상 최대 깊이까지 준비한다. 동시에 `RLIMIT_MEMLOCK`과 실패 반환값을 검사해야 한다.

#### 꼬리 질문

- **minor fault는 disk I/O가 없으니 무시해도 되는가?** 아니다. tail latency가 중요한 thread에서는 커널 진입, page table 변경, zeroing, TLB shootdown 가능성도 큰 outlier가 된다.
- **`MAP_POPULATE`만 사용하면 충분한가?** 초기 population을 돕지만 이후 COW, 새 mapping, stack 성장 등 모든 원인을 막지는 않는다.
- **메모리를 lock하면 OOM에 안전한가?** 오히려 회수할 수 없는 메모리를 늘린다. 용량 계획과 실패 처리 없이는 시스템 안정성을 해칠 수 있다.

#### 저지연 실무 연결

초기화 단계에서 pool·ring·stack을 할당하고 page별 write로 pre-touch한 뒤 거래 시작 barrier를 연다. `perf stat`의 page-faults, `/proc/<pid>/status`, production telemetry로 hot path fault가 0인지 감시한다.

---

### Q4. IRQ, softirq, NAPI는 packet 처리에서 어떻게 연결되는가?

#### 면접용 핵심 답변

고속 NIC는 보통 MSI-X interrupt로 RX queue의 work를 알리고, Linux driver는 interrupt handler에서 무거운 packet 처리를 전부 하지 않고 NAPI polling을 schedule한다. NAPI poll의 budget은 일반적으로 한 번의 poll에서 처리할 **RX packet 수**를 제한하며, TX completion은 그 RX budget과 별도로 회수할 수 있다. 부하가 계속되면 interrupt를 줄이고 polling 방식으로 여러 packet을 처리한다. softirq 처리 한도에 걸리거나 work가 밀리면 `ksoftirqd`가 이어받을 수 있어 scheduler 지연과 queueing이 발생한다.

#### 내부 동작 상세

1. NIC가 RX ring에 completion을 기록한다.
2. interrupt가 활성화된 상태라면 queue에 대응하는 MSI-X vector를 발생시킨다.
3. 짧은 hardirq handler가 NAPI instance를 schedule하고 중복 interrupt를 제한한다.
4. 보통 NET_RX softirq에서 driver poll 함수가 RX packet을 처리하고 TX completion을 회수한다. Threaded NAPI나 busy-poll 설정에서는 실행 context가 달라질 수 있다.
5. packet이 kernel network stack, socket receive queue로 이동한다.
6. Driver가 outstanding work 처리를 마치고 `napi_complete_done()`으로 NAPI ownership을 반환한 뒤, 조건이 맞으면 interrupt를 다시 활성화한다.

정확한 masking 방식과 처리 위치는 driver·kernel 설정에 따라 다르다. NAPI는 interrupt storm을 줄이고 처리량을 높이지만, budget, backlog, `ksoftirqd` 스케줄링이 tail을 만들 수 있다.

#### 꼬리 질문

- **interrupt를 완전히 끄고 busy polling하면 무조건 빠른가?** 낮은 부하의 wake-up은 빨라질 수 있지만 dedicated core와 전력 비용이 필요하고, 과부하 시 다른 작업을 굶길 수 있다.
- **`ksoftirqd`가 보이면 문제인가?** 존재 자체가 아니라 latency-sensitive packet이 그 경로에서 오래 queueing되는지를 확인해야 한다.
- **IRQ affinity만 옮기면 softirq도 반드시 옮겨지는가?** 기본 경로는 연관되지만 RPS 등 software steering과 backlog 처리 때문에 다른 CPU가 처리할 수 있다.

#### 저지연 실무 연결

`/proc/interrupts`, softnet 통계, per-queue NIC counter를 함께 본다. application core에서 예기치 않은 NIC IRQ나 `ksoftirqd`가 실행되지 않는지 확인하고, polling/interrupt 모드는 목표 부하 분포에서 비교한다.

---

## 5.2 NIC·Network Path·DMA·Kernel Bypass

### Q5. Ethernet frame이 NIC에 도착한 뒤 `recv()`가 반환할 때까지 설명해보라.

#### 면접용 핵심 답변

NIC는 frame을 검증·분류한 뒤 RSS 등으로 RX queue를 선택하고, driver가 미리 ring descriptor에 등록해 둔 buffer로 DMA한다. completion 상태를 기록한 뒤 interrupt 또는 polling으로 CPU에 알린다. driver/NAPI가 descriptor를 회수해 packet metadata를 만들고, Ethernet/IP/UDP 처리를 거쳐 socket receive queue에 넣는다. 대기 중인 application을 깨우며, 일반 socket `recv()`는 보통 kernel buffer에서 user buffer로 payload를 복사한다.

#### 내부 동작 상세

```text
wire → PHY/MAC → NIC classification/RSS
     → PCIe DMA to posted RX buffer
     → descriptor completion
     → MSI-X 또는 polling
     → driver NAPI poll
     → XDP(optional) → skb 생성/연결
     → L2/L3/L4 처리
     → socket receive queue
     → wake-up → recv/recvmmsg → user buffer
```

XDP는 일반적으로 skb 할당 이전 지점에서 packet을 drop, redirect, pass할 수 있다. GRO는 여러 packet을 상위 계층에 큰 단위로 전달할 수 있다. 실제 copy 횟수와 buffer 모델은 API, driver, offload에 따라 달라지므로 “항상 두 번 복사”처럼 고정해 말하면 안 된다.

#### 꼬리 질문

- **packet payload가 항상 DRAM까지 갔다가 CPU로 오는가?** 플랫폼의 coherent I/O와 DDIO 같은 기능에서는 LLC에 배치될 수 있지만 CPU·NIC·설정에 따라 다르다.
- **UDP는 TCP보다 항상 빠른가?** protocol 상태는 단순하지만 application이 loss, ordering, recovery를 책임진다. end-to-end 지연은 구현과 부하에 좌우된다.
- **`recvmmsg`의 trade-off는?** syscall amortization은 좋아지지만 batch를 기다리거나 큰 batch를 처리하면 개별 packet tail이 늘 수 있다.

#### 저지연 실무 연결

지연을 NIC timestamp, driver 수거, socket dequeue, parser 완료처럼 구간별로 나눈다. 어느 단계의 queue가 쌓였는지 모른 채 socket API만 교체하는 것은 원인 해결이 아니다.

---

### Q6. RX/TX descriptor ring의 ownership은 어떻게 이동하는가?

#### 면접용 핵심 답변

Ring은 shared memory이지만 각 descriptor의 현재 소유자가 명확해야 한다. RX에서는 software가 빈 buffer 주소를 게시해 device에 넘기고, device가 packet을 DMA한 뒤 completion/owner 상태를 바꾸어 software에 돌려준다. software는 완료를 확인하고 payload를 읽은 다음 buffer를 재활용한다. TX에서는 software가 payload와 descriptor를 준비하고 device에 게시하며, completion을 확인하기 전에는 그 buffer를 재사용하면 안 된다.

#### 내부 동작 상세

publication 순서가 핵심이다.

```text
TX: payload 작성 → descriptor addr/len 작성
    → write barrier → tail/doorbell 갱신

RX: completion/owner 관측
    → read barrier → len/status/payload 읽기
    → 처리 완료 → buffer 재게시
```

Barrier의 정확한 종류와 MMIO accessor는 framework 계약에 따른다. Tail doorbell write는 흔히 PCIe posted write이므로, write 호출이 반환됐다고 NIC가 packet을 wire에 내보낸 것은 아니다. MMIO readback은 앞선 posted doorbell write가 장치 또는 정의된 ordering point에 도달하도록 flush하는 수단일 뿐, NIC가 descriptor 처리를 끝냈거나 packet을 wire에 송출했다는 뜻은 아니다. TX buffer 재사용 가능 시점은 장치가 정의한 completion 또는 ownership으로 판단하고, 실제 wire 송출 시각이 필요하면 해당 NIC의 completion과 hardware timestamp 의미를 확인한다.

#### 꼬리 질문

- **descriptor를 너무 적게 두면?** burst 흡수력이 줄고 NIC no-buffer drop 가능성이 커진다.
- **너무 많이 두면?** drop은 줄 수 있지만 오래된 packet이 queue 안에 머물러 latency와 stale-data 위험이 커진다.
- **completion 전에 TX buffer를 수정하면?** NIC가 읽는 중인 데이터를 바꿔 wire corruption 또는 잘못된 packet을 만들 수 있다.

#### 저지연 실무 연결

ring 크기는 처리량 최대화가 아니라 허용 가능한 queueing 상한과 burst 크기로 정한다. descriptor exhaustion, oldest age, completion lag를 telemetry로 노출한다.

---

### Q7. RSS, RPS, RFS, XPS의 차이와 NUMA 영향을 설명해보라.

#### 면접용 핵심 답변

RSS는 NIC hardware가 packet header hash와 indirection table로 RX queue를 선택하는 기능이다. RPS는 수신 후 kernel이 software로 처리 CPU를 분산하고, RFS는 flow를 해당 socket을 소비하는 CPU 쪽으로 유도해 cache locality를 개선하려 한다. XPS는 transmit queue 선택을 CPU/queue 기준으로 조정한다. 저지연 시스템에서는 NIC queue, interrupt/poll thread, application, memory를 같은 NUMA node에 배치해 remote memory와 cache-line 이동을 줄이는 것이 중요하다.

#### 내부 동작 상세

RSS는 보통 Toeplitz hash를 사용하지만 hash 입력 field와 key, indirection table은 설정에 따라 달라진다. 같은 flow의 packet을 같은 queue로 보내 ordering을 유지하는 것이 일반적이다. RPS/RFS는 hardware queue 이후 software handoff를 추가하므로 분산 이점과 cache/queue 비용을 교환한다.

NUMA mismatch의 예:

```text
NIC가 socket 0 PCIe root에 연결
→ RX buffer도 실제 node 0에 할당되도록 구성·확인
→ 처리 thread는 node 1
→ payload와 ring metadata를 원격으로 읽고
  free/recycle 시 cache line이 node 사이를 왕복
```

NIC의 PCIe 연결 위치가 RX buffer의 NUMA node를 자동으로 결정하지는 않는다. Buffer 위치는 driver/page-pool이 할당을 수행한 CPU와 memory policy, userspace UMEM의 할당 방식 등에 좌우되므로 실제 배치를 측정하거나 명시적으로 고정해야 한다.

#### 꼬리 질문

- **queue를 CPU 수만큼 만들면 항상 좋은가?** 저부하에서는 flow 분산과 cache footprint만 늘 수 있다. active thread와 flow 특성에 맞춘다.
- **RSS는 양방향 flow를 같은 queue에 넣는가?** 기본 hash field와 key에 따라 달라진다. symmetric hashing이 필요한 경우 별도 설정을 확인한다.
- **RPS를 켜면 latency가 낮아지는가?** overloaded queue 분산에는 유리하지만 software enqueue와 inter-CPU wakeup이 추가된다.

#### 저지연 실무 연결

시장 데이터 multicast feed별 queue, order-entry connection, application core를 의도적으로 배치한다. `/sys/class/net/.../queues`, `ethtool`, `/proc/interrupts`, NUMA topology를 배포 검증 항목으로 만든다.

---

### Q8. GRO/TSO/checksum offload와 interrupt moderation은 latency에 어떤 영향을 주는가?

#### 면접용 핵심 답변

Offload는 CPU당 처리량을 높이지만 batching과 가시성 변화를 만들 수 있다. Checksum offload는 대체로 CPU 작업을 줄인다. GRO/LRO는 여러 수신 packet을 합쳐 stack 비용을 줄이지만 packet 단위 처리 시점을 늦추거나 burst로 보이게 할 수 있다. TSO/GSO는 큰 buffer를 작은 wire packet으로 나누는 일을 뒤로 미룬다. Interrupt moderation은 일정 packet 수나 시간을 모아 interrupt를 발생시켜 overhead를 줄이지만 첫 packet의 대기 시간을 늘릴 수 있다.

#### 내부 동작 상세

“offload off = 저지연”도, “offload on = 고성능”도 일반 법칙이 아니다. Interrupt를 매우 자주 발생시키면 저부하 latency는 좋아질 수 있지만 고부하에서는 CPU가 interrupt 처리에 잠식되어 NAPI backlog와 packet loss가 커진다. 그러면 tail은 오히려 나빠진다. 반대로 큰 coalescing timer는 처리량을 높여도 latency floor를 만든다.

GRO가 켜진 환경에서 capture 지점에 따라 여러 wire packet이 하나의 큰 logical packet처럼 관측될 수 있으므로 packet 분석 결과도 주의해야 한다.

#### 꼬리 질문

- **hardware timestamp도 interrupt 시점인가?** NIC 기능에 따라 PHY/MAC 근처에서 찍히며 interrupt·software timestamp와 의미가 다르다.
- **checksum offload는 latency를 무조건 낮추는가?** 대체로 CPU를 절약하지만 NIC/driver 구현과 작은 packet workload에서 측정해야 한다.
- **coalescing 0이 가장 낮은 p99인가?** burst가 있을 때 interrupt storm이 p99를 악화시킬 수 있다.

#### 저지연 실무 연결

평균 packet rate뿐 아니라 실제 microburst를 재생해 coalescing parameter를 튜닝한다. 변경 전후의 packet drop, CPU softirq 비율, p50과 p99.99를 함께 비교한다.

---

### Q9. UDP packet loss가 어디서 발생했는지 어떻게 찾는가?

#### 면접용 핵심 답변

하나의 global drop counter로는 부족하다. **wire/NIC → RX ring → driver/NAPI → kernel backlog/protocol → socket receive queue → application queue**로 경계를 나누고, feed sequence number와 각 계층 counter·timestamp를 상관시킨다. 가장 먼저 sequence gap이 송신 측 결손인지 로컬 drop인지 구분하고, gap 시점의 per-queue counter 증가를 본다.

#### 내부 동작 상세

- NIC: missed packet, no-buffer, CRC/error, per-queue drop. 이름은 driver별로 다르므로 `ethtool -S` 설명을 확인한다.
- Driver/NAPI: RX ring exhaustion, budget pressure.
- Kernel: `/proc/net/softnet_stat`의 backlog drop/time squeeze 계열 지표, protocol 통계.
- Socket: `SO_RCVBUF`, UDP receive error/drop 통계, `ss -u -i` 등.
- Application: dequeue lag, bounded queue overflow, sequence gap, processing pause.

Packet capture도 capture 지점 앞에서 발생한 drop만 보여준다. 두 위치의 hardware tap 또는 송·수신 sequence/timestamp가 있으면 구분력이 높아진다. Counter는 rollover, reset, driver 의미 차이를 고려해 delta로 읽는다.

#### 꼬리 질문

- **socket buffer를 크게 만들면 해결되는가?** burst 흡수에는 도움 되지만 stale packet을 더 오래 보존해 latency를 숨길 수 있다.
- **gap 뒤 packet을 계속 적용해도 되는가?** order book protocol에서는 누락 하나로 state가 틀릴 수 있어 보통 invalid 상태로 전환하고 복구해야 한다.
- **pcap에 packet이 없으면 NIC 전에 유실된 것인가?** capture hook 이전 로컬 경로에서 drop됐을 수도 있다. capture 위치를 알아야 한다.

#### 저지연 실무 연결

운영 dashboard에는 sequence gap뿐 아니라 RX queue별 `no buffer`, softnet drop, socket/application queue high-watermark를 같은 시간축에 표시한다. 장애 후 “UDP라서 유실”이라고 끝내지 않고 최초 drop 경계를 좁힌다.

---

### Q10. DPDK나 AF_XDP 같은 kernel bypass는 왜 빠르며 무엇을 잃는가?

#### 면접용 핵심 답변

DPDK는 보통 userspace poll-mode driver가 NIC queue를 직접 polling하고, 미리 등록한 huge-page memory pool의 packet buffer를 사용해 syscall, interrupt, 일반 kernel network stack과 allocation 비용을 줄인다. AF_XDP는 XDP 지점과 userspace UMEM을 ring으로 연결해 낮은 overhead의 packet I/O를 제공한다. 대신 dedicated core, memory ownership, driver/device 설정, routing·firewall·TCP 같은 kernel 기능, 보안 격리, 운영 관측과 장애 복구를 애플리케이션이 더 많이 책임진다.

#### 내부 동작 상세

- 미리 할당한 DMA 가능 buffer와 bounded descriptor ring을 반복 사용한다.
- poll loop는 wake-up 지연을 피하지만 idle 상태에서도 CPU를 소비한다.
- zero-copy 가능 여부는 NIC driver와 mode에 따라 다르며, unsupported 경로에서는 copy mode가 될 수 있다.
- VFIO/IOMMU는 device access 격리와 DMA mapping에 사용된다.
- kernel을 우회해도 NIC, PCIe, DMA, queue, coherence 비용까지 사라지는 것은 아니다.

#### 꼬리 질문

- **kernel bypass면 packet loss가 없어지는가?** 아니다. application이 제때 ring을 비우지 못하면 동일하게 drop된다.
- **busy polling이면 latency 상한이 생기는가?** 아니다. SMI, page fault, cache miss, PCIe, 경쟁, 장치 queue 같은 outlier는 남는다.
- **AF_XDP와 DPDK 중 무엇이 더 빠른가?** device·mode·운영 요구에 따라 다르다. zero-copy 지원과 실제 workload로 비교해야 한다.

#### 저지연 실무 연결

기술 선택 전에 “줄이려는 구간”을 명시한다. 일반 socket 경로가 이미 SLA를 만족한다면 bypass의 복잡도가 정당화되지 않을 수 있다. 과부하 시 ring full 정책과 recovery를 설계하지 않으면 단지 drop 위치만 옮긴다.

---

## 5.3 Latency 측정·Benchmarking

### Q11. 평균 latency보다 percentile이 중요한 이유는 무엇인가?

#### 면접용 핵심 답변

평균은 드문 큰 지연을 숨기며 실제 주문 손실은 tail에서 발생한다. p99는 관측값의 99%가 그 이하라는 뜻이고, p99.9·p99.99는 더 드문 outlier를 본다. 하지만 percentile만으로 충분하지 않다. 표본 수, 측정 기간, 부하 분포, histogram 범위·해상도, max와 timeout 처리까지 함께 제시해야 한다.

#### 내부 동작 상세

초당 10만 건을 측정해도 p99.999는 초당 한 건 수준이라 짧은 실행에서는 안정적으로 추정하기 어렵다. 서로 다른 시간대의 값을 단순 평균 내는 것도 잘못될 수 있다. 원본 histogram을 합쳐야 전체 percentile을 다시 계산할 수 있다.

Tail 원인에는 queueing, scheduler pause, page fault, interrupt burst, cache/TLB miss, lock contention, GC가 없더라도 allocator slow path 등이 있다. Max는 중요한 장애 신호지만 단 한 번의 외부 간섭에 민감하므로 발생 원인의 trace를 함께 보관한다.

#### 꼬리 질문

- **두 시스템의 p99가 같으면 품질도 같은가?** p99.9 이후 꼬리, drop·timeout, 부하 조건, 결과 정확성이 다를 수 있다.
- **percentile들의 평균은 전체 percentile인가?** 아니다. 각 구간 sample 수와 분포를 잃기 때문이다.
- **p100은 무엇인가?** 사실상 측정 구간의 max이며 표본 수에 크게 의존한다.

#### 저지연 실무 연결

서비스 SLA는 `부하/버스트 모델 + percentile + 허용 drop + 측정 구간`으로 정의한다. 배포 비교에는 동일 histogram 설정과 sample 수를 사용한다.

---

### Q12. Coordinated omission이란 무엇이며 어떻게 피하는가?

#### 면접용 핵심 답변

Closed-loop 부하 생성기가 요청 응답을 기다린 뒤 다음 요청을 보내면, 시스템이 멈춘 동안 원래 도착했어야 할 요청 자체를 만들지 않는다. 그래서 긴 stall이 적은 수의 느린 sample로만 기록되어 tail이 실제보다 좋아 보인다. 이를 coordinated omission이라 한다. 고정된 목표 도착 시각을 가진 open-loop 생성기를 사용하고, 실제 전송/처리가 늦어졌다면 예정 시각부터의 지연을 기록해야 한다.

#### 내부 동작 상세

예를 들어 1ms마다 요청해야 하는데 서버가 100ms 멈췄다고 하자. Closed loop는 100ms짜리 요청 하나만 기록할 수 있다. 실제 시장에서는 그동안 약 100개의 message가 도착해 queueing되며 각각 큰 지연을 겪는다. Open loop는 예정된 arrival schedule을 유지하거나 적어도 missed arrivals를 모델에 반영한다.

단, 생성기가 목표 rate를 감당하지 못하면 client 자체가 bottleneck이 된다. Arrival distribution도 현실의 burst를 반영해야 하며, 무한 open-loop queue는 비현실적인 수치를 만들 수 있다.

#### 꼬리 질문

- **HDR Histogram의 correction 기능만 쓰면 해결되는가?** 일정 expected interval 가정의 보정일 뿐 실제 arrival과 overload 정책을 대신하지 않는다.
- **Closed loop는 쓸모없는가?** 동기식 client 경험을 측정하는 목적에는 맞다. 무엇을 모델링하는지가 중요하다.
- **시장 데이터 replay는 open loop인가?** 원본 timestamp 간격대로 독립 재생하면 가깝지만 replay engine 지연과 burst 보존을 검증해야 한다.

#### 저지연 실무 연결

평균 rate만 맞춘 synthetic load와 실제 feed microburst replay를 모두 사용한다. 부하 생성기 CPU를 SUT와 분리하고 예정 timestamp, 실제 send, receive, completion을 남긴다.

---

### Q13. TSC, `clock_gettime`, PTP hardware timestamp는 어떻게 다른가?

#### 면접용 핵심 답변

TSC는 CPU cycle counter 계열이라 읽기 비용이 낮지만 invariant·cross-core synchronization, 명령 재정렬, cycle-to-time 변환을 확인해야 한다. `CLOCK_MONOTONIC`은 monotonic하지만 시간 조정의 영향을 속도 보정 형태로 받을 수 있고, `CLOCK_MONOTONIC_RAW`는 원시 hardware clock에 더 가깝다. PTP PHC와 NIC hardware timestamp는 packet이 NIC의 특정 지점을 통과한 시각을 기록해 software scheduling 시간을 분리하는 데 유용하다.

#### 내부 동작 상세

- `RDTSC`는 일반적인 serializing instruction이 아니므로 측정 구간의 앞뒤 instruction이 넘어오지 않게 fence 또는 정해진 패턴을 사용해야 한다.
- `RDTSCP`는 이전 instruction과의 ordering에 도움을 주고 CPU ID 정보를 얻을 수 있지만, 이후 instruction까지 완전히 막는다고 단정하면 안 된다.
- 최신 x86의 invariant TSC는 P-state와 무관한 일정 rate를 제공할 수 있으나 VM, 구형 시스템, socket 간 동기화는 확인 대상이다.
- `clock_gettime`은 vDSO 경로라면 syscall 없이 빠르게 읽을 수 있지만 clock 종류에 따라 다르다.
- 서로 다른 host의 timestamp를 비교하려면 PTP/NTP 동기화 오차와 asymmetry를 error budget으로 포함해야 한다.

#### 꼬리 질문

- **TSC tick이 CPU core cycle과 같은가?** invariant TSC에서는 실제 순간 core frequency와 같지 않을 수 있다.
- **software timestamp로 wire latency를 잴 수 있는가?** software stack·scheduler 구간이 섞인다. 목적에 따라 hardware timestamp가 필요하다.
- **CPU migration이 왜 문제인가?** 시스템에서 TSC가 완전히 동기화되지 않았다면 역행·offset이 생길 수 있고 cache locality도 변한다.

#### 저지연 실무 연결

측정 clock, serialization 방식, 변환 계수, cross-core 검증을 benchmark 코드에 명시한다. NIC RX/TX hardware timestamp와 application timestamp를 같이 수집해 wire, stack, application 구간을 분리한다.

---

### Q14. 신뢰할 수 있는 microbenchmark를 만드는 방법은?

#### 면접용 핵심 답변

측정 대상이 compiler에 의해 제거·상수화되지 않게 하고, clock 비용을 파악하며, cache hot/cold와 branch input distribution을 의도적으로 설계해야 한다. CPU pinning, warm-up, frequency/power 상태, NUMA placement, compiler flags와 binary를 고정하고 단일 숫자가 아니라 분포를 수집한다. 실제 production input과 동떨어진 loop는 “그 loop의 속도”만 측정한다.

#### 내부 동작 상세

대표적인 오류:

- 결과를 사용하지 않아 dead-code elimination됨.
- 상수 input 때문에 constant folding 또는 완전한 loop 제거가 발생함.
- 같은 데이터만 반복해 모든 접근이 L1에 남음.
- branch pattern이 지나치게 규칙적이라 predictor가 실제보다 잘 맞춤.
- benchmark loop overhead와 timestamp 비용이 대상보다 큼.
- 많은 iteration을 한 번에 재서 평균만 남기고 outlier를 숨김.
- debug/production flag, LTO, PGO, allocator가 다름.
- benchmark process와 load generator가 같은 core·LLC·NUMA 자원을 경쟁함.

Assembly를 확인하고, baseline empty loop와 clock overhead를 비교한다. 단순 subtraction은 noise가 독립이라는 보장이 없으므로 raw distribution도 유지한다.

#### 꼬리 질문

- **warm-up은 얼마나 해야 하는가?** 시간 상수가 안정화됐음을 metric으로 판단한다. 고정 횟수는 보편 답이 아니다.
- **cache를 flush하고 재면 현실적인가?** cold-start를 묻는다면 맞지만 steady-state hot path를 묻는다면 틀릴 수 있다.
- **benchmark framework의 `DoNotOptimize`면 충분한가?** compiler 제거는 막아도 workload 대표성, CPU noise, clock 오류는 해결하지 않는다.

#### 저지연 실무 연결

unit microbenchmark, component replay, end-to-end replay를 계층적으로 운영한다. 코드 변경은 instruction-level 개선뿐 아니라 실제 p99.9와 처리 정확성까지 통과해야 한다.

---

### Q15. 각 단계의 p99를 더하면 end-to-end p99인가? PMU counter로 원인을 확정할 수 있는가?

#### 면접용 핵심 답변

둘 다 아니다. 각 단계의 p99가 같은 요청에서 발생한다는 보장이 없으므로 합은 end-to-end p99가 아니다. 단계 사이의 의존 구조와 각 분포에 따라 합이 과대 또는 과소 추정될 수 있으며, marginal percentile만으로 joint percentile을 복원할 수 없다. 동일 trace/sample의 구간 timestamp를 합쳐 end-to-end 분포를 직접 계산해야 한다. PMU counter는 cache miss, branch miss, stalled cycle과의 상관을 보여주지만 그 자체로 인과를 확정하지 않는다.

#### 내부 동작 상세

```text
request A: network 1us, strategy 100us
request B: network 100us, strategy 1us
```

이 단순한 upper-tail 예시는 서로 다른 단계의 느린 관측값이 같은 요청에서 발생하지 않을 수 있음을 보여준다. 정확한 percentile은 충분한 표본과 명시한 quantile 계산법으로 전체 분포에서 구해야 한다. Queue 대기와 service time도 분리해야 한다. 가능한 경우 하나의 message ID에 NIC, receive, parse, decision, send timestamp를 연결한다.

PMU는 일정 이벤트를 count하지만 다음 제약이 있다.

- 사용 가능한 hardware counter보다 이벤트가 많으면 multiplexing된다.
- sampling event의 instruction attribution에는 skid가 있을 수 있다.
- speculative event와 retired event 의미가 다르다.
- cache miss가 원인일 수도, 다른 stall로 실행이 늘어난 결과일 수도 있다.

#### 꼬리 질문

- **branch miss가 줄었는데 latency가 늘 수 있는가?** branchless 변환으로 instruction 수, dependency chain, memory traffic이 늘 수 있다.
- **CPU utilization이 낮으니 CPU 문제는 아닌가?** 평균 utilization은 짧은 core saturation, memory stall, single-thread bottleneck을 숨긴다.
- **단계별 timestamp 자체의 overhead는?** 읽기·저장·cache pollution을 측정하고 sampling 또는 별도 buffer로 제한한다.

#### 저지연 실무 연결

outlier trace를 기준으로 scheduler event, page fault, IRQ, PMU sample, queue depth를 시간축에 맞춘다. Counter 하나를 보고 최적화하지 않고 hypothesis를 만든 뒤 A/B 실험으로 검증한다.

---

## 5.4 트레이딩 도메인·복구·신뢰성

### Q16. Incremental market data로 order book을 어떻게 정확하게 재구성하는가?

#### 면접용 핵심 답변

Feed의 sequence와 venue가 정의한 이벤트 의미를 기준으로 **연속된 update만 결정적으로 적용**한다. 다음 예상 sequence보다 작은 duplicate/old event는 규칙에 따라 무시하고, 큰 sequence가 오면 gap으로 판단해 현재 book을 신뢰 불가 상태로 전환한다. Gap을 숨긴 채 이후 update를 계속 적용하면 형태는 그럴듯하지만 잘못된 가격으로 주문할 수 있으므로 거래 가능 여부를 명시적으로 차단하거나 제한해야 한다.

#### 내부 동작 상세

book builder는 적어도 다음 상태를 가진다.

- 현재 session/channel과 마지막 적용 sequence
- symbol별 또는 channel별 validity
- price level/order ID 자료구조
- snapshot/recovery 진행 상태
- duplicate·out-of-order·gap 통계

Feed가 order-level인지 price-level인지, add/modify/delete의 의미, trade event가 book quantity를 직접 바꾸는지, sequence가 channel 공통인지 symbol별인지가 venue마다 다르다. 따라서 “표준 order book 알고리즘”보다 protocol contract가 우선이다.

#### 꼬리 질문

- **UDP reorder를 잠깐 기다리면 되는가?** 짧은 reorder buffer를 둘 수 있지만 기다리는 동안 freshness를 잃는다. venue의 dual-feed/recovery 방식과 latency budget으로 결정한다.
- **duplicate는 항상 무시하면 되는가?** 동일 session·sequence·payload임을 확인해야 한다. session reset이나 sequence wrap 규칙을 고려한다.
- **gap 동안 기존 book으로 거래할 수 있는가?** 위험 정책에 따른다. 일반적으로 aggressive order는 막고 cancel 등 exposure 감소 동작만 허용하는 식의 fail-safe가 필요하다.

#### 저지연 실무 연결

Book object에 `Valid`, `GapDetected`, `Recovering`, `CaughtUp` 상태를 둔다. 전략은 가격 값뿐 아니라 validity와 data age를 함께 확인해야 하며, gap을 metric과 audit log로 남긴다.

---

### Q17. Snapshot을 받는 동안 incremental update가 계속 오면 어떻게 합치는가?

#### 면접용 핵심 답변

먼저 incremental을 구독해 buffer에 쌓고 snapshot을 요청한다. Snapshot이 기준 sequence `S`의 상태라면 buffer에서 `S` 이하를 버리고 `S+1`부터 끊김 없이 적용한다. 그 사이에도 gap이 있거나 snapshot과 incremental의 sequence 의미가 맞지 않으면 다시 복구한다. 정확한 순서는 venue protocol이 snapshot에 어떤 sequence를 부여하는지에 따라 달라진다.

#### 내부 동작 상세

```text
1. Incremental 수신 시작 및 bounded recovery buffer 저장
2. Snapshot 요청/수신
3. Snapshot 자체 무결성 검증
4. Snapshot 기준 sequence S 확인
5. buffered update 중 <= S 제거
6. S+1부터 contiguous apply
7. 수신 producer와 같은 sequence domain에서 drain 완료를 handshake한 뒤 CaughtUp으로 전환
```

Atomic flag 하나만 바꾸는 것으로는 충분하지 않다. Consumer가 buffer가 비었다고 확인한 직후 producer가 update를 넣는 handoff race를 막으려면 단일 sequenced consumer를 유지하거나 lock·epoch handshake를 사용해 recovery와 live 처리 사이에 빈틈이 없음을 보장해야 한다.

Recovery buffer가 가득 차면 오래된 update를 조용히 덮어쓰지 말고 복구 실패로 처리한다. Snapshot 생성 시점과 전송 시점은 다를 수 있으므로 timestamp만으로 merge하면 안 된다. Dual multicast feed가 있다면 두 feed의 동일 sequence를 deduplicate하고 먼저 도착한 정상 packet을 활용할 수 있다.

#### 꼬리 질문

- **snapshot을 먼저 받고 incremental을 구독하면 안 되는가?** 둘 사이의 update를 잃는 race가 생긴다.
- **snapshot 적용 중 전략 thread가 book을 보면?** 별도 builder에 구성한 뒤 version/pointer swap하거나, 명확한 invalid 상태로 가려야 한다.
- **buffer를 무한히 키우면 복구되는가?** memory 폭주와 stale recovery를 만든다. 시간·크기 상한과 재시도 정책이 필요하다.

#### 저지연 실무 연결

복구 경로도 production rate의 replay test를 해야 한다. 정상 경로가 빠르더라도 snapshot parse와 buffered catch-up이 CPU를 독점해 다른 symbol을 지연시키지 않게 격리한다.

---

### Q18. Cancel 요청과 fill이 동시에 발생하면 주문 상태를 어떻게 관리하는가?

#### 면접용 핵심 답변

Cancel을 보냈다는 사실은 취소 완료가 아니다. Venue가 fill을 먼저 실행했거나 cancel 처리 중 추가 fill이 올 수 있다. Local state는 `Live → CancelPending`으로 바뀌어도 executable quantity와 risk를 유지하고, execution report를 sequence 검증과 deduplication 후 authoritative event로 적용한다. `Filled`와 `Canceled` 같은 terminal 결과는 venue가 정의한 event ordering, status와 quantity 의미를 기준으로 일관되게 결정한다.

#### 내부 동작 상세

예시 상태:

```text
NewPending → Live → PartiallyFilled
Live/PartiallyFilled -- cancel request --> CancelPending
CancelPending -- partial fill --> CancelPending(cumQty/leavesQty 갱신)
CancelPending -- full fill --> Filled
CancelPending -- cancel ack --> Canceled
CancelPending -- cancel reject --> Live/PartiallyFilled/Unknown
NewPending -- new reject --> Rejected
```

`CancelPending`은 “현재까지 얼마나 체결됐는가”와 별개의 pending operation이다. 구현에서는 두 차원을 분리하거나 `PartiallyFilled + CancelPending` 같은 합성 상태로 표현해 partial fill이 cancel 요청 자체를 지워버리지 않게 한다. Cancel reject 이후 상태는 venue가 보고한 결과와 reconciliation에 따라 결정한다.

중요 invariant:

- `cumQty`는 일반 fill만 처리하는 동안 단조 증가하지만, trade cancel·bust·correct는 명시적인 correction event로 적용되어 감소할 수 있다.
- Active order에서는 보통 `LeavesQty = OrderQty - CumQty`이지만, canceled·expired·rejected처럼 inactive인 주문의 executable leaves는 0일 수 있다. 미체결 잔량과 지금 실행 가능한 잔량을 구분한다.
- cancel pending이어도 fill을 정상 반영한다.
- terminal 상태 이후 늦게 온 중복 event는 idempotently 처리한다.
- transport disconnect를 주문 취소로 간주하지 않는다.

Cancel reject는 “주문이 반드시 live”라는 의미가 아닐 수 있다. 이미 filled/canceled, unknown order, race 등 reason을 해석하고 drop copy나 query로 reconciliation해야 한다.

#### 꼬리 질문

- **cancel ack 뒤 fill이 오면 버그인가?** Venue가 정의한 event ordering과 trade bust/correction 규칙을 먼저 봐야 한다. 단순히 버리는 것이 더 위험하다.
- **socket이 끊기면 outstanding order는?** 세션/venue의 cancel-on-disconnect 설정을 맹신하지 말고 별도 연결로 상태를 조회·reconcile한다.
- **local timeout이면 canceled인가?** 아니다. 상태를 `Unknown`으로 두고 신규 위험 증가를 막으며 확인해야 한다.

#### 저지연 실무 연결

Order state transition을 단일 reducer처럼 결정적으로 구현하고 모든 입력 event를 journal한다. Race scenario를 property test와 replay fixture로 반복한다.

---

### Q19. Duplicate 또는 out-of-order execution report로 position이 두 번 반영되는 것을 어떻게 막는가?

#### 면접용 핵심 답변

Venue가 제공하는 session sequence, execution ID, order ID와 cumulative quantity를 사용해 idempotency key와 monotonic invariant를 만든다. 동일 execution을 재수신해도 position delta를 한 번만 반영해야 한다. 단순히 “마지막 sequence보다 작으면 버린다”만으로는 reconnect, session reset, multi-channel report를 처리하지 못하므로 venue별 identity와 recovery contract가 필요하다.

#### 내부 동작 상세

- FIX라면 MsgSeqNum gap/duplicate flag와 PossDup, ExecID, CumQty 등 의미를 함께 해석한다.
- Incremental `LastQty`만 누적하기보다 `CumQty`와 기존 상태 차이를 검증하면 duplicate에 강해질 수 있다.
- 같은 ExecID의 payload가 다르면 조용히 하나를 택하지 말고 protocol violation 또는 correction으로 처리한다.
- Dedup set을 무한히 보관할 수 없으므로 session boundary, durable checkpoint, bounded window 정책이 필요하다.
- Trade cancel/bust/correct는 기존 execution을 되돌리거나 수정하는 별도 business event이지 duplicate가 아니다.

#### 꼬리 질문

- **database unique key면 exactly-once인가?** database 내부 insert는 중복 방지할 수 있지만 venue side effect와 network delivery까지 하나의 transaction이 되지는 않는다.
- **sequence gap이면 이후 report를 버리는가?** buffer하고 resend/recovery를 요청하되 risk view는 보수적으로 관리한다.
- **CumQty가 감소하면?** 일반 fill이라면 invariant 위반이지만 bust/correction protocol인지 확인해야 한다.

#### 저지연 실무 연결

Position update와 processed-execution 기록을 같은 durable transaction 또는 재생 가능한 journal event로 묶는다. 재접속과 duplicate storm을 fault-injection test에 포함한다.

---

### Q20. Pre-trade risk에서 주문과 replace의 exposure는 언제 예약·해제하는가?

#### 면접용 핵심 답변

주문을 외부로 보내기 전에 최악의 실행 가능 exposure를 원자적으로 예약해야 한다. Partial fill이 오면 해당 수량은 open-order reservation에서 실제 position/exposure로 이동한다. Cancel 요청 시점이 아니라 venue가 취소를 확정하거나 주문이 terminal임을 확인한 뒤 나머지 reservation을 해제한다. Replace는 venue가 atomic cancel/replace를 제공하는지에 따라 old와 new가 동시에 실행될 가능성을 포함해 보수적으로 예약한다.

#### 내부 동작 상세

Risk는 quantity만이 아니라 price/notional, side, instrument·underlying aggregate, credit, fat-finger limit를 볼 수 있다. 동시 전략 thread가 같은 limit을 검사한 뒤 둘 다 통과하는 TOCTOU를 막으려면 check-and-reserve가 하나의 synchronization domain에서 일어나야 한다.

Replace 예시:

- 새 수량이 증가하면 적어도 증가분 또는 old/new 동시 노출의 venue worst case를 먼저 확보한다.
- 감소 replace라도 ack 전에는 old quantity가 체결될 수 있으므로 곧바로 전부 해제하면 안 된다.
- replace reject 시 old order가 계속 live인지 protocol 의미를 확인한다.

#### 꼬리 질문

- **risk check를 lock-free로 만들면 정확한가?** 자료구조 progress와 여러 limit의 원자적 invariant는 별개다. CAS 하나로 복합 risk가 보장되는지 증명해야 한다.
- **cancel pending 수량을 limit에서 빼도 되는가?** 일반적으로 아직 실행 가능하므로 빼면 안 된다.
- **market order notional은 어떻게 잡는가?** price collar, worst acceptable price, available liquidity와 정책상 상한을 사용한다.

#### 저지연 실무 연결

Risk reservation ledger를 order state machine과 같은 event stream으로 갱신한다. 정상 throughput뿐 아니라 gap, reconnect, reject, partial-fill/cancel race 뒤에도 `reserved + realized` invariant가 맞는지 replay 검증한다.

---

### Q21. 가격과 수량 계산에 floating point 대신 fixed-point를 쓰는 이유는?

#### 면접용 핵심 답변

금융 protocol은 tick, lot, decimal scale이 명확하며 binary floating point는 `0.1` 같은 값을 정확히 표현하지 못한다. 비교·반올림 경계에서 서로 다른 결과가 나면 order validation, book key, PnL이 틀릴 수 있다. 따라서 가격을 tick 수 또는 정수 mantissa와 scale로 표현하고 모든 변환·반올림 규칙을 명시하는 것이 일반적이다.

#### 내부 동작 상세

Fixed-point도 자동으로 안전하지 않다.

- `price * quantity`에서 64-bit overflow가 날 수 있어 범위를 증명하거나 넓은 intermediate를 쓴다.
- Instrument마다 scale과 tick table이 다를 수 있다.
- Tick size가 가격 구간에 따라 달라지는 시장도 있다.
- 외부 decimal 문자열 parse 시 digit, sign, precision, overflow를 검사한다.
- division과 currency conversion의 rounding mode를 business rule로 고정한다.
- “센트 정수”는 소수 자릿수가 다른 상품에 보편적 표현이 아니다.

#### 꼬리 질문

- **double 비교에 epsilon을 쓰면 되지 않는가?** 규제된 tick 경계는 모호한 epsilon이 아니라 정확한 decimal/tick 규칙이어야 한다.
- **fixed-point가 항상 더 빠른가?** 정확성이 주된 이유다. overflow 검사, scale 변환 비용까지 실제로 측정해야 한다.
- **PnL은 같은 scale이면 충분한가?** quantity, multiplier, FX, fee가 결합되므로 intermediate 범위와 scale 정책이 필요하다.

#### 저지연 실무 연결

Instrument metadata 로딩 시 scale/tick 변환기를 생성해 hot path의 division을 줄인다. 주문을 보내기 직전에 price가 tick grid와 venue 범위를 만족하는지 integer 연산으로 검증한다.

---

### Q22. Bounded queue가 가득 찼을 때 market data와 execution report를 어떻게 처리하는가?

#### 면접용 핵심 답변

Queue full은 예외가 아니라 overload 정책이 실행되는 정상 설계 지점이다. 데이터 중요도에 따라 정책이 달라야 한다. Market data는 일부 protocol에서 최신 상태로 coalesce하거나 drop 후 book을 invalid하게 만들고 recovery할 수 있다. 반면 execution report와 risk/control event를 조용히 버리면 position과 주문 상태가 틀어지므로 durable handoff, 즉시 fail-safe, producer throttling, 예약된 capacity 또는 별도 처리 lane이 필요하다. 별도 lane을 사용하더라도 reducer에 합칠 때 venue sequence와 causal ordering을 보존해야 한다.

#### 내부 동작 상세

선택지는 다음과 같다.

- **spin/retry:** 짧은 burst에는 유효하지만 downstream 정지가 길면 upstream core까지 소모한다.
- **block:** loss를 막을 수 있지만 network ring까지 역압이 전파되어 packet drop과 head-of-line blocking을 만들 수 있다.
- **drop newest/oldest:** semantic상 허용 여부를 명시해야 한다. incremental book update 하나를 버리면 이후 전체 상태가 무효가 된다.
- **coalesce:** absolute price-level update처럼 최신 값만 의미 있을 때 가능할 수 있으나 order-level delta에는 일반적으로 안전하지 않다.
- **priority/separate lane:** 중요한 event의 자원을 보장할 수 있지만, merge 시 execution·order-control event의 원래 순서를 복원해야 한다.
- **shed load/fail closed:** 신규 order를 막고 cancel·execution·risk message에 자원을 보존한다.

Queue depth가 크면 drop은 늦어지지만 stale message를 처리하며 잘못된 결정을 내릴 수 있다. 따라서 capacity뿐 아니라 message age와 processing lag를 limit으로 둔다.

#### 꼬리 질문

- **ring을 크게 하면 backpressure가 해결되는가?** burst 흡수 시간만 늘고 지속 overload는 해결하지 못한다.
- **execution queue producer를 block하면 안전한가?** 그 producer가 network receive까지 담당하면 다른 중요한 event 수신도 막을 수 있다.
- **kill switch도 같은 queue로 보내도 되는가?** 혼잡 시 전달되지 않을 수 있으므로 독립 경로와 우선순위, 직접 차단 수단이 필요하다. Kill switch는 일반 event 순서를 우연히 깨뜨리는 것이 아니라 의도적으로 신규 위험을 즉시 차단하는 별도 safety action으로 정의한다.

#### 저지연 실무 연결

Queue마다 capacity, high-watermark, oldest age, full count, drop/coalesce 정책을 문서화한다. 장애 테스트에서 consumer pause를 주입해 신규 주문 차단과 execution 보존이 실제로 동작하는지 확인한다.

---

### Q23. 주문을 보낸 직후 프로세스가 죽으면 어떻게 복구하며, exactly-once가 가능한가?

#### 면접용 핵심 답변

프로세스와 venue를 하나의 원자적 transaction으로 묶을 수 없으므로 end-to-end exactly-once send를 일반적으로 보장할 수 없다. Journal을 먼저 쓰면 “기록했지만 전송하지 못한” 상태가 생기고, 전송을 먼저 하면 “venue는 받았지만 로컬 기록이 없는” 상태가 생긴다. 실무에서는 unique client order ID, durable write-ahead intent/event log, at-least-once retry의 idempotency, venue open-order/execution reconciliation을 결합해 불확실 상태를 복구한다.

#### 내부 동작 상세

복구에 필요한 정보:

- stable client/session/order identifier
- outbound intent와 전송/ack 상태
- inbound execution의 dedup identity
- last confirmed venue sequence/checkpoint
- risk reservation과 position event
- binary/config/reference-data version

Durability를 위해 매 주문마다 `fsync`하면 tail이 커질 수 있다. Group commit은 비용을 분산하지만 durability window를 만든다. 배터리 백업 장치, replicated log도 각자 failure model이 있으므로 “write 반환 = 영구 저장”이라고 단정하지 않는다.

Deterministic replay는 기록된 입력을 같은 순서로 넣었을 때 같은 state transition을 재현하는 것이다. Wall clock, random seed, thread race, 외부 reference data, unordered iteration, floating-point 차이까지 통제하거나 입력으로 기록해야 한다. Replay 결과만 믿지 말고 최종 상태를 venue drop copy/조회와 reconcile한다.

#### 꼬리 질문

- **WAL을 쓰고 나서만 전송하면 중복이 없는가?** 전송 후 ack 전에 죽으면 재시도 여부가 불확실해 중복 가능성이 남는다. Venue idempotency key와 조회가 필요하다.
- **TCP ack를 받았으면 venue가 주문을 처리했는가?** 전송 계층 전달과 business acceptance는 다르다.
- **replay가 같으면 시스템이 정확한가?** 결정적 버그도 똑같이 재현된다. Invariant와 외부 truth로 검증해야 한다.

#### 저지연 실무 연결

Restart 시 신규 주문을 즉시 허용하지 않고 journal 복구, session recovery, venue reconciliation, risk invariant 검증을 끝낸 뒤 gate를 연다. “Unknown” 주문을 명시적 상태로 유지하고, 운영자가 확인할 수 있는 audit trail을 제공한다.

---

# 마지막 점검표

한 주제를 공부한 뒤 다음 항목을 모두 만족하는지 확인한다.

- 30초 핵심 답변과 3분 상세 답변을 각각 말할 수 있다.
- 언어 표준의 보장과 특정 CPU·kernel의 구현을 구분한다.
- 정상 흐름뿐 아니라 race, overflow, packet loss와 restart를 설명한다.
- 성능 개선이 correctness invariant를 깨지 않는다는 근거가 있다.
- 평균뿐 아니라 p99, p99.9, max와 drop을 함께 본다.
- Compiler output, PMU, kernel counter 또는 replay 중 하나로 주장을 검증한다.
- 모르는 구현 세부 사항은 확인할 공식 문서의 종류를 말할 수 있다.

# 참고 자료

- [ISO C++ Working Draft — Object Lifetime](https://eel.is/c++draft/basic.life)
- [ISO C++ Working Draft — Constructors and Destructors in Exception Handling](https://eel.is/c++draft/except.ctor)
- [ISO C++ Working Draft — Atomics Order](https://eel.is/c++draft/atomics.order)
- [ISO C++ Working Draft — Mutex Requirements](https://eel.is/c++draft/thread.mutex.requirements)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [Intel® 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html)
- [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [AMD Software Optimization Guide for AMD EPYC Processors](https://docs.amd.com/v/u/en-US/56305)
- [Arm Architecture Reference Manual](https://developer.arm.com/documentation/ddi0487/latest/)
- [System V AMD64 ABI](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Linux Kernel — CPU Isolation](https://docs.kernel.org/admin-guide/cpu-isolation.html)
- [Linux Kernel — NAPI](https://docs.kernel.org/networking/napi.html)
- [Linux Kernel — Scaling in the Linux Networking Stack](https://docs.kernel.org/networking/scaling.html)
- [Linux Kernel — Dynamic DMA Mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)
- [Linux Kernel — Timestamping](https://docs.kernel.org/networking/timestamping.html)
- [Linux man-pages — sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)
- [Linux man-pages — mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html)
- [Linux man-pages — futex(2)](https://man7.org/linux/man-pages/man2/futex.2.html)
- [Linux perf-stat Manual](https://man7.org/linux/man-pages/man1/perf-stat.1.html)
- [DPDK — Writing Efficient Code](https://doc.dpdk.org/guides/prog_guide/writing_efficient_code.html)
- [RFC 768 — User Datagram Protocol](https://www.rfc-editor.org/rfc/rfc768.html)
- [RFC 9293 — Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293.html)
- [FIX Trading Community — FIX Session Layer](https://www.fixtrading.org/standards/fix-session-layer-online/)
- [FIX Trading Community — Order State Change Matrices](https://www.fixtrading.org/online-specification/order-state-changes/)
- [Nasdaq — TotalView-ITCH 5.0 Specification](https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf)
- [Nasdaq — OUCH 5.0 Specification](https://nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/Ouch5.0.pdf)
