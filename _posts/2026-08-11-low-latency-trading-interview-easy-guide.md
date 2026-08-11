---
title: '[Low Latency Trading] 면접 기초 해설 — CPU에서 주문 복구까지 처음부터 이해하기'
date: 2026-08-11 01:30:00 +09:00
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

[[Low Latency Trading] 면접 심화 질문과 답변](/posts/low-latency-trading-interview-deep-dive/)에는 실제 면접에서 사용할 수 있는 56개의 답변이 들어 있다. 하지만 처음부터 그 글을 읽으면 `retire`, `coherence`, `happens-before`, `NAPI` 같은 단어가 한꺼번에 등장한다. 단어의 정의를 읽어도 전체 그림이 없으면 오래 기억하기 어렵다.

이 글은 그 심화판을 읽기 위한 **쉬운 해설판**이다. 설명 순서는 다음과 같다.

```text
왜 이런 문제가 생기는가?
  → 처음 등장하는 기술 용어의 뜻을 먼저 확인한다
  → 실제 컴퓨터에서는 무엇이 움직이는지 본다
  → 서로 헷갈리는 개념의 경계를 나눈다
  → 30초 면접 답변으로 압축한다
  → 심화판에서 정확한 용어와 예외를 공부한다
```

영어 용어를 억지로 번역하면 오히려 실제 문서와 code를 읽기 어려울 수 있다. 따라서 `retire`, `cache line`, `publication` 같은 원래 용어는 유지하되, 처음 나오는 자리에서 **무엇을 가리키는 말인지**, **어떤 사건과 구분해야 하는지**를 풀어 쓴다.

이 글을 처음 읽을 때 모든 세부 사항을 암기하지 않아도 된다. 각 절의 **한 문장 기억법**과 **30초 답변**을 먼저 익히고, 두 번째 읽을 때 내부 동작을 연결하면 된다.

# 0. 먼저 전체 시스템을 한 장으로 본다

## 0.1 이 글에서 계속 사용하는 기본 용어

| 용어 | 쉬운 설명 |
| --- | --- |
| Latency | 작업 하나가 시작해서 끝날 때까지 걸린 시간 |
| Throughput | 일정 시간 동안 처리한 작업의 수 |
| Tail latency | 전체 측정값 중 드물게 매우 느린 쪽의 latency |
| Hot path | 정상 운영 중 매우 자주 실행되어 성능에 직접 영향을 주는 code 경로 |
| Fast path | 흔한 입력을 짧게 처리하도록 만든 경로. Hot path와 겹칠 수 있지만 같은 뜻은 아니다 |
| Slow path | 예외 상황이나 드문 조건을 처리하는 상대적으로 긴 경로 |
| State | 주문 상태, 수량처럼 system이 기억하고 다음 판단에 사용하는 값 |
| State transition | Event를 적용해 state가 이전 값에서 다음 값으로 바뀌는 과정 |
| Invariant | 어떤 정상 event를 처리한 뒤에도 반드시 참이어야 하는 조건 |
| Metadata | 실제 payload를 설명하는 부가 정보. 길이, sequence, timestamp 등이 해당한다 |
| Ownership | 현재 어느 thread, core 또는 device가 state를 변경하거나 buffer를 재사용할 권한이 있는지 나타내는 규칙 |
| Publication | 한 실행 주체가 완성한 data를 다른 실행 주체가 안전하게 읽을 수 있도록 공개하는 과정 |
| Visibility | 한 core의 write를 다른 core가 관찰할 수 있게 된 상태. C++ correctness는 visibility라는 말만으로 증명하지 않고 happens-before를 사용한다 |
| Ordering | 여러 operation이 다른 실행 주체에게 어떤 순서로 관찰될 수 있는지에 대한 규칙 |
| Contention | 여러 thread나 core가 같은 lock, cache line, queue 같은 자원을 동시에 사용하려고 경쟁하는 상황 |
| Queueing | 처리되지 못한 작업이 queue에 쌓여 service를 기다리는 현상 |
| Backpressure | Consumer의 처리 지연이 producer나 더 앞 단계까지 전달되는 현상 |
| Deterministic | 같은 초기 state와 같은 입력 순서를 주면 같은 결과를 만드는 성질 |
| Protocol | 통신 당사자가 message 형식, 순서와 오류 처리를 합의한 규칙 |
| Venue | 주문을 받거나 market data를 제공하는 거래소·거래 시스템을 통칭하는 표현 |

이 용어들은 뒤에서 더 정확한 의미로 좁혀진다. 예를 들어 CPU의 visibility, C++의 happens-before와 device DMA completion은 서로 관련되지만 같은 보장은 아니다.

## 0.2 주문 한 번에 어떤 길을 지나가는가?

저지연 트레이딩 프로그램은 단순히 빠른 알고리즘 하나가 아니다. 시장 데이터가 들어오고 주문이 나가며 체결 결과가 돌아오는 전체 경로다.

```text
거래소
  │ market data packet
  ▼
NIC → DMA buffer → Linux network path 또는 kernel bypass
  → feed parser → sequence 검사 → order book 갱신
  → strategy 판단 → pre-trade risk 검사
  → order encoder → NIC → 거래소
  │
  └──────────────── execution report / fill / cancel 결과
                         ↓
               order state · position · risk 갱신
                         ↓
                    event log · replay
```

이 길에서 지연이 생기는 원인은 크게 세 가지다.

| 원인 | 쉬운 뜻 | 예시 |
| --- | --- | --- |
| 해야 할 일이 많다 | 계산량 자체가 많다 | 복잡한 parsing, 불필요한 copy |
| 기다려야 한다 | 다른 자원이나 작업을 기다린다 | cache miss, lock, queue |
| 평소와 다른 일이 끼어든다 | 드문 운영체제·하드웨어 사건이다 | page fault, interrupt, context switch |

평균 latency가 낮아도 세 번째 종류가 가끔 발생하면 p99.9가 커진다. 그래서 저지연 개발자는 “코드가 몇 줄인가?”보다 “무엇을 기다리며, 가장 느린 실행에서는 무슨 일이 끼어드는가?”를 묻는다.

### 한 문장 기억법

> 저지연 시스템은 계산을 빨리 하는 문제인 동시에, 기다림과 드문 사건을 통제하는 문제다.

## 0.3 빠름과 정확함은 따로 떨어져 있지 않다

주문 처리에서 가장 빠른 프로그램이 상태를 틀리게 계산하면 쓸 수 없다. 반대로 항상 정확하지만 시장 데이터가 밀린 뒤 오래된 가격으로 주문을 내도 위험하다.

따라서 모든 최적화에는 두 질문이 따라온다.

1. **정확성:** 이 변경 뒤에도 주문·포지션·리스크 불변식이 유지되는가?
2. **시간:** 정상 부하와 폭주 상황에서 latency 분포가 어떻게 바뀌는가?

예를 들어 execution report queue가 가득 찼다고 체결 메시지를 버리면 프로그램은 빨리 계속 움직일 수 있다. 하지만 실제 포지션과 프로그램의 포지션이 달라진다. 이것은 성능 개선이 아니라 correctness failure다.

## 0.4 면접 답변은 네 단계로 말한다

낯선 질문을 받아도 다음 틀을 사용하면 답이 무너지지 않는다.

```text
1. 결론: 둘은 같은가, 다른가? 무엇을 보장하는가?
2. 이유: 내부에서 어떤 자원과 상태가 움직이는가?
3. 경계: 언어 표준의 보장인가, 특정 CPU·Linux 구현인가?
4. 실무: latency와 correctness에 어떤 영향을 주며 어떻게 측정하는가?
```

예를 들어 “atomic이면 빠릅니까?”에는 이렇게 답한다.

> Atomic은 공유 객체의 원자성과 순서 규칙을 표현하는 수단이지 빠르다는 뜻은 아닙니다. 경쟁이 없으면 저렴할 수 있지만, 여러 코어가 같은 cache line을 수정하면 ownership 이동 때문에 느려질 수 있습니다. 대상 CPU에서 instruction과 cache-to-cache traffic, tail latency를 측정해야 합니다.

# 1. CPU와 메모리: 코드는 실제로 어떻게 움직이는가?

## 1.1 CPU는 한 명령씩 끝내고 다음 명령을 시작하지 않는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Instruction | Program이 CPU에 요구하는 명령. Load, add, branch 등이 있다 |
| uop | 복잡한 instruction을 CPU 내부에서 실행하기 쉬운 단위로 나눈 작업 |
| Pipeline | 여러 instruction의 처리 단계를 시간상 겹쳐 실행하는 구조 |
| Front-end | Instruction을 가져오고 예측하고 decode해 uop을 공급하는 부분 |
| Back-end | uop의 operand를 준비하고 실행 장치에 보내 결과를 만드는 부분 |
| Register | CPU가 연산에 바로 사용하는 매우 작은 저장 공간 |
| Dependency | 앞 작업의 결과가 있어야 뒤 작업을 실행할 수 있는 관계 |
| Speculation | 아직 확정되지 않은 분기나 memory 관계를 예측해 미리 실행하는 것 |
| Architectural state | Program이 관찰하도록 CPU architecture가 정의한 register와 memory 상태 |
| Retire | 실행 결과를 취소할 수 없는 program의 결과로 확정하는 단계 |

Pipeline은 서로 다른 instruction이 서로 다른 처리 단계에 동시에 있도록 만든다.

```text
시간 1: instruction A fetch
시간 2: instruction A decode  + instruction B fetch
시간 3: instruction A execute + instruction B decode + instruction C fetch
```

CPU pipeline도 instruction을 fetch, decode, execute, retire 같은 단계로 겹쳐 처리한다. 현대 CPU는 여기서 더 나아가, 앞 명령이 memory를 기다리는 동안 **그 결과와 무관한 뒤 명령**을 먼저 실행한다. 이것이 out-of-order execution이다.

### 실제 실행 단계

```text
fetch → branch predict → decode → rename → issue/execute → retire
```

- **Fetch:** 다음 instruction bytes를 가져온다.
- **Branch prediction:** 분기의 다음 경로를 미리 고른다.
- **Decode:** instruction을 CPU 내부 작업인 uop으로 바꾼다.
- **Rename:** 같은 architectural register 이름 때문에 생기는 거짓 의존성을 없앤다.
- **Issue/execute:** operand와 실행 장치가 준비된 uop부터 실행한다.
- **Retire:** 프로그램 순서대로 결과를 확정한다.

뒤 명령을 먼저 실행해도 retire는 프로그램 순서를 지킨다. 그래야 앞 명령에서 예외가 났을 때 뒤의 추측 실행을 버리고 정확한 상태로 돌아갈 수 있다.

### 꼭 구분하기

- **실행이 끝남:** 실행 장치가 결과를 계산했다.
- **Retire됨:** 그 결과가 취소되지 않을 architectural 작업으로 확정됐다.
- **다른 코어가 관찰함:** memory ordering과 coherence를 통해 외부에 보이게 됐다.

세 사건은 같은 순간이 아니다.

### 한 문장 기억법

> CPU는 준비된 일을 순서와 다르게 실행할 수 있지만, 프로그램에 결과를 확정할 때는 순서를 복원한다.

### 30초 면접 답변

> 현대 CPU는 instruction을 uop으로 바꾸고 register rename으로 거짓 의존성을 제거한 뒤, operand가 준비된 uop부터 out-of-order로 실행합니다. 하지만 정확한 예외와 architectural state를 위해 reorder buffer에서 프로그램 순서대로 retire합니다. 따라서 실행 순서와 retire 순서는 다르며, cache miss나 branch miss가 독립 작업의 범위를 넘으면 pipeline이 정체됩니다.

## 1.2 Store를 실행했다고 다른 코어가 즉시 보는 것은 아니다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Load | Memory의 값을 CPU register로 읽는 operation |
| Store | CPU가 계산한 값을 memory 위치에 쓰는 operation |
| Store buffer | 아직 cache hierarchy에 완전히 반영되지 않은 store를 core 내부에서 보관하는 구조 |
| Visibility | 한 core의 write를 다른 core가 관찰할 수 있게 된 상태 |
| Publication | Payload를 먼저 완성한 뒤, 다른 thread가 안전하게 읽도록 알리는 동기화 과정 |
| Store-to-load forwarding | 같은 core의 뒤 load가 앞 store의 값을 cache 대신 store buffer에서 직접 받는 동작 |
| Disambiguation | 앞 store와 뒤 load의 주소가 겹치는지 CPU가 판단하거나 예측하는 과정 |

Store instruction 하나에도 실행, retire, 다른 core의 관찰과 DRAM write-back이라는 서로 다른 사건이 있다.

CPU store도 여러 사건으로 나뉜다.

```text
store 실행
  → store buffer에 들어감
  → retire
  → cache line write ownership 획득
  → 다른 코어가 새 값을 관찰 가능
  → 언젠가 dirty line이 DRAM에 write-back
```

Store buffer는 CPU가 cache ownership을 기다리는 동안 다음 일을 계속하게 해준다. 같은 코어의 뒤 load는 cache 반영 전이라도 store buffer에서 값을 전달받을 수 있다. 이를 store-to-load forwarding이라 한다. 그래서 “내 코어가 새 값을 읽었다”가 “다른 코어도 이미 읽을 수 있다”는 뜻은 아니다.

### Publication 예제

Producer가 payload를 채운 뒤 consumer에게 준비됐다고 알려야 한다.

```cpp
// producer
payload = 42;
ready.store(true, std::memory_order_release);

// consumer
if (ready.load(std::memory_order_acquire)) {
    use(payload);
}
```

Release/acquire는 단순히 CPU cache를 빨리 갱신하라는 명령이 아니다. Consumer가 release로 저장된 `true`를 acquire로 읽었을 때 payload write가 consumer의 payload read보다 먼저라는 C++ 관계를 만든다.

### Load/store disambiguation도 함께 이해하기

CPU는 앞 store의 주소가 아직 완전히 계산되지 않았을 때 뒤 load가 겹치지 않을 것이라고 예측해 먼저 실행할 수 있다. 나중에 같은 주소였음이 밝혀지면 load와 뒤 작업을 다시 실행한다. 이를 memory-order violation replay라고 생각하면 된다.

### 한 문장 기억법

> Retire는 내 명령의 확정이고, visibility는 다른 코어가 관찰할 수 있게 되는 별도의 사건이다.

### 30초 면접 답변

> Store는 retire된 뒤에도 store buffer에서 cache-line ownership을 기다릴 수 있으므로 retire와 다른 코어에 대한 visibility는 다릅니다. 같은 코어의 load는 store forwarding으로 먼저 값을 볼 수도 있습니다. Thread 간 publication은 이런 구현 관찰에 기대지 않고 C++ release/acquire로 happens-before를 만들어야 합니다.

## 1.3 Cache는 byte가 아니라 주로 cache line을 옮긴다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Cache | 느리고 먼 memory의 data 일부를 CPU 가까이에 복사해 두는 작은 저장소 |
| Cache line | Cache가 data를 가져오고 내보내며 coherence를 관리하는 대표 전송 단위 |
| Cache hit | 요청한 data가 해당 cache level에 있는 경우 |
| Cache miss | 요청한 data가 해당 cache level에 없어 더 아래 level에 요청해야 하는 경우 |
| L1·L2 | 일반적으로 core에 가까운 private cache level. 정확한 구조는 CPU별로 다르다 |
| LLC | Last Level Cache. DRAM 전에 있는 마지막 cache level을 통칭하며 공유 방식은 CPU별로 다르다 |
| Working set | 일정 실행 구간에서 실제로 자주 접근하는 code와 data의 집합 |
| Spatial locality | 한 주소를 사용한 뒤 주변 주소도 곧 사용할 가능성이 높은 성질 |

### Cache가 필요한 이유

CPU 계산 장치는 매우 빠르지만 DRAM은 상대적으로 멀다. 자주 쓰는 데이터를 가까운 작은 저장소에 복사해 두는 것이 cache다.

```text
Core
 ├─ L1: 가장 작고 빠름
 ├─ L2: 더 크고 조금 느림
 └─ LLC: 여러 core가 공유하는 더 큰 cache
       └─ Memory controller → DRAM
```

정확한 크기와 latency, L2/LLC 공유 구조는 CPU 제품마다 다르다. 중요한 것은 가까운 cache일수록 작고 빠르며, 멀어질수록 기다림이 커진다는 방향이다.

CPU가 `int` 하나를 읽어도 보통 그 4 byte만 가져오지 않는다. 주변 bytes를 묶은 cache line 단위로 가져온다. 많은 x86 server에서 line은 64 byte지만 C++ 표준이나 모든 CPU의 보장은 아니다.

### 왜 연속 배열이 유리한가?

```cpp
std::vector<Order> orders;   // 원소가 연속됨
```

한 `Order`를 읽을 때 같은 line에 다음 `Order` 일부도 함께 들어올 수 있다. 반면 linked list는 다음 node 주소가 전혀 다른 곳일 수 있다.

```text
연속 배열: [Order0][Order1][Order2][Order3]
linked list: [Node0] ─────→ [Node1] ─────→ [Node2]
```

연속성은 cache line 활용뿐 아니라 다음 주소를 미리 알 수 있게 해 memory-level parallelism과 prefetch에도 도움을 준다.

### Inclusive와 non-inclusive를 너무 일찍 외우지 않는다

- **Inclusive:** 가까운 cache의 line이 아래 level에도 포함되는 관계를 보장한다.
- **Exclusive:** level 사이 중복을 줄이려는 관계다.
- **Non-inclusive/non-exclusive:** 어느 포함 관계도 보장하지 않는다.

이는 제품별 정책이다. “LLC에 없으니 어느 L1에도 없다”는 추론은 inclusive라고 확인한 CPU에서만 가능하다.

### 한 문장 기억법

> Cache 최적화의 기본 단위는 변수 하나가 아니라, 함께 움직이고 ownership을 공유하는 cache line이다.

### 30초 면접 답변

> CPU cache는 보통 line 단위로 데이터를 이동합니다. L1, L2, LLC로 갈수록 용량은 커지고 latency도 커지며 실제 hierarchy와 inclusion 정책은 제품별입니다. 연속 배치는 한 line에 유용한 데이터를 많이 담고 pointer chasing을 줄이지만, 같은 line을 여러 코어가 쓰면 false sharing이 생길 수 있습니다.

## 1.4 Cache miss가 나면 CPU는 어디에서 데이터를 찾는가?

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Virtual address | Process가 사용하는 논리 주소 |
| Physical address | 실제 memory와 cache coherence에서 식별되는 물리 주소 |
| TLB | Virtual address에서 physical address로의 최근 변환을 저장하는 cache |
| Cache set·tag | 주소로 후보 set을 고르고 tag를 비교해 원하는 line인지 확인하는 구조 |
| Miss-tracking entry | 진행 중인 cache miss와 기다리는 load를 기록하는 CPU 내부 자원. MSHR 계열 구조가 대표적이다 |
| Memory-level parallelism | 여러 독립 memory 요청을 동시에 진행하는 능력 |
| Pointer chasing | 다음에 읽을 주소가 현재 pointer load 결과에 의존하는 접근 형태 |

Load 주소는 virtual address다. CPU는 먼저 page table translation 결과를 TLB에서 찾고, 동시에 가능한 범위에서 L1 cache의 set과 tag를 조회한다.

```text
virtual address
  ├─ TLB에서 virtual → physical translation 확인
  └─ L1D cache lookup
        miss → L2 → LLC/directory
                       ├─ 다른 core cache의 최신 사본
                       └─ memory controller → DRAM
```

Miss가 날 때마다 CPU 전체가 즉시 멈추는 것은 아니다. CPU는 miss를 추적하는 entry를 만들고 다른 독립 작업을 진행할 수 있다. 같은 line을 기다리는 여러 load는 한 요청에 합쳐질 수도 있다.

하지만 entry 수와 동시에 처리할 miss 수는 유한하다. 특히 다음 주소를 앞 load 결과로 알아내는 pointer chain은 병렬화하기 어렵다.

```cpp
node = node->next;       // next 주소를 알려면 현재 node load가 끝나야 함
node = node->next;
node = node->next;
```

### 한 문장 기억법

> Cache miss 하나의 비용보다 더 중요한 것은, 그 miss를 다른 독립 작업과 겹칠 수 있는가이다.

### 30초 면접 답변

> L1 miss가 나면 CPU는 유한한 miss-tracking entry에 요청을 기록하고 L2, LLC와 coherence directory를 거쳐 peer cache나 DRAM에서 line을 받습니다. 독립 miss는 겹칠 수 있지만 dependent pointer chasing은 다음 주소를 앞 load 뒤에야 알아 memory-level parallelism이 낮습니다.

## 1.5 Cache coherence와 false sharing은 ownership 문제다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Cache coherence | 여러 cache에 같은 line의 복사본이 있을 때 write 순서와 ownership을 맞추는 hardware 규칙과 동작 |
| Sharer | 같은 cache line의 읽기 가능한 복사본을 가진 core 또는 cache |
| Write ownership | 해당 line을 수정할 수 있는 권한 |
| Invalidation | 다른 cache의 복사본을 더 이상 사용할 수 없게 만드는 coherence 동작 |
| RFO | Read For Ownership. Line의 data와 write 권한을 요청하는 동작을 설명할 때 쓰는 용어 |
| Clean·dirty | DRAM의 값과 같은 line은 clean, memory보다 새 write를 가진 line은 dirty라고 부른다 |
| MESI·MOESI | Line의 coherence 상태를 설명하는 대표적인 state model |
| False sharing | 서로 다른 변수가 같은 line에 있어 독립된 write끼리 ownership 경쟁을 일으키는 현상 |
| Padding | 두 field 사이에 사용하지 않는 공간을 넣어 다른 cache line에 배치하려는 방법 |

두 코어가 같은 line을 읽기만 할 때는 각 cache에 clean copy를 가질 수 있다. 하지만 한 코어가 그 line에 쓰려면 다른 복사본이 더 이상 옛 값을 사용하지 못하게 해야 한다.

```text
처음:
Core A cache: line X = Shared
Core B cache: line X = Shared

Core B가 X에 store:
1. B가 write ownership 요청
2. A의 copy를 invalidate
3. acknowledgement를 기다림
4. B가 Modified 계열 ownership으로 write
```

MESI/MOESI는 이 흐름을 설명하는 대표 모델이다. 실제 CPU에는 더 많은 transient state와 directory message가 있을 수 있다.

### False sharing

다음 두 변수는 논리적으로 서로 다르다.

```cpp
struct Counters {
    std::atomic<std::uint64_t> producer;
    std::atomic<std::uint64_t> consumer;
};
```

하지만 같은 cache line에 놓이고 서로 다른 코어가 계속 쓴다면 line ownership 전체가 왕복한다.

```text
Core A writes producer → line ownership이 A로 이동
Core B writes consumer → line ownership이 B로 이동
Core A writes producer → 다시 A로 이동
```

이것이 false sharing이다. Atomic을 사용해도 coherence 단위는 더 작아지지 않으므로 false sharing은 남는다.

### 해결 방향

- 한 상태에는 한 writer만 두는 single-writer 설계
- producer와 consumer metadata를 다른 line에 배치
- 여러 번의 공유 write를 local에 모은 뒤 batch publish
- 전역 counter 대신 core별 counter를 두고 나중에 합산

Padding은 footprint와 TLB pressure를 늘릴 수 있으므로 무조건 적용하지 않고 실제 write sharing이 있는 곳에 쓴다.

### 한 문장 기억법

> Atomic은 경쟁을 없애지 않는다. 같은 line을 여러 코어가 쓰면 안전할 수는 있어도 비쌀 수 있다.

### 30초 면접 답변

> Coherence는 같은 memory location의 write order와 cache-line ownership을 조정합니다. 다른 sharer가 있는 line에 쓰려면 ownership을 얻고 복사본을 invalidate해야 합니다. 서로 다른 변수도 같은 line에서 여러 코어가 쓰면 false sharing으로 line이 ping-pong하며, atomic은 correctness를 주지만 이 traffic을 없애지는 않습니다.

## 1.6 Branch prediction은 분기 결과가 나오기 전에 실행 경로를 고른다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Branch | 조건이나 target에 따라 다음 instruction 주소를 바꾸는 명령 |
| Conditional branch | 조건이 참인지에 따라 두 경로 중 하나를 선택하는 branch |
| Branch target | Branch 뒤에 실행할 것으로 선택된 instruction 주소 |
| Branch predictor | 과거 실행 정보 등으로 branch 방향과 target을 예측하는 CPU 구조 |
| Speculative execution | 예측이 맞다고 가정하고 뒤 instruction을 미리 실행하는 것 |
| Misprediction | 예측한 경로가 실제 결과와 다른 상황 |
| Younger work | Program 순서상 해당 branch보다 뒤에 있는 instruction 작업 |
| Branchless | Branch 대신 `cmov`, mask나 산술 operation을 사용하도록 만든 code 형태 |
| PMU | Performance Monitoring Unit. Branch miss와 cache miss 같은 hardware event를 세는 CPU 기능 |

`if` 결과를 계산할 때까지 fetch를 멈추면 pipeline이 자주 빈다. CPU는 과거 기록과 branch 주소를 이용해 다음 경로를 예측하고 미리 실행한다.

```text
if (condition) hot_path();
else           rare_path();

예측 성공 → 미리 한 일이 그대로 사용됨
예측 실패 → 잘못된 경로의 younger work를 버리고 다시 시작
```

예측 실패 비용은 고정 숫자가 아니다. Condition이 빨리 준비됐는지, front-end가 얼마나 깊이 진행했는지에 따라 달라진다.

### Branchless가 항상 답은 아니다

Branch를 `cmov`나 산술식으로 바꾸면 misprediction을 피할 수 있다. 그러나 양쪽 값을 모두 계산하거나 긴 dependency를 만들 수 있다. 예측이 매우 잘 되는 branch는 그대로 두는 편이 더 빠를 수 있다.

### 한 문장 기억법

> Branch prediction은 맞으면 공짜에 가깝지만, 틀리면 미리 한 일을 버린다. Branchless는 그 대가를 항상 계산하는 방식일 수 있다.

### 30초 면접 답변

> CPU는 direction과 target을 예측해 다음 instruction을 speculative하게 실행합니다. Mispredict면 younger work를 폐기하고 올바른 target에서 front-end를 다시 채웁니다. Branchless 변환도 instruction 수와 dependency를 늘릴 수 있으므로 production 분포와 PMU로 비교해야 합니다.

## 1.7 TLB miss는 page fault로 이어질 수 있지만 같은 사건은 아니다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Page | Virtual memory와 physical memory를 관리하는 고정 크기 단위 |
| Page table | Virtual page와 physical page의 mapping을 기록하는 자료구조 |
| PTE | Page Table Entry. 한 page mapping의 주소와 권한 등을 담은 항목 |
| Page walk | TLB에 변환이 없을 때 page table의 여러 level을 따라 mapping을 찾는 과정 |
| Page fault | Mapping 부재, 권한, COW 등을 kernel이 처리하도록 발생하는 CPU exception |
| Minor fault | Storage I/O 없이 mapping 설치, zero page, COW 등으로 해결할 수 있는 fault |
| Major fault | File/storage에서 data를 읽어야 할 수 있는 fault |
| COW | Copy-on-Write. 처음에는 page를 공유하고 write 시 복사하는 방식 |
| Huge page | 일반 page보다 큰 page. TLB 한 entry가 더 넓은 주소 범위를 덮게 한다 |
| THP | Transparent Huge Pages. Linux가 일반 mapping을 huge page로 관리하려는 기능 |
| Pre-touch | 실제 page를 미리 할당하고 page table을 준비하도록 시작 단계에서 page마다 접근하는 것 |

Program은 virtual address를 사용한다. CPU는 page table을 이용해 physical address로 바꾸며, 최근 변환을 TLB에 보관한다.

```text
virtual address
  → TLB hit: 변환을 바로 얻음
  → TLB miss: page table을 걸어 변환을 찾음
       → valid mapping 발견: hardware page walk로 끝날 수 있음
       → mapping 없음/COW/권한 문제: page fault로 kernel 진입
```

따라서 TLB miss는 cache miss와 비슷한 “변환 cache miss”이고, page fault는 kernel의 처리가 필요한 예외다.

### Huge page는 무엇을 줄이는가?

4 KiB page 대신 2 MiB page를 사용하면 같은 memory 범위를 더 적은 TLB entry로 덮을 수 있다. Page walk 단계도 줄어들 수 있다. 하지만 huge page 자체가 data를 연속적으로 읽게 만들거나 data-cache miss를 없애는 것은 아니다.

Huge page에는 allocation 실패, fragmentation, NUMA placement, THP promotion·split에 따른 jitter 같은 대가가 있다.

### `mlockall`도 만능이 아니다

Memory를 swap 대상에서 제외해도 이후 새 mapping, stack 성장, COW와 lazy allocation에서 fault가 생길 수 있다. 저지연 program은 시작할 때 memory와 stack을 확보하고 실제 write로 page를 pre-touch하며 실행 중 fault counter를 확인한다.

### 한 문장 기억법

> TLB miss는 주소 번역을 다시 찾는 일이고, page fault는 kernel이 mapping 문제를 처리하는 일이다.

### 30초 면접 답변

> TLB miss는 translation cache에 항목이 없어 page table walk가 필요한 상황이며 page fault와 같지 않습니다. Valid PTE를 찾으면 hardware walk로 끝날 수 있지만 not-present나 COW 처리는 fault를 일으킵니다. Huge page는 TLB coverage를 늘리지만 fragmentation과 NUMA, promotion jitter 때문에 workload별로 검증해야 합니다.

## 1.8 NUMA와 DMA는 CPU 밖의 길까지 보게 만든다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Socket | 하나의 CPU package와 그 주변 memory·I/O 연결을 구분하는 단위 |
| NUMA node | 가까운 CPU와 memory를 묶은 topology 단위 |
| Local·remote memory | 현재 core와 같은 node의 memory가 local, 다른 node의 memory가 remote다 |
| Interconnect·fabric | Core, cache, socket, memory controller와 I/O를 연결하는 내부 통신 경로 |
| NUMA placement | Thread와 memory page가 어느 node에 배치됐는지 나타내는 상태 |
| DMA | Direct Memory Access. Device가 CPU의 byte별 copy 없이 host memory를 읽거나 쓰는 방식 |
| NIC | Network Interface Controller. Network frame 송수신과 queue, DMA를 처리하는 장치 |
| Descriptor | NIC에 buffer 주소, 길이와 상태를 알려주는 metadata |
| Doorbell | 새 descriptor가 준비됐음을 CPU가 device에 알리는 MMIO write |
| Completion | Device가 descriptor 처리를 마쳤거나 ownership을 돌려줬음을 알리는 상태·event |
| Polling | Interrupt를 기다리지 않고 program이 상태를 반복 확인하는 방식 |

한 socket server처럼 보여도 memory까지의 거리가 모두 같지 않을 수 있다.

```text
Socket 0 ─ local memory 0
   │
   └─ inter-socket fabric ─ Socket 1 ─ local memory 1
```

Socket 0의 core가 node 1 memory를 읽으면 interconnect를 건너야 한다. 그래서 thread만 CPU 0에 pinning하고 memory는 node 1에 두면 locality 문제가 남는다.

Linux의 anonymous page는 흔히 처음 실제 touch한 CPU의 node에 놓인다. 초기화 thread가 다른 socket에서 pool 전체를 pre-touch하면 worker를 올바른 core에 붙여도 pool은 원격일 수 있다.

### NIC DMA도 같은 지도를 사용한다

NIC는 CPU가 byte마다 복사해 주기를 기다리지 않고 DMA로 host memory의 RX buffer에 packet을 쓴다.

```text
NIC receives packet
  → RX descriptor가 가리키는 buffer로 DMA
  → completion/ownership 갱신
  → interrupt 또는 polling으로 CPU에 알림
```

Cache-coherent DMA 플랫폼도 ordering이 저절로 완성되는 것은 아니다. Driver나 framework가 정한 순서대로 descriptor, barrier, doorbell, completion을 다뤄야 한다. Doorbell write가 반환됐다고 packet이 wire로 나간 것도 아니다.

### 함께 맞춰야 할 것

- NIC PCIe 위치
- RX/TX queue와 interrupt CPU
- polling/application thread CPU
- descriptor와 packet buffer가 실제 할당된 NUMA node
- strategy와 risk가 사용하는 hot state의 node

### 한 문장 기억법

> CPU affinity만 맞추지 말고 CPU, memory, NIC queue가 지나는 전체 topology를 맞춘다.

### 30초 면접 답변

> NUMA에서는 local memory controller보다 remote node 접근이 inter-socket fabric을 거쳐 보통 더 느리고 queueing 영향도 받습니다. Thread pinning만으로 page 위치는 바뀌지 않습니다. NIC path에서는 PCIe 위치, queue CPU, DMA buffer와 application memory의 실제 NUMA placement를 함께 검증해야 합니다.

## 1.9 Compiler, C++와 CPU는 서로 다른 규칙 층이다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Compiler | C++ source를 target CPU가 실행할 machine code로 변환하는 program |
| ISA | Instruction Set Architecture. Software에 보이는 instruction, register와 memory-order 규칙 |
| C++ memory model | 여러 thread의 access와 atomic ordering이 언제 정의되는지 정하는 language 규칙 |
| Undefined behavior | C++ 표준이 program의 결과를 요구하지 않는 상황. Compiler는 이런 상황이 없는 code를 전제로 최적화할 수 있다 |
| `volatile` | 해당 object access를 일반 access처럼 제거·합치지 못하게 하는 C++ type qualifier. Thread synchronization은 아니다 |
| Compiler barrier | Compiler의 memory operation 재배치를 제한하는 수단 |
| CPU fence | ISA level에서 특정 load/store ordering을 제한하는 instruction 또는 효과 |
| MMIO | Memory-Mapped I/O. Memory 주소처럼 보이는 device register 접근 방식 |
| ABI | Binary 사이의 호출, register 사용, stack, object layout과 symbol 규칙 |
| Alignment | Object 시작 주소가 type이나 hardware가 요구하는 byte 경계에 맞는 성질 |
| Endian | 여러 byte로 된 값을 memory나 wire에 배치하는 byte 순서 |

다음 코드를 보고 “내 CPU에서는 64-bit load가 한 번에 되니까 안전하다”고 말하면 부족하다.

```cpp
bool ready;  // plain non-atomic
int payload;
```

서로 다른 thread가 동기화 없이 `ready`를 읽고 쓰면 C++ data race다. CPU instruction이 값을 찢지 않아도 C++ program은 undefined behavior가 될 수 있다. Compiler는 data race가 없는 정의된 program을 전제로 최적화하기 때문이다.

### 세 규칙 층

| 층 | 묻는 질문 |
| --- | --- |
| C++ memory model | 이 program에 data race가 없고 happens-before가 있는가? |
| ISA memory ordering | CPU가 load/store를 어떤 순서로 관찰하게 하는가? |
| Cache coherence | 같은 line의 복사본과 write ownership을 어떻게 맞추는가? |

한 층이 다른 층을 자동으로 대신하지 않는다.

### `volatile`과 barrier

- C++ `volatile`은 일반 thread synchronization 수단이 아니다.
- Compiler barrier는 compiler의 재배치를 제한할 수 있지만 CPU ordering까지 보장하지 않을 수 있다.
- CPU fence는 hardware ordering을 제한하지만 C++ data race를 합법화하지 않는다.
- MMIO는 OS·driver가 제공하는 accessor와 device barrier 계약을 따른다.

### Undefined behavior가 위험한 이유

Signed overflow, out-of-bounds, 잘못된 object lifetime, strict aliasing 위반, data race가 대표적이다. UB는 단지 해당 줄에서 이상한 값이 나온다는 뜻이 아니다. Compiler가 “정의된 program이라면 이 상황은 없다”고 가정해 주변 code 전체를 바꿀 수 있다.

Wire bytes를 packed struct pointer로 바로 cast하는 코드는 alignment, lifetime, aliasing과 endian 문제를 동시에 만들 수 있다. 경계에서 안전하게 decode하는 것이 우선이다.

### ABI까지 보는 이유

ABI는 argument를 register로 전달할지 stack으로 전달할지, stack alignment와 saved register, object layout을 정한다. 작은 struct는 값으로 넘겨도 register에 들어갈 수 있고, reference는 오히려 memory load와 alias 가능성을 만들 수 있다. “값 전달은 느리다” 같은 규칙 대신 실제 compiler output을 확인한다.

### 한 문장 기억법

> C++가 허용한 program을 compiler가 ISA 명령으로 만들고, CPU가 coherence와 ordering 규칙으로 실행한다. 아래 층의 우연한 동작이 위 층의 UB를 고치지 않는다.

### 30초 면접 답변

> C++ memory model, ISA ordering과 cache coherence는 서로 다른 층입니다. Coherent x86에서도 plain non-atomic data race는 C++ UB입니다. `volatile`은 thread synchronization을 제공하지 않으며, atomic이나 mutex로 language-level 관계를 만든 뒤 target ISA의 codegen과 cache traffic을 별도로 분석해야 합니다.

# 2. Modern C++와 동시성: 객체와 값은 언제까지 살아 있는가?

## 2.1 RAII는 cleanup 문법이 아니라 ownership 설계다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Resource | Memory, file descriptor, socket, lock처럼 획득 후 반드시 반납·해제해야 하는 대상 |
| RAII | Resource Acquisition Is Initialization. Resource ownership을 object lifetime에 묶는 C++ 설계 기법 |
| Owner | Resource를 반납할 책임과 사용 권한을 가진 object |
| Constructor | Object lifetime을 시작하며 초기 state를 만드는 함수 |
| Destructor | Object lifetime 종료 시 cleanup을 수행하는 함수 |
| Class invariant | Constructor 성공 뒤 public operation 전후에 유지돼야 하는 class 상태 조건 |
| Scope | Local variable 이름과 lifetime이 적용되는 source 범위 |
| Stack unwinding | Exception 전파 중 이미 생성된 automatic object를 역순으로 파괴하는 과정 |

RAII object는 생성 성공 시 resource ownership을 얻고, lifetime 종료 시 destructor에서 반납한다.

```text
객체 생성 성공
  → 자원 소유권과 class invariant 성립
객체 lifetime 종료
  → destructor가 자원 반납
```

자원은 heap memory만이 아니다.

- File descriptor와 socket
- Mutex lock
- Memory mapping
- Thread
- NIC queue handle
- 임시로 변경한 설정을 원래 값으로 되돌리는 작업

### RAII가 stack 객체라는 뜻은 아니다

RAII 객체는 stack, heap, 다른 객체의 member 어디에 있어도 된다. 핵심은 storage 위치가 아니라 **객체 lifetime과 ownership을 묶는 것**이다.

```cpp
class Fd {
public:
    explicit Fd(int fd) noexcept : fd_(fd) {}
    ~Fd() { if (fd_ >= 0) ::close(fd_); }

    Fd(const Fd&) = delete;
    Fd& operator=(const Fd&) = delete;

private:
    int fd_;
};
```

정상 scope 종료나 exception unwinding에서는 destructor가 호출된다. 하지만 process crash, `_Exit`, 강제 종료처럼 unwinding이 없는 경로까지 RAII가 복구해 주지는 않는다. 주문 상태의 durability에는 event log와 recovery protocol이 별도로 필요하다.

### 생성 중 실패하면 어떻게 되는가?

일반적인 생성자가 중간에 실패하면 완성된 가장 바깥 객체의 destructor는 호출되지 않는다. 대신 이미 생성이 끝난 base와 member가 역순으로 파괴된다.

```text
Socket 생성 성공
Buffer 생성 성공
Parser 생성 실패
  → Buffer 파괴
  → Socket 파괴
  → 가장 바깥 객체는 완성되지 않았으므로 그 destructor는 호출되지 않음
```

그래서 raw handle을 얻은 직후 RAII member나 local owner에 담아야 한다. “생성자 끝에서 한꺼번에 정리 객체에 넣겠다”는 사이의 예외가 누수를 만들 수 있다.

### 한 문장 기억법

> RAII는 자원을 객체가 살아 있는 동안만 소유하게 만들어, 여러 종료 경로의 cleanup을 한곳에 모은다.

### 30초 면접 답변

> RAII는 resource ownership을 object lifetime에 결합하는 기법입니다. 생성 성공 시 invariant와 ownership이 성립하고 정상 scope 종료나 stack unwinding에서 destructor가 자원을 반납합니다. Memory뿐 아니라 lock과 fd에도 적용되며, crash처럼 unwinding이 없는 경로의 durability는 별도 recovery가 필요합니다.

## 2.2 Exception guarantee는 실패 뒤 상태를 약속한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Exception | 정상 반환 대신 호출 경로 위쪽으로 오류를 전달하는 C++ mechanism |
| Exception guarantee | Operation 실패 뒤 object state와 resource에 대해 제공하는 약속 |
| Commit | 준비한 변경을 외부에서 관찰 가능한 state로 확정하는 단계 |
| Rollback | 실패 시 변경 전 state로 되돌리는 처리 |
| `noexcept` | 함수 밖으로 exception이 전파되지 않는다는 C++ 선언. 위반하면 `std::terminate`가 호출된다 |
| `std::terminate` | Exception을 더 처리할 수 없는 상황 등에서 program을 종료하는 C++ 함수 |
| Side effect | 함수 밖에서도 관찰되는 변화. Memory state, file write, network send 등이 포함된다 |

Exception을 쓰는가보다 중요한 질문은 **연산이 실패한 뒤 객체가 어떤 상태인가**이다. Error code나 `expected`를 사용해도 같은 질문이 남는다.

| 보장 | 실패 뒤 상태 |
| --- | --- |
| Basic | 누수 없이 invariant는 유지되지만 값 일부가 바뀔 수 있다 |
| Strong | 호출 전과 같은 관찰 가능한 상태다 |
| No-throw | 실패를 보고하는 exception을 외부로 던지지 않는다 |

Strong guarantee는 흔히 임시 상태에서 실패 가능한 일을 끝낸 뒤, 실패하지 않는 commit으로 공개한다.

```cpp
Book next = current;       // 여기서 실패해도 current는 그대로
next.apply(update);        // 여기서 실패해도 current는 그대로
current.swap(next);        // noexcept commit
```

하지만 전체 order book을 매 packet마다 복사하면 너무 비싸다. 실제 hot path에서는 다음 방법을 검토한다.

- 적용 전에 모든 조건을 검사하는 prevalidation
- 용량이 정해진 bounded storage
- 바뀐 부분만 되돌리는 undo log
- 별도 snapshot을 만든 뒤 pointer/version swap

또한 NIC에 주문을 이미 보냈다면 memory를 이전 값으로 되돌려도 외부 세계의 주문은 사라지지 않는다. 외부 side effect에는 idempotency, journal과 reconciliation이 필요하다.

### Destructor에서 exception을 피하는 이유

다른 exception을 처리하며 stack unwinding 중인데 destructor에서도 exception이 밖으로 나오면 `std::terminate`가 호출될 수 있다. 중요한 `flush` 실패를 caller가 알아야 한다면 명시적인 `close()`나 `commit()` API로 먼저 보고하고 destructor는 best-effort cleanup을 담당하게 한다.

### 한 문장 기억법

> Exception safety는 예외 문법의 문제가 아니라, 실패 뒤에도 어떤 불변식을 지킬지 정하는 문제다.

### 30초 면접 답변

> Basic guarantee는 실패 뒤에도 자원과 invariant를 보존하고, strong guarantee는 호출 전 상태로 보이며, no-throw guarantee는 실패를 보고하는 예외를 외부로 던지지 않는 계약입니다. Hot path에서는 copy-and-swap 대신 prevalidation이나 bounded storage로 같은 invariant를 더 싸게 구현할 수 있습니다.

## 2.3 Rule of Zero/Five와 smart pointer는 소유권을 코드에 보이게 한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Special member function | Constructor, destructor, copy·move constructor/assignment처럼 compiler가 자동 생성할 수 있는 함수 |
| Rule of Zero | Resource를 RAII member에 맡겨 special member를 직접 작성하지 않는 설계 원칙 |
| Rule of Five | 직접 ownership을 구현할 때 destructor와 copy·move 네 operation을 함께 검토하는 원칙 |
| Move | Source object의 resource ownership을 destination object로 이전하는 operation |
| Rvalue | Move overload 선택에 사용할 수 있는 value category |
| Control block | `shared_ptr`가 reference count, deleter 등을 저장하는 별도 관리 object |
| Reference count | 현재 shared owner 수를 기록하는 count |
| Deleter | Pointer가 소유한 resource를 실제로 반납하는 함수·object |

### Rule of Zero

자원을 이미 잘 관리하는 member를 사용하면 destructor와 copy/move를 직접 작성하지 않아도 된다.

```cpp
struct Session {
    std::string name;
    std::vector<std::byte> buffer;
    std::unique_ptr<Socket> socket;
};
```

Compiler가 생성한 special member가 각 member의 올바른 동작을 조합한다. 직접 raw resource ownership을 구현한다면 destructor, copy constructor/assignment, move constructor/assignment를 함께 검토하는 Rule of Five가 필요하다.

### `std::move`가 data를 직접 옮기는 것은 아니다

`std::move`는 대체로 rvalue로 cast해 move overload가 선택될 기회를 준다. 실제 ownership 이동은 move constructor나 assignment가 한다.

`noexcept` move는 container 재할당에서 중요하다. Move 중 실패 가능성이 있고 copy가 가능하면 `vector`가 strong guarantee를 위해 copy를 고를 수 있다. 하지만 거짓으로 `noexcept`를 붙여 exception이 나오면 program은 종료된다.

### `unique_ptr`와 `shared_ptr`

```text
unique_ptr: owner 한 명
shared_ptr: control block의 strong count가 owner 수를 기록
```

`unique_ptr`는 소유권이 명확하고 move로 handoff한다. `shared_ptr` 복사와 파괴는 보통 atomic reference count를 갱신한다.

여기서 자주 하는 오해가 있다.

- 서로 다른 `shared_ptr` 복사본이 같은 control block을 갱신하는 것은 안전하다.
- 그렇다고 가리키는 `OrderBook` 객체 자체가 thread-safe해지는 것은 아니다.
- 같은 `shared_ptr` 변수 하나를 여러 thread가 동시에 바꾸려면 별도 동기화나 `atomic<shared_ptr<T>>`가 필요하다.
- 마지막 owner를 놓는 thread에서 무거운 destructor와 deallocation이 실행될 수 있다.

### 저지연에서는 무엇을 선택하는가?

무조건 smart pointer를 없애는 것이 답이 아니다. Hot path의 ownership을 single writer와 queue handoff로 명확히 만들고, 불가피한 공유 snapshot에는 RCU, epoch, double buffering과 `shared_ptr`를 비교한다.

### 한 문장 기억법

> Pointer 종류는 주소 저장 방식이 아니라 누가 언제까지 객체를 살려 둘지 표현한다.

### 30초 면접 답변

> Rule of Zero는 resource 관리를 RAII member에 맡겨 special member를 직접 쓰지 않는 원칙입니다. `unique_ptr`는 단독 ownership을, `shared_ptr`는 atomic reference count 기반 공유 ownership을 표현합니다. `shared_ptr`가 pointee를 thread-safe하게 만들지는 않으며 마지막 release 위치의 destructor 비용도 고려해야 합니다.

## 2.4 Storage와 object lifetime은 같은 것이 아니다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Storage | Object representation을 놓을 수 있는 연속된 byte 공간 |
| Storage duration | Storage가 존재하는 기간. Automatic, static, thread, dynamic duration 등이 있다 |
| Object lifetime | 특정 type의 object가 storage 안에 실제로 존재하며 type 규칙으로 접근할 수 있는 기간 |
| Placement construction | 이미 확보한 storage 주소에 object를 생성하는 방식 |
| `construct_at`·`destroy_at` | 지정 주소에서 object lifetime을 시작·종료하도록 돕는 C++ 함수 |
| Pointer provenance | Pointer가 어떤 storage와 object에서 유래했고 어떤 접근이 허용되는지에 관한 규칙 |
| Strict aliasing | 서로 호환되지 않는 type의 pointer로 같은 object를 임의 접근하지 못하게 하는 규칙 |
| Out-of-bounds | Object나 array가 차지하는 허용 범위 밖의 주소에 접근하는 것 |

```text
storage = 객체를 놓을 수 있는 byte 공간
lifetime = 그 공간에 특정 type의 객체가 실제로 존재하는 기간
```

Allocator가 memory를 확보한 것은 storage를 얻은 일이다. 그곳에 `Order`를 생성해야 `Order` lifetime이 시작된다. `Order`를 파괴해도 pool의 storage는 남아 다음 객체에 재사용할 수 있다.

```cpp
alignas(Order) std::byte storage[sizeof(Order)];

Order* order = std::construct_at(
    reinterpret_cast<Order*>(storage), args...);

use(*order);
std::destroy_at(order);
```

이 패턴에서는 다음 질문을 모두 답해야 한다.

- 주소가 `Order` alignment를 만족하는가?
- 정확히 한 번 construct하는가?
- Consumer가 다 읽은 뒤에만 destroy하는가?
- 파괴한 객체의 pointer를 다시 사용하지 않는가?
- 같은 slot에 새 객체를 만들었을 때 이전 reference가 유효한가?

`std::launder`는 같은 storage를 재사용한 뒤 특정 pointer가 새 object를 가리키도록 다루는 제한된 상황의 함수다. Out-of-bounds나 strict aliasing 위반을 합법화하지 않는다.

### 한 문장 기억법

> Pool slot의 byte 공간이 살아 있는 것과 그 안의 C++ 객체가 살아 있는 것은 별개다.

### 30초 면접 답변

> Storage duration은 byte 공간이 존재하는 기간이고 object lifetime은 그 공간에 특정 type의 object가 존재하는 기간입니다. Placement construction을 쓰는 pool에서는 alignment, construction, destruction과 slot 재사용 순서를 모두 지켜야 하며 lifetime 밖의 member access는 UB가 될 수 있습니다.

## 2.5 범용 allocator의 평균이 빨라도 tail은 흔들릴 수 있다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Allocator | Dynamic storage를 확보하고 반납하는 구성 요소 |
| Heap | Dynamic allocation에 사용하는 process의 memory 영역과 관리 구조를 통칭하는 표현 |
| Arena | 여러 allocation을 한 영역에서 관리하고 묶어서 반납할 수 있게 한 allocator 구조 |
| Object pool | 정해진 크기·type의 object slot을 미리 준비해 재사용하는 구조 |
| Thread cache | Allocator가 작은 block을 thread별로 보관해 global contention을 줄이는 구조 |
| Refill | Thread cache나 pool에 빈 block이 없어 상위 allocator에서 새 block 묶음을 가져오는 과정 |
| Preallocation | 실행 전 필요한 storage를 미리 확보하는 것 |
| Exhaustion | Pool이나 bounded container의 남은 slot이 없는 상태 |
| Fallback | 기본 경로가 실패했을 때 사용하는 대체 경로. Hot path의 heap fallback은 tail을 흔들 수 있다 |
| `std::pmr` | Memory resource를 container에 주입할 수 있는 C++ polymorphic allocator interface |

`new` 한 번이 항상 system call을 하는 것은 아니다. 일반 allocator는 thread cache와 arena를 사용해 흔한 요청을 빠르게 처리한다. 문제는 드물게 다른 경로가 나타난다는 점이다.

```text
빠른 경우: thread-local free block 반환
느린 경우: refill → shared arena lock → 새 page → page fault/zeroing
```

이런 드문 경로가 p99.9 outlier가 될 수 있다. 또한 객체가 흩어지면 cache와 TLB locality도 나빠진다.

### 대안

- 시작할 때 최대 용량을 정해 사전 할당
- Fixed-size object pool
- 한 batch가 끝날 때 한꺼번에 버리는 monotonic arena
- `std::pmr`로 allocation 정책 주입
- 용량이 늘지 않는 bounded container

### 사전 할당만으로 끝나지 않는다

주소 공간만 예약되고 physical page는 첫 write 때 할당될 수 있다. 실제 page별 write로 pre-touch하고 NUMA node를 확인해야 한다.

Pool이 가득 찼을 때 몰래 heap으로 fallback하면 장애 순간 latency 목표가 무너진다. Reject, backpressure, fail-fast 중 하나를 명시하고 telemetry를 남긴다.

### 한 문장 기억법

> 저지연 allocation의 핵심은 평균 속도보다 최대 용량과 실패 경로를 미리 정하는 것이다.

### 30초 면접 답변

> 범용 allocator는 fast path가 빨라도 contention, arena refill, page fault와 coalescing으로 tail latency를 만들 수 있습니다. Hot path는 시작 시 preallocate와 pre-touch하고 fixed pool이나 bounded container를 사용하며, pool exhaustion 때 heap fallback이 아닌 명시적 reject나 fail-safe 정책을 둡니다.

## 2.6 C++ thread 통신은 happens-before 관계 그래프로 설명한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Memory location | C++ memory model이 read/write 충돌을 판단하는 저장 위치 단위 |
| Conflicting access | 같은 memory location에 대한 access 중 하나 이상이 write인 조합 |
| Plain non-atomic | `std::atomic`이 아닌 일반 object access를 설명할 때 쓰는 표현 |
| Data race | 서로 다른 thread의 conflicting access가 happens-before로 정렬되지 않고 적어도 하나가 non-atomic인 상황 |
| Sequenced-before | 한 thread 안에서 evaluation 사이에 성립하는 순서 관계 |
| Synchronizes-with | Release 값을 acquire가 읽는 경우처럼 thread 사이 synchronization을 만드는 관계 |
| Happens-before | Sequenced-before와 synchronizes-with 등을 연결해 access의 안전한 순서를 표현하는 관계 |
| Atomicity | Atomic operation이 중간 상태로 관찰되지 않도록 하는 성질 |

### Data race는 단순히 값이 오래된 문제가 아니다

두 thread가 같은 memory location에 충돌하는 접근을 하고, 하나 이상이 write이며, atomic이나 synchronization으로 정렬되지 않았다면 C++ data race가 될 수 있다. Data race는 undefined behavior다.

```cpp
int payload = 0;
bool ready = false;

// producer                  // consumer
payload = 42;               if (ready) use(payload);
ready = true;
```

Aligned access가 hardware에서 한 번에 수행돼도 이 코드는 안전한 thread 통신이 아니다.

### 관계를 그래프로 그린다

```text
producer 내부 순서                     consumer 내부 순서
payload = 42                           ready acquire load
     │ sequenced-before                     │ sequenced-before
     ▼                                      ▼
ready release store ─ synchronizes-with → use(payload)

전체 연결: payload write happens-before payload read
```

- **Sequenced-before:** 한 thread 안의 C++ 평가 순서
- **Synchronizes-with:** release 값을 acquire가 읽는 것 같은 thread 간 연결
- **Happens-before:** 이 관계를 연결한 안전한 관찰 순서

Acquire load가 이전 `false`를 읽은 실행에는 그 release와의 연결이 없다. “Acquire를 한 번 썼다”만으로 모든 producer store와 연결되는 것은 아니다.

### Atomicity와 ordering도 나눈다

Atomicity는 atomic object 연산이 중간 상태로 관찰되지 않게 한다. Ordering은 그 atomic 주변의 다른 연산까지 어떤 순서로 보이게 하는가를 다룬다. Atomic이라고 모든 주변 data가 자동으로 publish되는 것은 아니다.

### 한 문장 기억법

> 공유 payload를 안전하게 읽으려면 flag가 atomic인지만 보지 말고, payload write에서 read까지 happens-before 관계를 그린다.

### 30초 면접 답변

> Conflicting non-atomic access가 happens-before로 정렬되지 않으면 C++ data race이고 UB입니다. Release store가 쓴 값을 acquire load가 관찰하면 synchronizes-with가 생기고, thread 내부 sequenced-before와 합쳐 payload write가 payload read보다 happens-before가 됩니다.

## 2.7 Memory order는 atomic 주변 operation의 ordering 범위를 정한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Memory order | C++ atomic operation이 다른 operation과 어떤 ordering 관계를 만드는지 지정하는 값 |
| Modification order | 한 atomic object에 대한 모든 store와 RMW modification의 단일 순서 |
| Read-from | Atomic load가 어느 modification의 값을 읽었는지 나타내는 관계 |
| `relaxed` | Atomic object 자체의 원자성은 제공하지만 주변 data의 synchronization은 만들지 않는 order |
| `release` | 앞선 operation을 다른 thread에 공개하는 쪽에 사용하는 order |
| `acquire` | 대응 release 값을 읽어 앞선 effect를 관찰하는 쪽에 사용하는 order |
| `acq_rel` | RMW에서 acquire와 release 역할을 함께 수행하는 order |
| `seq_cst` | Acquire/release 계열 의미와 함께 seq_cst operation의 추가 total order 제약을 주는 order |
| Atomic fence | 특정 atomic access와 연결해 ordering을 만드는 operation |

### `relaxed`

Relaxed atomic도 그 atomic 접근의 원자성을 유지한다. Store와 RMW는 해당 atomic object의 modification order에 들어간다. 하지만 주변 plain data를 다른 thread에 publish하는 happens-before는 만들지 않는다.

적합한 예는 독립적인 통계 counter다.

```cpp
packet_count.fetch_add(1, std::memory_order_relaxed);
```

Counter 값 자체의 증가만 필요하고, 그 증가를 통해 다른 payload를 전달하지 않는 경우다.

### `release`와 `acquire`

Producer가 payload를 완성한 뒤 release store로 공개하고, consumer가 그 값을 acquire load로 읽으면 payload가 안전하게 전달된다.

```text
payload write → release publish → acquire observe → payload read
```

### `seq_cst`

Sequential consistency는 acquire/release 계열 의미에 더해 모든 seq_cst operation이 참여하는 하나의 total order 제약을 둔다. 추론은 쉬워질 수 있지만 다음 문제는 해결하지 않는다.

- Object lifetime 오류
- ABA
- 잘못된 queue full 조건
- Non-atomic data race
- 잘못된 business invariant

### Fence는 혼자서 의미가 완성되지 않는다

Atomic fence는 어떤 atomic read/write와 연결되는지까지 보아야 한다. Compiler barrier, CPU fence, C++ atomic fence는 이름이 비슷해도 다루는 층이 다르다.

X86에서 acquire load가 plain load instruction으로 보일 수 있는 것은 C++ 계약이 그 ISA에서 저렴하게 구현된다는 뜻이다. Source에서 atomic을 빼도 된다는 뜻은 아니다.

### 한 문장 기억법

> 가장 약한 memory order를 고르는 일은 instruction을 줄이는 게임이 아니라 필요한 happens-before만 정확히 남기는 증명이다.

### 30초 면접 답변

> Relaxed는 atomic object 자체의 atomicity와 modification order를 제공하지만 주변 payload publication은 만들지 않습니다. Release가 쓴 값을 acquire가 읽으면 happens-before가 연결됩니다. Seq_cst는 추가 total order로 추론을 단순화하지만 lifetime이나 algorithm invariant 오류를 해결하지는 않습니다.

## 2.8 CAS는 비교와 조건부 write를 하나의 atomic operation으로 수행한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| CAS | Compare-And-Swap 또는 compare-exchange. 현재 값 비교와 조건부 변경을 atomic하게 수행하는 operation |
| RMW | Read-Modify-Write. 한 atomic object를 읽고 계산한 뒤 수정하는 하나의 atomic operation |
| Expected | CAS가 현재 값과 같을 것으로 예상하는 값 |
| Desired | Expected가 맞을 때 저장하려는 새 값 |
| Spurious failure | 실제 값이 expected와 같아도 `compare_exchange_weak`가 실패할 수 있는 경우 |
| Retry loop | 실패한 CAS가 새 관찰 값을 바탕으로 다시 시도하는 반복 구조 |
| Retry storm | 높은 contention에서 많은 thread가 CAS 실패와 재시도를 반복하는 상황 |

여러 thread가 `value`를 읽고 조건이 맞으면 바꾸려 한다고 하자.

```text
일반 코드:
1. value 읽기
2. 조건 확인
3. value 쓰기

문제: 1과 3 사이에 다른 thread가 바꿀 수 있음
```

Compare-and-exchange는 “현재 값이 내가 예상한 값이면 새 값으로 교환”을 atomic RMW로 수행한다.

```cpp
auto expected = old_value;
while (!value.compare_exchange_weak(
           expected,
           new_value,
           std::memory_order_acq_rel,
           std::memory_order_acquire)) {
    // 실패하면 expected에는 실제 관찰 값이 들어간다.
    new_value = make_next(expected);
}
```

### Weak와 strong

- `weak`는 값이 같아도 spurious failure가 가능해 loop에 자연스럽다.
- `strong`은 spurious failure가 없지만 실제 값 불일치에는 똑같이 실패한다.

### 성공·실패 memory order가 다른 이유

성공은 read와 write를 모두 하는 RMW다. 실패는 새 값을 쓰지 않고 현재 값을 읽기만 한다. 따라서 failure order에는 release 의미를 줄 수 없다.

### CAS loop의 성능

CAS가 lock-free여도 여러 core가 같은 line을 갱신하면 다음 일이 반복될 수 있다.

```text
line ownership 경쟁 → 한 thread 성공 → 나머지 실패
→ 새 값 읽기 → 다시 계산 → 다시 CAS
```

이를 retry storm이라고 볼 수 있다. 먼저 state를 shard하거나 single writer로 만들어 경쟁 자체를 줄일 수 있는지 본다.

### 한 문장 기억법

> CAS는 읽기와 조건부 쓰기를 원자적으로 묶지만, 여러 thread가 몰리면 실패와 재시도가 새로운 queue가 된다.

### 30초 면접 답변

> Compare-exchange는 현재 값이 expected와 같을 때 원하는 값으로 바꾸는 atomic RMW입니다. Weak는 spurious failure가 가능해 loop에 적합하고, 실패 경로는 write하지 않으므로 failure ordering이 success보다 강할 수 없습니다. Correctness와 별개로 contention 시 retry와 cache-line bouncing을 측정해야 합니다.

## 2.9 Lock-free라는 이름은 latency 상한을 뜻하지 않는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Progress guarantee | Concurrent algorithm에서 어떤 operation이 어느 조건에 완료되는지에 대한 보장 |
| Obstruction-free | 다른 thread가 방해하지 않고 충분히 실행되면 현재 operation이 완료되는 성질 |
| Lock-free | 전체 실행이 계속되면 어떤 operation은 계속 완료되지만 특정 thread 완료는 보장하지 않는 성질 |
| Wait-free | 각 operation이 다른 thread 동작과 무관하게 유한한 step 상한 안에 완료되는 성질 |
| Starvation | System 전체는 진행하지만 특정 thread가 계속 완료하지 못하는 상황 |
| Preemption | Scheduler가 현재 thread 실행을 멈추고 다른 task에 CPU를 주는 동작 |
| Wall-clock latency | Algorithm step 수가 아니라 실제 clock으로 측정한 경과 시간 |
| `is_lock_free()` | 특정 atomic object operation이 내부 lock 없이 구현되는지 조회하는 C++ interface |

Progress guarantee는 누가 얼마나 진행할 수 있는지를 말한다.

| 종류 | 쉬운 뜻 |
| --- | --- |
| Obstruction-free | 혼자 충분히 실행하면 끝난다 |
| Lock-free | 전체적으로 어떤 operation은 계속 끝나지만 특정 thread는 굶을 수 있다 |
| Wait-free | 각 operation이 유한한 step 상한 안에 끝난다 |

`atomic<T>::is_lock_free()`는 해당 atomic operation이 내부 lock 없이 구현되는지를 말할 뿐이다. 그 atomic으로 만든 queue 전체가 lock-free라는 증명은 아니다.

또한 algorithm이 lock-free여도 다음 사건은 wall-clock latency를 크게 만들 수 있다.

- Thread preemption
- Page fault
- Cache miss
- CAS contention
- Memory reclamation 지연

Wait-free step bound도 CPU 시간을 보장받는다는 뜻은 아니다. Scheduler가 thread를 실행하지 않으면 현실 시간 deadline은 지킬 수 없다.

### 한 문장 기억법

> Progress guarantee는 algorithm의 step에 대한 약속이지, 운영체제까지 포함한 microsecond deadline 약속이 아니다.

### 30초 면접 답변

> Lock-free는 system-wide progress를 보장하지만 특정 thread starvation과 고정 step bound는 허용합니다. Wait-free는 각 operation의 bounded steps를 요구합니다. 그러나 preemption과 page fault를 포함한 wall-clock bound는 별도 문제이며, `is_lock_free()`도 자료구조 전체의 progress를 증명하지 않습니다.

## 2.10 ABA는 bit 값이 같아도 중간 state와 object가 바뀐 문제다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| ABA | 관찰 값이 A에서 다른 상태를 거쳐 다시 A가 되어 CAS가 중간 변화를 알아채지 못하는 문제 |
| Version tag | Pointer나 value에 변경 version을 함께 저장해 같은 bit 값의 세대를 구분하는 값 |
| Wrap | 증가하는 counter가 type 최댓값을 지나 다시 작은 값으로 돌아가는 현상 |
| Unlink | Node를 자료구조에서 논리적으로 제거하는 operation |
| Reclamation | 제거된 node의 memory를 실제 해제하거나 재사용 가능하게 만드는 과정 |
| Use-after-free | 이미 lifetime이 끝나거나 해제된 object를 pointer로 접근하는 오류 |
| Hazard pointer | Reader가 곧 접근할 pointer를 공개해 reclaimer가 해당 node를 해제하지 못하게 하는 기법 |
| Epoch·EBR | Reader의 실행 세대를 추적해 이전 세대 reader가 끝난 뒤 일괄 reclaim하는 기법 |
| RCU grace period | 제거 전 read-side 작업이 모두 끝났다고 판단할 수 있을 때까지 기다리는 기간 |

Stack top이 주소 A라고 하자.

```text
Thread 1: top=A를 읽고 잠시 멈춤
Thread 2: A를 pop
Thread 2: B를 pop
Thread 2: A 주소를 다시 push 또는 재사용
Thread 1: top이 아직 A라고 보고 CAS 성공
```

Thread 1에게 값은 A→A로 같아 보인다. 하지만 중간에 구조와 object lifetime이 바뀌었다. 이것이 ABA다.

Pointer에 version tag를 함께 저장하면 `(A, version 1)`과 `(A, version 3)`을 구분할 수 있다. 하지만 tag wrap도 고려해야 한다.

### Unlink와 reclaim을 분리한다

Node를 자료구조에서 제거해도 다른 reader가 raw pointer를 들고 있을 수 있다. 즉시 `delete`하면 use-after-free가 된다.

대표적인 reclamation 방법은 다음과 같다.

- **Hazard pointer:** reader가 지금 사용할 pointer를 공개한다.
- **Epoch/EBR:** 제거 전 reader들이 이전 epoch를 모두 떠난 뒤 일괄 해제한다.
- **RCU:** reader path를 가볍게 두고 grace period 뒤 reclaim한다.

Hazard pointer reader는 pointer를 읽고 hazard를 공개한 뒤, 공유 pointer를 다시 읽어 같은지 확인해야 한다. 공개하기 직전에 reclaimer가 지웠을 수 있기 때문이다.

Epoch 방식은 reader가 싸지만 한 thread가 epoch를 떠나지 않으면 retired memory가 계속 쌓일 수 있다.

### 한 문장 기억법

> Lock-free 구조에서는 node를 목록에서 빼는 순간과 그 memory를 다시 써도 되는 순간이 다르다.

### 30초 면접 답변

> ABA는 CAS가 같은 bit pattern을 봤지만 그 사이 object나 논리 상태가 A에서 다른 상태를 거쳐 다시 A가 된 문제입니다. Version tag로 구분할 수 있지만 wrap을 봐야 하고, node unlink 뒤 reader가 남아 있을 수 있으므로 hazard pointer나 epoch/RCU 같은 safe reclamation이 필요합니다.

## 2.11 SPSC ring은 head와 tail에 각각 한 writer만 둔다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| SPSC | Single Producer Single Consumer. Producer와 consumer가 각각 한 개인 queue |
| Ring buffer | 고정 크기 slot 배열을 끝에서 처음으로 순환해 재사용하는 queue |
| Slot | Message나 object 하나를 저장하는 ring의 한 칸 |
| Head | Consumer가 다음에 읽을 위치나 소비 진행 상태를 나타내는 index |
| Tail | Producer가 다음에 쓸 위치나 생산 진행 상태를 나타내는 index |
| Capacity | Ring이 동시에 보관할 수 있는 최대 item 수 |
| Full·empty | `tail - head == capacity`면 full, `head == tail`이면 empty로 판단하는 대표 counter 방식 |
| Index wrap | 증가하는 index가 integer 범위를 돌아오거나 ring position이 처음으로 순환하는 현상 |
| Single writer | 특정 state를 수정하는 thread를 정확히 하나로 제한하는 ownership 규칙 |

SPSC는 single producer, single consumer queue다.

```text
Producer만 tail을 쓴다.
Consumer만 head를 쓴다.

head                                     tail
  ▼                                        ▼
[읽을 slot][읽을 slot][빈 slot][빈 slot][빈 slot]
```

여러 producer가 tail 하나를 경쟁하지 않으므로 CAS가 필요 없다. 핵심은 payload를 쓰는 시점과 slot을 공개하는 시점이다.

```cpp
// producer
auto t = tail.load(std::memory_order_relaxed); // producer만 write
auto h = head.load(std::memory_order_acquire); // consumer의 재사용 허용 관찰
if (t - h == capacity) return false;           // full
slot[t & mask] = value;                       // 1. payload 완성
tail.store(t + 1, std::memory_order_release); // 2. 공개

// consumer
auto h = head.load(std::memory_order_relaxed); // consumer만 write
auto t = tail.load(std::memory_order_acquire); // 3. 공개 관찰
if (h == t) return false;                      // empty
out = slot[h & mask];                         // 4. payload 읽기
head.store(h + 1, std::memory_order_release); // 5. 재사용 허용
```

Consumer가 producer의 새 tail을 acquire로 봤다면 payload write가 payload read보다 happens-before가 된다. 반대 방향에서는 consumer가 slot 읽기를 끝낸 뒤 head를 release해 producer가 그 slot을 덮어도 된다고 알린다.

### 구현 전제

- Mask indexing이면 capacity는 2의 거듭제곱이다.
- Head/tail은 충분히 넓은 unsigned counter를 사용한다.
- Producer-consumer 거리는 항상 `0..capacity`다.
- Non-trivial object를 raw slot에 두면 construct/destroy lifetime도 증명한다.
- Head와 tail을 서로 다른 cache line에 둘지 측정한다.

### 한 문장 기억법

> SPSC의 단순함은 atomic 기법보다 “각 index의 writer가 정확히 한 명”이라는 ownership에서 나온다.

### 30초 면접 답변

> SPSC에서는 producer만 tail, consumer만 head를 쓰므로 index update에 CAS가 필요 없습니다. Producer가 payload 뒤 release tail로 publish하고 consumer가 acquire tail 뒤 payload를 읽어 happens-before를 만듭니다. Consumer의 release head는 slot 재사용 방향을 보호합니다.

## 2.12 MPMC와 lock 선택은 경쟁 형태를 먼저 본다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| MPSC | Multiple Producer Single Consumer queue |
| MPMC | Multiple Producer Multiple Consumer queue |
| Reservation | Producer가 자신이 쓸 slot을 먼저 확보하는 단계 |
| Publication | Slot payload를 완성한 뒤 consumer가 읽어도 된다고 공개하는 단계 |
| Lap | Ring의 index가 capacity만큼 진행해 같은 physical slot 위치로 돌아온 한 세대 |
| Fan-in | 여러 input queue를 한 consumer가 모아 처리하는 구조 |
| Fairness | 여러 waiter나 queue가 무기한 굶지 않도록 처리 기회를 나누는 성질 |
| Mutex | 한 번에 한 thread만 critical section에 들어가게 하는 mutual-exclusion primitive |
| Futex | User-space integer 상태와 kernel sleep/wake를 연결하는 Linux primitive |
| Spinlock | Sleep하지 않고 lock 상태를 반복 확인하며 기다리는 lock |

SPSC에 producer가 여러 명이 되면 tail 증가만 atomic으로 바꿔서는 부족하다.

```text
P1: slot 10 예약 후 멈춤
P2: slot 11 예약하고 payload 완성
Consumer: tail=12만 보고 slot 10을 읽으면 미완성 data
```

즉 **slot 예약**과 **payload publication**을 분리해야 한다. Bounded MPMC는 흔히 slot마다 sequence/state를 둬 이번 lap의 empty, ready, reusable을 표시한다.

가능하다면 범용 MPMC 하나보다 producer별 SPSC queue와 한 consumer의 fan-in이 더 단순할 수 있다. 대신 queue 사이 fairness와 event ordering을 정해야 한다.

### Mutex, futex, spinlock

- **Mutex:** 경쟁이 없을 때 user-space atomic fast path로 끝날 수 있다. 오래 기다리면 kernel을 통해 잠들 수 있다.
- **Futex:** mutex 자체가 아니라, user-space word가 특정 값일 때 sleep/wake를 돕는 Linux primitive다.
- **Spinlock:** 잠들지 않고 반복 확인한다. 짧은 대기에는 유리할 수 있지만 CPU와 shared line을 계속 사용한다.

Owner가 preempt된 상태에서 다른 core가 spin하면 아무도 lock을 풀 수 없는데 CPU만 소모한다. 반대로 contention이 낮은 mutex는 복잡한 lock-free 구조보다 빠르고 이해하기 쉬울 수 있다.

### 한 문장 기억법

> 먼저 공유를 없애고, 남은 공유의 대기 길이와 owner가 실행 중인지에 따라 lock을 선택한다.

### 30초 면접 답변

> MPMC는 index 예약과 slot publication, 여러 consumer의 ownership과 reclamation을 함께 해결해야 하므로 SPSC보다 훨씬 어렵습니다. Mutex는 uncontended fast path가 짧을 수 있고 futex로 sleep할 수 있으며, spinlock은 owner가 곧 실행을 끝낼 때만 유리합니다. Lock-free라는 이름보다 실제 contention과 p99.9를 측정합니다.

# 3. Linux와 Network: Packet은 왜 바로 application에 오지 않는가?

## 3.1 System call과 context switch는 같은 말이 아니다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| User mode | Application code가 제한된 권한으로 실행되는 CPU mode |
| Kernel mode | OS kernel이 device와 memory 관리 같은 privileged operation을 수행하는 CPU mode |
| System call | Thread가 kernel service를 요청하기 위해 정해진 entry로 진입하는 mechanism |
| Task | Linux scheduler가 실행 대상으로 관리하는 단위. Process와 thread 설명에서 문맥에 따라 사용된다 |
| Context switch | CPU에서 실행 중인 task를 다른 task로 교체하는 동작 |
| Scheduler | Runnable task 중 어느 task를 어떤 CPU에서 실행할지 정하는 kernel 구성 요소 |
| Runnable | 실행할 준비가 되어 CPU 시간을 기다리는 task 상태 |
| Sleep·blocked | I/O나 synchronization event를 기다려 현재 실행할 수 없는 상태 |
| Page cache | File data를 memory에 보관해 storage I/O를 줄이는 Linux cache |
| vDSO | 일부 kernel 정보를 system call 진입 없이 user space에서 읽도록 제공하는 mechanism |
| PCID·ASID | Address space를 구분해 context switch의 TLB invalidation 비용을 줄일 수 있는 architecture 기능 |

Application이 file, network, scheduler 같은 kernel 관리 기능을 요청하는 것이 system call이다. 같은 thread가 kernel code를 실행한 뒤 바로 돌아올 수 있으므로 task 교체가 필수는 아니다.

```text
system call:
같은 thread가 user mode → kernel mode → user mode로 이동

context switch:
CPU에서 실행하는 task A → task B로 교체
```

`read()`가 page cache의 data를 즉시 찾으면 kernel에 들어갔다 같은 thread로 돌아올 수 있다. System call은 있었지만 context switch가 반드시 생기지는 않는다.

반대로 I/O를 기다려 현재 thread가 sleep하거나 scheduler가 선점하면 다른 task가 실행되며 context switch가 일어난다.

### Context switch의 비용

- Scheduler가 다음 task를 고른다.
- Register와 execution context를 교체한다.
- 다른 working set 때문에 cache locality가 나빠질 수 있다.
- Address space가 바뀌면 TLB locality도 영향을 받을 수 있다.
- Branch predictor와 front-end locality도 흔들릴 수 있다.

“모든 context switch가 TLB 전체를 flush한다”처럼 단정하면 안 된다. PCID/ASID와 같은 기능, 같은 process의 thread인지에 따라 달라진다.

`clock_gettime()`처럼 일부 호출은 vDSO를 통해 kernel mode 진입 없이 user space에서 처리될 수 있다. 어떤 clock과 플랫폼인지는 확인해야 한다.

### 한 문장 기억법

> System call은 권한 영역을 오가는 것이고, context switch는 CPU에서 실행할 task가 바뀌는 것이다.

### 30초 면접 답변

> System call은 같은 thread가 kernel service를 실행하기 위해 user mode에서 kernel mode로 전환하는 것이고, context switch는 실행 task가 교체되는 것입니다. 즉시 끝나는 syscall은 switch 없이 복귀할 수 있지만 I/O wait나 preemption이 있으면 switch가 생기며 cache와 scheduler 비용이 추가됩니다.

## 3.2 CPU affinity는 실행 가능한 CPU를 제한하지만 core를 격리하지 않는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| CPU affinity | Task가 실행될 수 있는 logical CPU 집합 |
| Pinning | Affinity를 좁혀 thread가 특정 CPU에서 실행되게 하는 설정 |
| Jitter | 같은 작업의 latency가 실행마다 불규칙하게 달라지는 현상 |
| IRQ | Device나 hardware event가 CPU에 처리를 요청하는 interrupt |
| Softirq | Linux가 interrupt 관련 작업 등을 뒤에서 처리하는 kernel execution context |
| Kernel thread | Kernel 내부 작업을 task 형태로 수행하는 thread |
| RCU callback | RCU grace period 뒤 실행되는 지연된 cleanup 작업 |
| SMT sibling | 한 physical core의 execution resource 일부를 공유하는 다른 logical CPU |
| Scheduler tick | 시간 관리와 scheduling을 위해 주기적으로 발생할 수 있는 kernel tick |
| Housekeeping CPU | Timer, unbound kernel work 등 일반 관리 작업을 맡기도록 남겨 둔 CPU |
| `nohz_full` | 조건을 만족하는 CPU에서 주기적 tick을 줄이는 Linux 설정 |
| `isolcpus`·cpuset isolation | 일반 scheduler load balancing에서 CPU를 분리하는 데 사용하는 Linux mechanism |

Thread를 CPU 4에 pinning했다고 하자. 이는 scheduler에게 “이 thread를 CPU 4에서만 실행하라”고 제한하는 것이다. “CPU 4에서는 이 thread만 실행하라”는 뜻은 아니다.

CPU 4에는 다음 일이 끼어들 수 있다.

- NIC interrupt와 softirq
- Kernel thread와 RCU callback
- 같은 CPU가 허용된 다른 user task
- Scheduler tick과 timer
- SMT sibling의 resource 경쟁
- Frequency와 power state 변화

### 저지연 CPU 구성은 묶음이다

```text
application affinity
+ NIC IRQ/queue affinity
+ housekeeping CPU 분리
+ RCU callback 위치
+ NUMA memory placement
+ SMT sibling 정책
+ scheduler policy
```

`isolcpus`, `nohz_full`, `rcu_nocbs`, cpuset partition은 각각 줄이는 일이 다르다. 하나의 boot option이 모든 noise를 없애지는 않는다.

`SCHED_FIFO`는 일반 task보다 높은 우선순위로 계속 실행할 수 있지만 runaway loop가 housekeeping과 recovery까지 굶길 수 있다. Watchdog와 운영 절차 없이 “가장 빠른 설정”으로만 사용하면 위험하다.

### Page fault도 함께 본다

Thread를 잘 고정해도 hot path에서 page fault가 나면 kernel 처리가 끼어든다. 시작 단계에서 pool과 stack을 allocate하고 page별 write로 pre-touch하며, `mlockall` 반환값과 `RLIMIT_MEMLOCK`을 확인한다.

### 한 문장 기억법

> Pinning은 thread의 후보 CPU를 줄일 뿐이며, 그 CPU에 들어오는 다른 일과 memory 위치까지 따로 정리해야 한다.

### 30초 면접 답변

> CPU affinity는 task가 실행될 CPU 집합을 제한할 뿐 core 독점을 보장하지 않습니다. IRQ, softirq, kernel work와 SMT sibling 경쟁이 남으므로 application, IRQ, housekeeping과 NUMA placement를 함께 구성하고 migration, page fault와 interrupt counter로 검증해야 합니다.

## 3.3 Ethernet frame은 여러 queue를 지나 `recv()`에 도착한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Ethernet frame | Ethernet link에서 전달되는 header, payload와 검증 정보를 포함한 data 단위 |
| PHY·MAC | 물리 신호 처리와 Ethernet frame 송수신·주소 처리를 담당하는 NIC 계층 |
| RX·TX | Receive와 transmit의 약어 |
| RX queue | NIC가 수신 packet과 descriptor completion을 분리해 관리하는 queue |
| MSI-X | NIC queue별 interrupt vector를 여러 CPU에 배치할 수 있게 하는 PCIe interrupt 방식 |
| Hardirq | CPU가 interrupt에 반응해 우선 실행하는 짧은 kernel handler context |
| NAPI | Linux network driver가 interrupt와 polling을 결합해 packet을 처리하는 interface |
| Budget | NAPI poll 한 번에서 처리할 RX packet 수를 제한하는 값 |
| `sk_buff`·skb | Linux network stack이 packet data와 metadata를 표현하는 핵심 구조 |
| Socket receive queue | Protocol 처리가 끝난 data가 application read를 기다리는 socket별 queue |
| XDP | `sk_buff` 생성 전의 이른 receive 지점에서 packet 처리 program을 실행하는 Linux mechanism |

Packet이 cable에 도착한 시점과 application이 읽는 시점 사이에는 여러 단계가 있다.

```text
wire
 → NIC PHY/MAC
 → RSS classification과 RX queue 선택
 → posted RX buffer로 DMA
 → descriptor completion
 → MSI-X interrupt 또는 polling
 → driver NAPI poll
 → XDP(optional)
 → Linux IP/UDP stack
 → socket receive queue
 → application wake-up
 → recv/recvmmsg
```

### 왜 interrupt handler에서 다 처리하지 않는가?

Packet마다 긴 hard interrupt 작업을 하면 높은 부하에서 interrupt가 CPU를 점령할 수 있다. Linux driver는 보통 짧은 hardirq에서 NAPI를 schedule하고, poll 함수가 여러 RX packet을 budget 범위에서 처리한다.

```text
낮은 부하: interrupt가 빠르게 packet 도착을 알림
높은 부하: NAPI polling으로 여러 packet을 묶어 처리
```

Work가 계속 밀리면 `ksoftirqd`가 처리할 수 있어 scheduler queueing이 tail에 나타날 수 있다. Threaded NAPI와 busy polling에서는 실행 context가 달라질 수 있다.

### `recv()` copy 횟수는 고정 문장이 아니다

일반 socket path에서는 kernel buffer에서 user buffer로 payload copy가 흔하다. 하지만 XDP, AF_XDP, `io_uring`, driver buffer model과 offload에 따라 경로가 달라진다. “항상 두 번 copy한다”처럼 외우지 않는다.

### 한 문장 기억법

> Packet은 wire에서 application까지 DMA ring, driver poll, protocol stack과 socket queue라는 대기 지점을 차례로 지난다.

### 30초 면접 답변

> NIC는 RSS로 RX queue를 고르고 posted buffer에 DMA한 뒤 descriptor completion을 기록합니다. MSI-X나 polling으로 CPU에 알리면 driver의 NAPI poll이 RX packet을 처리해 network stack과 socket receive queue로 넘기고, application이 `recv`로 읽습니다. 각 ring과 queue가 loss와 tail latency 지점입니다.

## 3.4 Descriptor ring은 buffer 소유권을 주고받는 계약이다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Descriptor | Packet buffer 주소, 길이, status와 option을 NIC에 전달하는 metadata 항목 |
| Descriptor ring | Descriptor를 고정 크기 순환 배열로 관리하는 queue |
| Buffer ownership | CPU와 NIC 중 누가 해당 buffer를 읽거나 수정·재사용할 수 있는지 정한 상태 |
| Completion | NIC가 descriptor 처리를 끝내 ownership이나 결과를 CPU에 알리는 상태 |
| Memory barrier | CPU memory와 device access가 요구된 순서로 관찰되도록 제한하는 platform operation |
| MMIO | CPU가 device register를 memory 주소 형태로 access하는 방식 |
| Doorbell | 새 descriptor 범위를 NIC에 알리는 MMIO register update |
| Posted write | CPU가 device의 최종 처리를 기다리지 않고 전송을 진행할 수 있는 PCIe write 특성 |
| Buffer recycle | 처리가 끝난 buffer를 다시 RX나 TX에 사용할 수 있게 반환하는 과정 |

Descriptor는 packet payload가 아니라 “어느 buffer를 어떤 길이와 option으로 처리할지” NIC에 알려주는 metadata다.

### RX ownership

```text
1. CPU: 빈 buffer 주소를 RX descriptor에 기록
2. CPU: descriptor를 NIC 소유로 게시
3. NIC: packet을 buffer에 DMA
4. NIC: completion/status 갱신
5. CPU: completion 확인 뒤 payload 읽기
6. CPU: 처리를 끝내고 buffer 재게시
```

CPU가 5번 전에 buffer를 읽으면 아직 DMA가 끝나지 않은 data를 볼 수 있다. CPU가 처리를 끝내기 전에 6번으로 넘기면 NIC가 사용 중인 buffer를 덮을 수 있다.

### TX ownership

```text
1. CPU: payload 작성
2. CPU: TX descriptor의 주소와 길이 작성
3. CPU: device용 write ordering 보장
4. CPU: tail/doorbell 갱신
5. NIC: descriptor와 payload 읽기
6. NIC: completion으로 buffer 반환
```

Doorbell은 “새 descriptor가 있다”고 장치에 알리는 신호다. Doorbell write 함수가 반환됐다는 것은 packet이 wire에 나갔다는 뜻이 아니다. TX buffer 재사용은 device가 정의한 completion/ownership을 보고 결정한다.

### Ring 크기의 trade-off

- 너무 작으면 burst를 흡수하지 못해 no-buffer drop이 생길 수 있다.
- 너무 크면 memory footprint와 cache pressure가 늘고 오래된 packet이 queue에 숨어 latency가 커질 수 있다.

### 한 문장 기억법

> Descriptor bit 하나는 단순 flag가 아니라 CPU와 NIC 중 누가 buffer를 만질 수 있는지 정하는 ownership handoff다.

### 30초 면접 답변

> RX에서는 CPU가 empty buffer를 게시하고 NIC가 DMA와 completion 뒤 ownership을 돌려줍니다. TX에서는 CPU가 payload와 descriptor를 쓴 뒤 barrier와 doorbell로 게시하고 completion 뒤에만 buffer를 재사용합니다. MMIO write 완료와 NIC 처리 완료, wire 송출 완료는 서로 다른 사건입니다.

## 3.5 RSS, RPS, RFS, XPS는 packet을 어느 CPU로 보낼지 정한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Flow | Source/destination address와 port 등으로 구분하는 packet 흐름 |
| Hash | 입력 header를 고정 크기 값으로 계산해 queue 선택에 사용하는 함수 결과 |
| Steering | Packet이나 처리를 특정 queue·CPU로 보내는 동작 |
| RSS | Receive Side Scaling. NIC hardware가 header hash로 RX queue를 선택하는 기능 |
| RPS | Receive Packet Steering. Linux가 수신 packet의 이후 처리를 다른 CPU backlog로 보내는 기능 |
| RFS | Receive Flow Steering. Flow를 consuming application CPU에 가깝게 보내려는 Linux 기능 |
| XPS | Transmit Packet Steering. CPU와 queue mapping을 사용해 TX queue를 고르는 Linux 기능 |
| Backlog | 아직 처리되지 않고 CPU별 software queue에 쌓인 packet |
| Queue affinity | Queue interrupt나 polling thread를 특정 CPU와 연결하는 설정 |
| Flow ordering | 같은 flow의 packet을 protocol이 요구하는 순서대로 처리하는 성질 |

이름이 비슷하지만 위치가 다르다.

| 기능 | 위치 | 하는 일 |
| --- | --- | --- |
| RSS | NIC hardware | Header hash로 RX queue를 선택 |
| RPS | Linux software | 수신 packet 처리를 다른 CPU backlog로 보냄 |
| RFS | Linux software | Flow를 소비 application CPU에 가깝게 유도 |
| XPS | Linux software | TX queue 선택에 CPU/queue mapping을 사용 |

RSS를 사용하면 여러 queue와 core로 부하를 나눌 수 있다. 그러나 무조건 queue를 많이 만들면 좋은 것은 아니다.

- 한 flow의 packet ordering을 유지해야 한다.
- Queue마다 ring과 state가 있어 memory footprint가 늘어난다.
- Processing thread가 다른 NUMA node면 remote access가 생긴다.
- RPS handoff는 software queue와 cache 이동을 추가한다.

### Flow affinity의 목표

같은 flow를 같은 queue와 같은 parser state owner에게 보내면 lock과 state migration을 줄일 수 있다. 반대로 hot flow 하나가 queue를 독점하면 RSS hash 분산이 균등하지 않을 수 있다.

### 검증할 실제 값

- `/proc/interrupts`에서 vector가 실행되는 CPU
- NIC queue별 packet/drop counter
- RPS/RFS/XPS 설정
- Thread affinity와 migration
- Buffer page의 NUMA node
- Queue backlog와 oldest packet age

### 한 문장 기억법

> Steering의 목적은 CPU 수를 많이 쓰는 것이 아니라, 한 flow의 packet과 state를 예측 가능한 owner에게 보내는 것이다.

### 30초 면접 답변

> RSS는 NIC가 hash로 RX queue를 선택하고, RPS/RFS는 Linux가 이후 처리를 다른 CPU나 consuming application 쪽으로 유도하며, XPS는 TX queue 선택을 돕습니다. 분산은 처리량을 높일 수 있지만 software handoff와 NUMA remote access, flow ordering을 함께 봐야 합니다.

## 3.6 Offload와 interrupt moderation은 일을 없애기보다 묶는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Offload | CPU나 network stack의 일부 작업을 NIC 또는 다른 layer에 맡기는 기능 |
| GRO | Generic Receive Offload. 여러 receive packet을 더 큰 단위로 합쳐 상위 stack 처리 횟수를 줄이는 기능 |
| GSO | Generic Segmentation Offload. 큰 data를 lower layer가 packet 단위로 나누도록 넘기는 software mechanism |
| TSO | TCP Segmentation Offload. TCP segmentation 일부를 NIC에 맡기는 기능 |
| Checksum offload | Packet checksum 계산이나 검증 일부를 NIC에 맡기는 기능 |
| Interrupt moderation·coalescing | 여러 packet completion을 모아 interrupt 빈도를 줄이는 NIC 기능 |
| Batch | 여러 item을 한 번의 loop나 notification에서 묶어 처리하는 단위 |
| Packet boundary | 한 packet이 시작하고 끝나는 구분 |

### GRO와 TSO

- **GRO:** Receive path에서 여러 packet을 더 큰 단위로 합쳐 stack 처리 비용을 줄인다.
- **TSO/GSO:** 큰 data를 kernel에서 넘기고 NIC 또는 lower layer가 packet 단위로 나눈다.
- **Checksum offload:** Checksum 계산 일부를 NIC에 맡긴다.

이들은 CPU instruction과 packet당 overhead를 줄여 throughput을 높일 수 있다. 하지만 여러 packet을 모으거나 큰 batch로 처리하는 동안 첫 packet이 기다릴 수 있고, packet boundary를 그대로 보고 싶은 market data path에는 맞지 않을 수 있다.

### Interrupt moderation

NIC가 packet마다 즉시 interrupt를 보내지 않고 일정 수나 시간 동안 모아 알리면 interrupt overhead가 줄어든다.

```text
coalescing 작음: wake-up 빠름, interrupt 많음
coalescing 큼: overhead 감소, 첫 packet 대기 증가 가능
```

Coalescing을 0으로 만들면 무조건 p99가 좋아지는 것도 아니다. Interrupt storm과 CPU starvation이 오히려 tail을 키울 수 있다. Adaptive moderation도 부하 변화에 따라 latency 분포를 바꿀 수 있다.

### 한 문장 기억법

> Offload와 batching은 packet당 일을 줄이는 대신, 묶음이 만들어질 때까지의 대기를 추가할 수 있다.

### 30초 면접 답변

> GRO, TSO와 checksum offload는 CPU overhead를 줄이지만 batching과 packet-boundary 변화가 latency에 영향을 줄 수 있습니다. Interrupt moderation도 interrupt 수를 줄이는 대신 wake-up delay를 만들 수 있으므로 median뿐 아니라 burst 부하의 p99.9와 drop을 측정해 선택합니다.

## 3.7 UDP loss는 한 counter로 찾지 않는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| UDP | Delivery, ordering과 retransmission을 protocol 자체에서 보장하지 않는 datagram transport protocol |
| Sequence gap | 다음에 기대한 sequence보다 큰 값이 도착해 중간 message 누락 가능성을 발견한 상태 |
| Drop | Queue나 validation 단계가 packet·message를 폐기한 사건 |
| Overflow | Buffer나 queue capacity를 넘어 새 item을 보관하지 못하는 상태 |
| Hardware counter | NIC와 switch가 packet, error, drop 등을 누적 기록하는 값 |
| Softnet statistics | Linux network backlog와 drop 등 receive processing 상태를 보여주는 통계 |
| Packet capture | 특정 network 경계에서 관찰한 packet을 file이나 buffer에 기록하는 작업 |
| Retransmission | 누락한 sequence 범위의 message를 다시 요청·전송하는 recovery 방식 |
| Invalid state | 일부 event 누락 때문에 correctness를 보장할 수 없어 전략 사용을 막아야 하는 상태 |

UDP application이 sequence gap을 발견했다고 해서 cable에서 packet이 사라졌다고 바로 결론 내릴 수 없다.

```text
wire loss
 → NIC CRC/filter drop
 → RX ring no-buffer drop
 → driver/NAPI backlog drop
 → IP/UDP error
 → socket receive-buffer overflow
 → application queue full
 → parser가 늦어 sequence gap을 잘못 판단
```

### 바깥에서 안쪽으로 좁힌다

1. Protocol sequence number로 gap의 정확한 범위를 기록한다.
2. Switch port와 NIC hardware counter를 본다.
3. Driver queue별 drop과 no-buffer를 본다.
4. Linux softnet/backlog와 UDP counter를 본다.
5. Socket overflow와 application queue를 본다.
6. Packet capture 위치와 timestamp가 어느 경계를 관찰하는지 확인한다.

Packet capture에 없다고 capture 지점 이전에서 잃었다는 뜻은 아니다. Capture 자체가 drop했거나 offload 때문에 보이는 형태가 달라질 수 있다.

### Gap 뒤 state를 어떻게 할까?

Incremental order book update 하나를 잃으면 이후 update가 정상이어도 book state는 틀릴 수 있다. 조용히 계속 적용하지 않고 book을 invalid로 만들고 recovery protocol을 시작한다.

### 한 문장 기억법

> UDP loss는 protocol sequence를 기준으로 잡고, wire부터 application queue까지 경계별 counter로 범위를 좁힌다.

### 30초 면접 답변

> UDP gap은 wire, NIC, RX ring, softnet backlog, socket buffer와 application queue 어디서든 생길 수 있습니다. Sequence gap을 기준으로 switch/NIC/driver/kernel/socket/application counter를 같은 시간축에서 비교하고, incremental state는 gap 순간 invalid 처리해 recovery합니다.

## 3.8 Kernel bypass는 data path를 줄이는 대신 application 책임을 늘린다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Data path | Packet data가 NIC에서 application까지 실제로 통과하는 code와 queue 경로 |
| Control path | Queue 설정, link 관리, recovery처럼 data path를 구성·관리하는 경로 |
| Kernel bypass | 일반 kernel network stack의 일부를 지나지 않고 user space가 NIC queue와 buffer를 더 직접 처리하는 방식 |
| Busy polling | Sleep하지 않고 packet이나 completion 상태를 반복 확인하는 방식 |
| Zero-copy | 경계 사이에서 payload byte copy를 줄이거나 없애는 buffer 공유 방식. Ownership 전환은 여전히 필요하다 |
| DPDK | User-space polling driver, huge-page memory와 packet buffer library를 제공하는 framework |
| AF_XDP | XDP와 user-space UMEM ring을 연결해 짧은 packet path를 제공하는 Linux socket family |
| UMEM | AF_XDP에서 kernel과 user space가 packet buffer로 공유하는 memory 영역 |
| Observability | Counter, trace와 log를 통해 내부 상태와 문제를 관찰할 수 있는 능력 |
| Dedicated core | 특정 polling·processing 작업에 사실상 전용으로 사용하는 CPU core |

일반 Linux network stack은 socket API, routing, filtering과 여러 protocol 기능을 제공한다. DPDK나 AF_XDP 같은 방식은 packet을 user space에서 직접 polling하거나 더 짧은 data path로 처리해 syscall, allocation, scheduler wake-up과 stack overhead를 줄일 수 있다.

```text
일반 socket:
NIC → driver/NAPI → kernel stack → socket queue → application

kernel bypass 계열:
NIC/driver queue → shared/user buffer ring → polling application
```

### 빨라지는 이유

- Dedicated polling으로 wake-up 지연을 줄인다.
- Buffer를 사전 할당하고 copy를 줄일 수 있다.
- Batch로 descriptor 처리 비용을 나눈다.
- Queue와 core ownership을 고정하기 쉽다.

### 잃거나 직접 책임질 수 있는 것

- Dedicated core와 높은 전력 사용
- Driver/device별 설정과 운영 복잡성
- Routing, firewall, observability 도구 일부
- Buffer lifecycle와 DMA ordering
- Overload, fairness, link reset와 recovery
- Library와 NIC firmware upgrade 호환성

Kernel bypass여도 PCIe, DMA, cache miss와 queueing은 사라지지 않는다. Poll loop가 느려지면 RX ring은 여전히 넘칠 수 있다.

### 한 문장 기억법

> Kernel bypass는 일부 kernel 경로 비용을 줄이는 대신, buffer ownership과 overload·recovery를 application이 더 직접 책임지는 설계다.

### 30초 면접 답변

> DPDK나 AF_XDP는 polling, preallocated buffer와 짧은 data path로 syscall과 kernel-stack overhead를 줄일 수 있습니다. 대신 dedicated core, buffer ownership, DMA ordering, overload와 운영·관측 기능을 application이 책임해야 하므로 목표 부하에서 end-to-end tail을 비교합니다.

# 4. Latency 측정: 빠르다는 말을 숫자로 어떻게 증명하는가?

## 4.1 평균은 드문 긴 지연을 숨긴다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Sample | Operation 한 번에서 측정한 latency 값 하나 |
| Distribution | 여러 sample이 어떤 범위와 빈도로 나타나는지 보여주는 전체 형태 |
| Mean·average | 모든 sample 합을 sample 수로 나눈 평균 |
| Median·p50 | 정렬한 sample의 중앙에 해당하는 값 |
| Percentile·quantile | 정렬한 sample 중 지정 비율이 해당 값 이하가 되도록 고른 위치 값 |
| p99·p99.9 | 약 99% 또는 99.9%의 sample이 해당 값 이하라는 의미 |
| Outlier | 대부분의 sample 범위에서 크게 벗어난 관측값 |
| Maximum | 측정 구간에서 관찰한 가장 큰 값. System의 이론적 상한이라는 뜻은 아니다 |
| Burst | 짧은 시간에 평소보다 많은 packet이나 request가 몰리는 입력 형태 |

다음 열 번의 latency를 보자.

```text
10, 10, 10, 10, 10, 10, 10, 10, 10, 1000 microseconds
```

평균은 109 microseconds다. 하지만 실제 경험은 대부분 10이고 한 번은 1000이다. 평균 하나만으로는 이 두 모습을 설명하기 어렵다.

### Percentile의 뜻

- p50: 관측값의 절반이 이 값 이하
- p99: 약 99%가 이 값 이하
- p99.9: 약 99.9%가 이 값 이하
- max: 측정 구간에서 가장 큰 관측값

Percentile은 확률적 보장이지 절대 상한이 아니다. 표본이 1,000개뿐이면 p99.99를 정밀하게 말할 수 없다. 측정 시간, 표본 수, quantile 계산법을 함께 기록해야 한다.

### 왜 저지연에서 tail이 중요한가?

시장 burst 때 느린 message가 queue를 만들면 뒤 message까지 함께 늦어진다. Risk나 execution 처리의 outlier는 단일 요청을 넘어 상태 stale과 overload로 번질 수 있다.

### 한 문장 기억법

> 평균은 총 비용을 보여주지만, percentile과 max는 장애에 가까운 드문 실행을 보여준다.

### 30초 면접 답변

> 평균은 드문 긴 지연을 희석하므로 저지연 시스템에서는 p50, p99, p99.9, max와 sample count를 함께 봅니다. Tail outlier는 queueing을 통해 뒤 message에도 전파될 수 있으며 percentile은 절대 상한이 아니므로 측정 기간과 loss도 함께 보고합니다.

## 4.2 Coordinated omission은 느릴 때 요청을 덜 보내는 측정 오류다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Load generator | 측정 대상 system에 request나 packet을 만들어 보내는 program |
| Arrival rate | 외부에서 단위 시간당 도착하는 request 수 |
| Service time | Queue 대기를 제외하고 server가 request 자체를 처리하는 시간 |
| Response time | Request 도착 또는 예정 시점부터 응답 완료까지의 전체 시간 |
| Closed loop | 이전 response를 받은 뒤 다음 request를 보내는 부하 생성 방식 |
| Open loop | 이전 response와 독립적으로 예정된 arrival schedule에 따라 request를 보내는 방식 |
| Intended send time | 부하 모델상 request가 원래 도착했어야 하는 시각 |
| Generator lag | Load generator가 예정 시각보다 늦게 request를 실제 전송한 지연 |
| Coordinated omission | 대상이 느려질 때 generator도 request를 덜 보내 queueing sample이 누락되는 측정 오류 |

Closed-loop benchmark는 다음 response를 기다리므로 system이 느린 구간에 새 request를 만들지 않는다.

```text
request 전송
  → response를 기다림
  → 끝난 뒤 다음 request 전송
```

System이 느려지면 test generator도 요청을 덜 보낸다. 실제 시장처럼 외부 도착률이 계속되는 상황에서 생길 queueing을 측정하지 못한다. 이를 coordinated omission이라 한다.

### 피하는 방법

- 예정된 도착 시간에 따라 open-loop로 traffic을 생성한다.
- 각 request의 intended send time과 actual send time을 기록한다.
- Generator 자체가 목표 rate를 감당하는지 확인한다.
- Production trace의 timestamp 간격을 보존해 replay한다.
- Overload 시 drop과 queue age도 latency와 함께 기록한다.

Closed loop가 쓸모없는 것은 아니다. 한 operation의 service time이나 최대 throughput을 보는 실험에는 의미가 있다. 다만 “실제 arrival process의 latency”라고 해석하면 안 된다.

### 한 문장 기억법

> System이 느려질 때 부하 생성기도 쉬어 버리면, 가장 중요한 queueing 지연이 측정에서 빠진다.

### 30초 면접 답변

> Coordinated omission은 benchmark가 이전 응답을 기다리느라 느린 구간에 새 요청을 보내지 않아 실제 queueing latency를 누락하는 문제입니다. 외부 arrival을 모델링할 때는 intended schedule 기반 open-loop load와 request별 end-to-end latency, generator lag를 기록해야 합니다.

## 4.3 Timestamp는 어느 clock과 어느 경계인지가 핵심이다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Timestamp | 특정 event가 발생했다고 기록한 시간 값 |
| Clock source | Timestamp 값의 기준이 되는 hardware나 OS 시간원 |
| Clock domain | 같은 시간 기준과 조정 규칙을 공유하는 timestamp 범위 |
| TSC | Time Stamp Counter. X86에서 증가하는 hardware counter |
| Invariant TSC | Core frequency 변화와 분리된 일정 비율로 증가하도록 제공되는 TSC 특성 |
| Monotonic clock | 정상 동작 중 뒤로 가지 않는 경과 시간용 clock |
| `CLOCK_REALTIME` | 현재 wall-clock 시간을 나타내며 시간 보정의 영향을 받을 수 있는 Linux clock |
| PTP | Precision Time Protocol. Host와 NIC clock 등을 정밀하게 동기화하는 protocol과 기능 |
| Hardware timestamp | NIC hardware가 RX·TX의 특정 지점에서 기록한 timestamp |
| Clock synchronization error | 서로 다른 clock을 맞춘 뒤에도 남는 시간 차이와 불확실성 |

`start`와 `end`를 찍었다고 측정 의미가 자동으로 정해지지 않는다.

### 대표 clock

| Clock | 쉬운 용도 | 주의점 |
| --- | --- | --- |
| TSC | 같은 host의 매우 짧은 구간 | invariant/sync와 conversion 확인 |
| `clock_gettime` monotonic 계열 | process 내부 경과 시간 | 선택한 clock의 조정 의미 확인 |
| PTP hardware timestamp | NIC에 가까운 wire 경계 | NIC/driver가 찍는 정확한 위치 확인 |

TSC tick은 반드시 현재 core frequency의 한 cycle과 같은 뜻이 아니다. 현대 x86의 invariant TSC는 frequency 변화와 독립된 일정 비율로 증가할 수 있다.

`CLOCK_REALTIME`은 벽시계 보정으로 이동할 수 있어 짧은 duration에는 monotonic clock이 더 적합하다. Host 사이 timestamp를 비교하려면 PTP/NTP 동기화 오차와 clock domain을 기록해야 한다.

Hardware timestamp도 “wire의 정확한 첫 bit”라고 단정하지 않는다. MAC 앞인지 뒤인지, RX와 TX에서 어떤 event를 찍는지는 NIC와 driver 문서를 확인한다.

### 측정 overhead

Timestamp read 자체도 instruction과 memory write를 추가한다. 모든 packet에 여러 timestamp를 기록하면 cache footprint와 branch에 영향을 줄 수 있다. Sampling과 dedicated trace buffer를 검토하고, 측정 on/off 차이를 비교한다.

### 한 문장 기억법

> 숫자보다 먼저 “어느 clock으로, 어느 경계의 어떤 사건을 찍었는가?”를 말해야 한다.

### 30초 면접 답변

> TSC는 낮은 overhead의 local interval에 유용하고 monotonic `clock_gettime`은 OS가 정의한 시간축을 제공하며 PTP hardware timestamp는 NIC에 가까운 경계를 관찰합니다. 서로 다른 clock domain을 바로 빼지 않고 synchronization error, timestamp 위치와 측정 overhead를 명시해야 합니다.

## 4.4 Microbenchmark는 측정할 operation과 실행 조건을 통제해야 한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Microbenchmark | 함수나 짧은 code 조각의 비용을 격리해 측정하는 작은 benchmark |
| Dead-code elimination | 결과가 관찰되지 않는 code를 compiler가 제거하는 최적화 |
| Constant folding | Compile time에 값을 계산해 runtime operation을 없애는 최적화 |
| Warm-up | Code, cache, allocator와 runtime 상태가 초기 일회성 영향을 벗어나도록 미리 실행하는 단계 |
| Cold·warm cache | 필요한 data가 cache에 없는 상태와 이미 있는 상태 |
| Background load | 측정 대상 외에 CPU, memory, network를 사용하는 다른 작업 |
| Assembly | Compiler가 생성한 target ISA instruction 표현 |
| Controlled variable | 원인 비교를 위해 실험에서 의도적으로 바꾸거나 고정하는 조건 |
| Production trace | 실제 운영에서 기록한 input 순서, timing과 특성 |

다음 benchmark는 잘못될 수 있다.

```cpp
for (int i = 0; i < 1'000'000; ++i) {
    calculate(i); // 결과를 전혀 사용하지 않음
}
```

Compiler는 결과가 관찰되지 않으면 loop 전체를 제거할 수 있다. 반대로 `volatile`을 남용하면 production에는 없는 memory access를 측정하게 된다.

### 신뢰할 수 있는 실험 순서

1. 무엇을 측정할지 한 문장으로 정의한다.
2. Input 분포를 production과 cold/warm scenario로 나눈다.
3. 결과를 관찰 가능하게 만들어 dead-code elimination을 막는다.
4. Affinity, frequency, NUMA와 background load를 기록한다.
5. 충분히 warm-up하고 여러 process run을 반복한다.
6. Assembly로 실제 loop가 원하는 코드인지 확인한다.
7. Median, tail, confidence와 raw sample을 보관한다.
8. 한 번에 한 변수만 바꾼다.

Cache를 매번 flush하는 실험은 cold-cache 질문에는 맞지만, production hot working set을 대표하지 않을 수 있다. 한 benchmark로 모든 상황을 대표시키지 않는다.

### 한 문장 기억법

> Benchmark는 code를 실행하는 일이 아니라, 한 가설만 남도록 환경과 compiler output을 통제하는 실험이다.

### 30초 면접 답변

> Microbenchmark는 dead-code elimination과 constant folding을 막고 실제 assembly를 확인해야 합니다. Affinity, NUMA, frequency, warm-up과 input distribution을 고정하고 반복 실행의 분포를 봅니다. Cold와 warm scenario를 분리하며 한 번에 한 변수만 바꿔 production trace로 다시 검증합니다.

## 4.5 단계별 p99를 더해도 end-to-end p99가 되지 않는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| End-to-end latency | Request가 system 첫 경계에 들어와 최종 결과가 나올 때까지의 전체 latency |
| Stage latency | Network, parse, strategy처럼 한 처리 구간에서 측정한 latency |
| Marginal distribution | 각 stage를 따로 봤을 때의 latency distribution |
| Joint distribution | 같은 request에서 여러 stage 값이 함께 어떻게 나타나는지 포함한 distribution |
| Correlation | 두 측정값이 함께 증가·감소하는 경향을 나타내는 통계 관계 |
| Causality | 한 사건이 다른 사건의 원인이 되는 관계. Correlation만으로 확정할 수 없다 |
| PMU counter | CPU PMU가 세는 cache miss, branch miss, retired instruction 등의 hardware event 값 |
| Multiplexing | 제한된 PMU counter를 여러 event가 시간 분할해 사용하는 측정 방식 |
| Sampling skid | Sampling interrupt가 원인이 된 정확한 instruction보다 조금 뒤에서 기록될 수 있는 현상 |
| Retired·speculative event | 확정된 instruction의 event와 추측 실행 중 발생한 event의 구분 |

두 요청의 단계별 latency가 다음과 같다고 하자.

```text
Request A: network 1, strategy 100
Request B: network 100, strategy 1
```

각 단계의 느린 값 100을 더하면 200이지만, 실제 두 요청의 합은 각각 101이다. 반대로 느린 단계들이 같은 요청에 함께 몰리면 tail이 더 커질 수 있다.

각 단계의 percentile은 **주변 분포**만 보여준다. 어느 request에서 느린 값들이 함께 발생했는지라는 joint information이 없다. 따라서 같은 message ID에 구간 timestamp를 연결해 request별 end-to-end 값을 직접 계산한다.

### PMU counter도 단독 판결문이 아니다

`cache-misses`와 latency가 함께 늘었다고 cache miss가 유일한 원인이라고 바로 결론 내릴 수 없다.

- Queueing이 늘며 working set도 커졌을 수 있다.
- 다른 원인이 두 값을 함께 바꿨을 수 있다.
- Event multiplexing과 sampling skid가 있을 수 있다.
- Speculative event와 retired event가 다를 수 있다.

좋은 분석은 trace로 느린 sample을 찾고, PMU와 assembly로 가설을 만들며, code/layout A/B test로 원인을 검증한다.

### 한 문장 기억법

> End-to-end tail은 같은 요청을 끝까지 추적해 계산하고, PMU는 원인을 찾는 단서로 사용한다.

### 30초 면접 답변

> 단계별 p99는 서로 다른 request에서 발생할 수 있어 합이 end-to-end p99가 아닙니다. 동일 message의 timestamp를 연결해 전체 분포를 직접 계산해야 합니다. PMU counter는 병목 가설을 좁히지만 인과를 확정하지 않으므로 assembly와 controlled A/B experiment로 검증합니다.

# 5. Trading correctness: 빠른 상태 머신은 무엇을 지켜야 하는가?

## 5.1 Incremental market data는 event 누락 없이 순서대로 적용해야 한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Market data | 거래소가 제공하는 주문, 가격, 수량, 체결과 상태 정보 |
| Order book | 현재 매수·매도 주문이나 가격 단계의 상태를 표현한 자료구조 |
| Bid·ask | 사려는 쪽의 가격·수량과 팔려는 쪽의 가격·수량 |
| Incremental update | 전체 state가 아니라 직전 state에서 바뀐 event만 전달하는 message |
| Sequence number | Message 누락, 중복과 순서를 판단하도록 stream에서 증가하는 번호 |
| Expected sequence | 다음에 받아야 한다고 receiver가 계산한 sequence |
| Gap | Expected보다 큰 sequence가 도착해 중간 message 누락 가능성이 생긴 상태 |
| Duplicate | 이미 처리한 identity·sequence의 message가 다시 도착한 상태 |
| Order-level feed | 개별 order add, modify, delete를 전달하는 feed |
| Price-level feed | 가격 단계별 집계 수량과 변경을 전달하는 feed |
| Delta·absolute | 이전 값에 더할 변화량과 새 전체 값을 그대로 주는 표현 방식 |
| Stale state | 최신 event를 반영하지 못해 현재 시장과 달라진 state |

거래소는 매번 전체 order book을 보내기보다 “무엇이 바뀌었는가”를 연속 update로 보낼 수 있다.

```text
seq 100: price 101에 bid 10 추가
seq 101: price 102의 ask 5 제거
seq 102: price 101의 bid 3 체결
```

처음 상태에 100, 101, 102를 순서대로 적용하면 새 상태를 얻는다. 그런데 101을 잃고 102만 적용하면 program은 자신 있게 틀린 book을 만든다.

### Sequence number가 필요한 이유

Sequence는 다음 update가 정확히 무엇인지 알려준다.

```text
expected = 101
received = 101 → apply, expected=102
received < 101 → duplicate/late 여부를 protocol 규칙으로 처리
received > 101 → gap, 현재 book을 신뢰하지 않음
```

Gap을 발견하면 “다음 packet은 왔으니 계속하자”가 아니라 symbol/channel state를 invalid로 만들고 retransmission이나 snapshot recovery를 시작한다.

### Order-level과 price-level update

Order-level feed는 개별 order의 add, modify, delete를 적용한다. Price-level feed는 가격 단계별 수량을 보낸다. 같은 `quantity` field라도 absolute value인지 delta인지 protocol마다 다르므로 specification을 따라야 한다.

### 필요한 불변식

- Sequence가 protocol 규칙대로 연속이다.
- 존재하지 않는 order를 임의로 delete하지 않는다.
- Quantity는 허용 범위 안에 있다.
- Best bid가 best ask보다 비정상적으로 넘어가는 상태는 feed 의미를 확인한다.
- Gap 이후에는 valid flag 없이 전략이 stale book을 사용하지 않는다.

### 한 문장 기억법

> Incremental feed는 모든 update가 누락 없이 protocol 순서대로 적용돼야 현재 상태를 재구성할 수 있다.

### 30초 면접 답변

> Incremental market data는 sequence를 검증하며 deterministic state transition으로 적용합니다. Expected보다 큰 sequence가 오면 gap으로 보고 book을 invalid 처리한 뒤 retransmission이나 snapshot recovery를 수행합니다. Duplicate와 reset, absolute/delta quantity 의미는 venue specification을 따릅니다.

## 5.2 Snapshot과 live update 사이의 틈을 막아야 한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Snapshot | 특정 기준 sequence·시점의 전체 state를 담은 data |
| Live update | 현재 계속 수신되는 실시간 incremental message |
| Recovery buffer | Snapshot을 받는 동안 도착한 incremental update를 임시 보관하는 bounded queue |
| Baseline sequence | Snapshot state가 이미 반영하고 있는 마지막 sequence |
| Replay | 저장한 update를 sequence 순서대로 state에 다시 적용하는 과정 |
| Catch-up | Buffered update를 적용해 현재 live sequence 가까이 따라잡는 과정 |
| Handoff | Recovery mode에서 live mode로 ownership과 처리 상태를 넘기는 전환 |
| Race condition | 여러 실행의 timing에 따라 결과가 달라져 잘못된 state가 생길 수 있는 조건 |
| Handshake | Producer와 consumer가 특정 전환점과 상태를 서로 확인하는 synchronization 절차 |
| Contiguous | Sequence 사이에 빈 번호 없이 연속된 상태 |
Snapshot은 특정 기준 시점의 전체 상태다. 하지만 snapshot을 network로 받는 동안 live incremental update는 계속 온다.

```text
시간 ─────────────────────────────────→
       snapshot의 기준 S
       │             snapshot 도착
       │             │
live: S+1, S+2, S+3, S+4, S+5 ...
```

Snapshot을 받은 뒤 그때부터 live를 구독하면 S 이후와 구독 사이 update를 잃을 수 있다. 그래서 흔한 복구 순서는 다음과 같다.

```text
1. Incremental 수신을 먼저 시작하고 bounded buffer에 저장
2. Snapshot 요청
3. Snapshot 무결성과 기준 sequence S 확인
4. Buffered update 중 S 이하 제거
5. S+1부터 빈틈없이 적용
6. Buffer를 따라잡음
7. Producer와 handshake해 live mode로 전환
```

### 마지막 handoff race

Consumer가 “buffer가 비었다”고 확인한 직후 producer가 새 update를 넣고, consumer가 곧바로 live flag만 바꾸면 update 하나를 놓칠 수 있다. Recovery와 live 사이에 틈이 없어야 한다.

해결 방법은 다음 중 하나다.

- Recovery와 live를 계속 같은 single sequenced consumer가 처리
- Lock이나 epoch handshake로 producer와 전환점을 합의
- 하나의 queue를 계속 소비하고 state mode만 변경

Recovery buffer가 가득 차면 오래된 update를 덮어쓰지 않는다. Recovery 실패로 처리하고 snapshot을 다시 요청한다.

### 한 문장 기억법

> Snapshot의 기준 sequence 뒤에 도착한 모든 incremental update를 빈틈없이 적용해야 recovery state와 live state가 연결된다.

### 30초 면접 답변

> Incremental을 먼저 buffer하고 snapshot의 기준 sequence S를 확인한 뒤 S+1부터 contiguous하게 replay합니다. Catch-up 완료와 live 전환 사이의 producer-consumer race도 single sequenced queue나 handshake로 막아야 하며, buffer overflow는 조용히 덮지 않고 recovery failure로 처리합니다.

## 5.3 Cancel 요청은 주문이 취소됐다는 사실이 아니다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Order lifecycle | New request부터 fill, cancel, reject 등으로 주문 상태가 변하는 전체 과정 |
| `Live` | Venue에서 주문이 유효해 체결될 수 있는 상태 |
| `PartiallyFilled` | 주문 수량 일부가 체결되고 실행 가능한 잔량이 남은 상태 |
| `CancelPending` | Cancel request를 보냈지만 venue 결과가 아직 확정되지 않은 상태 |
| Acknowledgement·ack | Venue가 요청 처리 결과를 확인해 보내는 응답 |
| Reject | Venue가 new, cancel, replace 요청을 받아들이지 않은 결과 |
| Terminal state | 일반적으로 더 이상 실행 가능한 주문으로 진행하지 않는 `Filled`, `Canceled`, `Rejected` 등의 상태 |
| `CumQty` | 지금까지 누적 체결된 수량. Correction event가 있으면 감소할 수 있다 |
| `LeavesQty` | 현재 주문에서 실행 가능한 잔여 수량. Inactive order에서는 단순 미체결 수량과 다를 수 있다 |
| Authoritative event | Local 추측보다 우선해 state에 반영해야 하는 venue의 공식 event |
| Unknown | Venue 처리 결과를 확인하지 못해 live·terminal을 확정할 수 없는 상태 |
| Drop copy | Venue나 별도 channel이 제공하는 체결·주문 활동의 독립 확인 feed |
사용자가 cancel 버튼을 누른 순간과 거래소가 cancel을 승인한 순간 사이에 주문은 체결될 수 있다.

```text
내 시스템: cancel request 전송
거래소:    이미 match 중이거나 추가 fill 발생
network:   fill과 cancel ack가 각 protocol 순서로 도착
```

따라서 `CancelPending`은 terminal 상태가 아니다. “취소 작업이 진행 중”이라는 별도 차원이다.

```text
Live/PartiallyFilled -- cancel request --> CancelPending
CancelPending -- partial fill --> CancelPending, cumQty 갱신
CancelPending -- full fill --> Filled
CancelPending -- cancel ack --> Canceled
CancelPending -- cancel reject --> Live/PartiallyFilled/Unknown
```

구현에서는 fill status와 pending operation을 두 field로 나누는 편이 이해하기 쉽다.

```cpp
struct OrderState {
    FillStatus fill_status;
    PendingOperation pending;
    Quantity cum_qty;
    Quantity executable_leaves;
};
```

### Quantity 불변식의 예외

Active order에서는 보통 `LeavesQty = OrderQty - CumQty`다. 하지만 canceled, expired, rejected 주문은 미체결 수량이 남았어도 지금 실행 가능한 leaves는 0일 수 있다.

일반 fill만 처리할 때 `CumQty`는 증가한다. Trade bust/cancel/correction event가 있으면 명시적인 correction으로 감소할 수 있다. “절대 감소하지 않는다”라고 전역 불변식으로 두면 잘못이다.

### Disconnect도 cancel이 아니다

TCP session이 끊겼다고 outstanding order가 자동 취소됐다고 가정하지 않는다. Venue의 cancel-on-disconnect 규칙과 session state를 확인하고, 불확실하면 query/drop copy로 reconcile한다.

### 한 문장 기억법

> 요청을 보낸 상태와 거래소가 결과를 확정한 상태를 분리하고, pending 중에도 authoritative fill을 적용한다.

### 30초 면접 답변

> Cancel request 전송은 cancel 완료가 아니므로 executable exposure를 유지하고 fill을 계속 반영합니다. CancelPending은 fill status와 직교하게 모델링하며 full fill, cancel ack, reject를 venue event ordering대로 적용합니다. Disconnect나 local timeout도 terminal cancel로 간주하지 않습니다.

## 5.4 Execution report는 identity와 sequence로 한 번만 반영한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Execution report | 주문 접수, 체결, 취소, reject와 correction 결과를 알려주는 message |
| Fill | 주문 수량 일부 또는 전부가 실제 거래로 체결된 event |
| Execution ID | Fill이나 execution event를 식별하기 위해 venue가 부여하는 identifier |
| Deduplication | 이미 처리한 event identity를 기록해 같은 effect를 다시 적용하지 않는 처리 |
| Idempotent | 같은 event를 여러 번 입력해도 최종 effect가 한 번 입력한 것과 같은 성질 |
| Out-of-order | Message가 business·sequence상 기대한 순서와 다르게 도착한 상태 |
| Trade bust·cancel | 이전 체결 effect를 취소하는 correction event |
| Trade correction | 이전 체결의 가격·수량 등을 정정하는 event |
| Position | Instrument별 순매수·순매도 수량 상태 |
| Reducer | 현재 state와 event를 입력받아 다음 state를 만드는 함수·구성 요소 |
| Transactional boundary | 여러 state 변경이 모두 적용되거나 모두 적용되지 않도록 묶는 경계 |
| Ingest sequence | 여러 source event를 system에 받아들인 전체 순서를 기록한 local sequence |
Network retry와 session recovery 때문에 같은 execution report가 다시 올 수 있다. 도착 순서도 business event 순서와 다를 수 있다.

```text
실제 fill F1: +10 position
report F1 도착: +10 적용
session resend로 F1 다시 도착
중복을 또 적용하면 position +20  ← 오류
```

### Dedup key

정확한 key는 venue protocol에 따라 다르지만 일반적으로 다음을 조합한다.

- Session 또는 stream identity
- Venue execution ID
- Message sequence
- Order/client order identity
- Correction/bust가 참조하는 original execution

단순히 payload가 같다고 duplicate로 보면 안 된다. 같은 가격과 수량의 서로 다른 fill 두 개가 있을 수 있다.

### Out-of-order 처리

Per-stream sequence gap을 발견하면 뒤 report를 바로 business state에 적용할지 buffer할지 protocol recovery 규칙으로 정한다. 여러 stream 사이에는 sequence number만으로 total order가 생기지 않는다. System 전체 replay 순서가 필요하면 ingest sequence나 명시적 merge tie-break를 기록한다.

### Position은 event의 합이다

```text
position
= initial/snapshot position
+ unique fills
- busted fills
± corrections
```

Dedup record와 state update를 따로 commit하면 crash 사이에 한쪽만 남을 수 있다. Event log/reducer 또는 transactional boundary로 함께 다룬다.

### 한 문장 기억법

> 체결을 한 번만 반영하려면 값이 비슷한지를 보지 말고 venue가 부여한 event identity와 correction 관계를 기록한다.

### 30초 면접 답변

> Execution report는 session sequence와 venue execution identity로 deduplicate하고 protocol ordering을 검증한 뒤 reducer에 적용합니다. 동일 price/qty는 duplicate 근거가 아니며 bust와 correction은 original execution을 참조하는 별도 event입니다. Position은 snapshot baseline과 unique event 효과의 합으로 검증합니다.

## 5.5 Pre-trade risk는 주문 전송 전에 exposure를 예약한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Pre-trade risk | 주문을 venue로 보내기 전에 limit 위반 여부를 검사하는 risk control |
| Exposure | 주문과 position 때문에 손실·시장 변동에 노출될 수 있는 수량·금액 |
| Open-order exposure | 아직 체결·취소가 확정되지 않아 앞으로 실행될 수 있는 주문 위험 |
| Limit | 수량, notional, position 등에서 허용한 최대 위험 값 |
| Reservation | 주문이 전송되기 전에 해당 주문의 예상 exposure를 risk state에 먼저 반영하는 처리 |
| Atomic transition | Check와 state update 사이에 다른 operation이 끼어 잘못된 결과를 만들지 않도록 하나의 일관된 변경으로 처리하는 것 |
| Replace | 기존 주문의 가격이나 수량을 변경하는 요청 |
| Replace-up·down | Exposure를 늘리는 replace와 줄이는 replace |
| Delta | 기존 값과 새 값의 차이 |
| Fail closed | 결과가 불확실할 때 신규 위험을 허용하지 않는 보수적 동작 |
| Reconciliation | Local risk·order state를 venue의 확인 정보와 비교해 다시 맞추는 과정 |
현재 open exposure가 90이고 limit이 100이라고 하자. 두 thread가 동시에 수량 10 주문을 검사한다.

```text
Thread A: 90 + 10 <= 100 확인
Thread B: 90 + 10 <= 100 확인
둘 다 전송
결과 exposure = 110
```

Check와 reservation이 하나의 일관된 state transition이어야 한다. 가장 단순한 방법은 risk state의 single writer가 순서대로 처리하는 것이다.

### 언제 예약하고 해제하는가?

```text
New order:
risk check → exposure reserve → wire send

Reject:
venue가 주문을 받지 않았다고 확정 → reservation release

Fill:
open-order exposure 감소 + position/executed exposure 갱신

Cancel request:
아직 체결 가능 → reservation 유지

Cancel ack:
취소가 확정된 remaining exposure release
```

### Replace가 어려운 이유

수량 10을 15로 늘리는 replace는 추가 5를 wire send 전에 예약해야 한다. 그렇지 않으면 replace pending 여러 개가 limit을 넘을 수 있다.

수량 15를 10으로 줄이는 replace는 요청을 보냈다고 5를 즉시 풀면 안 된다. Venue가 reject하거나 그 사이 기존 15 기준으로 fill할 수 있다. Venue acceptance 뒤에만 줄어든 exposure를 확정 해제한다.

### Unknown 상태

Timeout이나 disconnect로 venue가 주문을 받았는지 모르면 exposure를 낙관적으로 풀지 않는다. Unknown으로 보수적으로 유지하고 query/drop copy/recovery로 확인한다.

### 한 문장 기억법

> Risk는 “보냈을 것 같은 수량”이 아니라 아직 거래소에서 실행될 수 있는 최대 exposure를 보수적으로 예약한다.

### 30초 면접 답변

> Risk check와 exposure reservation은 주문 전송 전 하나의 일관된 transition으로 수행합니다. Cancel pending과 unknown order는 여전히 executable할 수 있어 예약을 유지합니다. Replace-up delta는 전송 전에 추가 예약하고, replace-down 감소분은 venue acceptance 뒤에만 해제합니다.

## 5.6 Trading 숫자는 단위를 type과 integer에 담는다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Binary floating point | 값을 2진수 기반 가수와 지수로 표현하는 실수 형식. 일부 decimal 값을 정확히 표현하지 못한다 |
| Fixed-point | Integer 값과 소수점 위치를 정하는 scale을 함께 사용하는 수 표현 |
| Scale | Integer의 어느 자리가 실제 소수점 위치인지 정하는 값 |
| Tick size | Venue가 허용하는 최소 가격 변화 단위 |
| Lot size | 주문 수량이 따라야 하는 최소 거래 단위 |
| Strong type | 같은 integer representation을 사용해도 Price, Quantity처럼 의미가 다른 값을 type으로 구분하는 설계 |
| Currency | 금액이 어떤 통화 단위인지 나타내는 정보 |
| Overflow | 계산 결과가 integer type의 표현 범위를 벗어나는 상황 |
| Intermediate | 여러 단계 계산 중 최종 결과 전에 사용하는 임시 값과 type |
| Rounding | 표현 가능한 단위로 값을 맞추기 위해 올림·내림·반올림하는 규칙 |
| Deterministic numeric | 같은 입력과 규칙에서 platform·실행마다 같은 결과를 내도록 정의한 수치 처리 |
가격 `12.34`를 binary floating point로 표현하면 정확히 같은 decimal 값이 아닐 수 있다. 비교와 반올림이 venue tick rule과 어긋날 수 있다.

```text
표시 가격 12.34
scale = 2 decimal digits
내부 integer = 1234
```

Fixed-point는 integer와 scale을 함께 사용한다.

```cpp
struct PriceTicks {
    std::int64_t value;
};

struct QuantityLots {
    std::int64_t value;
};
```

Strong type을 쓰면 가격과 수량을 실수로 더하거나 서로 다른 scale/currency를 섞는 오류를 줄일 수 있다. `MoneyMinor`라는 이름만으로 currency가 정해지는 것은 아니므로 currency와 scale metadata도 필요하다.

### Integer도 자동으로 안전하지 않다

`price * quantity`는 각 operand가 64-bit 범위여도 곱셈에서 overflow할 수 있다. 계산 전 범위를 검사하거나 더 넓은 intermediate를 사용하고 결과 범위를 확인한다.

Division과 rounding rule도 명시해야 한다.

- Tick grid에 맞는가?
- Buy와 sell에서 rounding 방향이 같은가?
- Fee와 PnL의 scale은 무엇인가?
- Negative 값과 overflow를 어떻게 처리하는가?

### 한 문장 기억법

> Fixed-point의 목적은 단지 빠른 integer 연산이 아니라, 가격 단위와 반올림 규칙을 결정적으로 만드는 것이다.

### 30초 면접 답변

> Trading price와 quantity는 decimal tick과 lot 규칙이 중요하므로 integer fixed-point와 명시적 scale을 사용합니다. Strong type으로 단위 혼합을 막고 multiply/add의 overflow와 rounding direction을 검사합니다. Fixed-point가 항상 빠르다는 이유보다 deterministic representation이 핵심입니다.

## 5.7 Bounded queue가 가득 차면 data 의미에 따라 다르게 대응한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Bounded queue | 최대 item 수가 고정된 queue |
| Producer·consumer | Queue에 item을 넣는 실행 주체와 꺼내 처리하는 실행 주체 |
| Queue full | Producer가 새 item을 넣으려 하지만 빈 slot이 없는 상태 |
| Overload | 입력률이 처리 능력을 일정 시간 이상 넘어 backlog가 증가하는 상태 |
| Drop newest·oldest | 새 item 또는 가장 오래된 item을 버리는 overload policy |
| Coalesce | 여러 update를 의미가 보존되는 하나의 최신 state update로 합치는 처리 |
| Priority lane | 중요 event를 별도 queue·capacity로 전달하는 경로 |
| Reserved capacity | 특정 message 종류를 위해 다른 traffic이 사용할 수 없게 남겨 둔 slot |
| High-watermark | Queue 사용량이 위험 수준에 가까워졌음을 알리는 임계값 |
| Oldest age | Queue에서 가장 오래 기다린 item의 대기 시간 |
| Kill switch | 신규 주문·위험을 즉시 차단하는 safety mechanism |
모든 queue는 유한하다. “절대 가득 차지 않을 만큼 크게 만든다”는 정책은 지속 overload 앞에서 실패한다.

### Market data와 execution은 같은 방식으로 버릴 수 없다

일부 absolute market data는 최신 값으로 coalesce할 수 있다. 하지만 order-level delta 하나를 버리면 이후 book 전체가 틀릴 수 있어 invalidation과 recovery가 필요하다.

Execution report를 조용히 버리면 position과 risk가 실제 거래소와 달라진다. 중요한 state event는 reserved capacity, durable handoff, fail-safe와 신규 주문 차단 같은 별도 정책이 필요하다.

### 가능한 정책

| 정책 | 장점 | 위험 |
| --- | --- | --- |
| 짧게 spin/retry | 짧은 burst 흡수 | Consumer 정지 시 CPU 소모 |
| Block | loss 방지 가능 | Backpressure가 NIC까지 전파 |
| Drop | 진행 유지 | Protocol state가 무효가 될 수 있음 |
| Coalesce | 최신 상태만 필요할 때 효율적 | Delta event에는 부적합 |
| 별도 priority lane | 중요 event 자원 보존 | Merge ordering을 다시 맞춰야 함 |
| Fail closed | 신규 위험 차단 | 서비스 기능 축소 |

Priority lane을 사용해도 execution reducer에서 venue sequence와 causal ordering을 보존해야 한다. Kill switch는 일반 event의 순서를 실수로 추월하는 것이 아니라, 신규 위험을 즉시 막는 별도 safety action으로 명확히 정의한다.

Queue depth만 보지 말고 oldest message age를 본다. Queue가 크면 drop은 늦어지지만 오래된 시장 상태를 처리하는 문제가 생긴다.

### 한 문장 기억법

> Queue full은 예외가 아니라 설계된 overload 상태이며, 무엇을 버려도 되는지는 message의 business 의미가 결정한다.

### 30초 면접 답변

> Bounded queue full에는 명시적 overload policy가 필요합니다. Incremental market data drop은 book을 invalid화하고 recovery해야 하며 execution과 risk event는 조용히 버리지 않고 reserved path나 fail-closed로 보호합니다. Depth, oldest age, full count와 drop을 함께 관측합니다.

## 5.8 정확히 한 번 전송보다, 중복되어도 한 번만 효과를 내게 한다

### 용어부터 풀기

| 용어 | 뜻 |
| --- | --- |
| Exactly-once effect | Retry와 중복 delivery가 있어도 business state에는 effect가 한 번만 반영되는 성질 |
| At-least-once delivery | Message가 누락되지 않도록 재전송하지만 duplicate가 생길 수 있는 전달 방식 |
| Intent | System이 외부에 수행하려고 결정한 주문·operation |
| Journal·event log | Intent와 event를 복구 가능하도록 순서대로 기록하는 durable log |
| WAL | Write-Ahead Log. 외부 effect나 state commit 전에 intent·변경 기록을 먼저 durable하게 쓰는 방식 |
| Durability | Process나 host 장애 뒤에도 필요한 기록이 남아 복구에 사용되는 성질 |
| Idempotency key | 같은 logical request의 retry를 venue나 receiver가 식별하도록 제공하는 stable ID |
| Session recovery | Message sequence와 resend를 사용해 disconnect 전후 session message를 복구하는 과정 |
| Outstanding order | Venue에서 아직 체결·취소·종료가 확정되지 않은 주문 |
| Deterministic replay | 같은 초기 state와 기록된 event 순서를 다시 적용해 같은 local state를 만드는 과정 |
| External truth | Local 추정과 비교할 venue query, drop copy, clearing record 같은 외부 확인 정보 |
| New-order gate | Recovery가 끝날 때까지 신규 주문 전송을 막는 제어 지점 |
주문을 network로 보낸 직후 process가 죽었다고 하자.

```text
가능성 A: packet이 NIC에도 못 감
가능성 B: 거래소에 도착했지만 ack가 오기 전 죽음
가능성 C: ack가 host에 왔지만 process가 기록하기 전 죽음
```

Restart한 program은 local memory만 보고 어느 경우인지 알 수 없다. TCP ACK도 bytes의 전송 계층 전달을 뜻할 뿐 business acceptance를 뜻하지 않는다.

### 왜 end-to-end exactly-once가 어려운가?

송신자가 결과를 확인하기 전에 죽으면 “처음 요청이 처리됐는가?”라는 불확실성이 남는다. 무조건 재전송하면 duplicate가 될 수 있고, 재전송하지 않으면 주문을 잃을 수 있다.

실무에서는 다음을 조합한다.

- Stable client order ID 또는 venue idempotency key
- Outbound intent를 전송 전 journal에 기록
- Venue session sequence와 recovery
- Execution ID deduplication
- Outstanding order query와 drop copy
- Unknown 상태와 신규 주문 gate
- Deterministic replay와 invariant 검사

### WAL도 혼자 해결하지 못한다

Write-ahead log를 먼저 쓴 뒤 전송해도 전송 후 ack 전 crash의 불확실성은 남는다. WAL은 “무엇을 하려고 했는가”를 복구하게 해주며, 외부 효과 확인에는 venue identity와 reconciliation이 필요하다.

### Restart 순서

```text
1. Durable log와 snapshot 로드
2. Event replay로 local state 복구
3. Session sequence 복구
4. Venue의 outstanding order/execution과 reconcile
5. Position과 risk invariant 확인
6. Unknown 해결 또는 보수적 제한
7. 신규 주문 gate 열기
```

Replay가 같은 결과를 만든다고 그 결과가 현실과 정확하다는 뜻도 아니다. 결정적인 bug는 똑같이 재현된다. Venue라는 외부 truth와 비교한다.

### 한 문장 기억법

> Network 경계에서는 “한 번만 보냈다”보다, stable identity로 중복을 알아보고 외부 상태와 다시 맞출 수 있는지가 중요하다.

### 30초 면접 답변

> 주문 전송 후 ack 전 crash에서는 venue 처리 여부를 local state만으로 알 수 없어 end-to-end exactly-once를 단정하기 어렵습니다. Intent journal, stable order ID, session recovery와 execution dedup을 사용하고 restart 시 venue query/drop copy로 reconcile한 뒤 risk invariant를 확인하고 신규 주문을 허용합니다.

# 6. 이 해설판을 실제 면접 공부에 사용하는 방법

## 6.1 첫 번째 읽기: 그림과 한 문장만 본다

처음에는 code와 예외 조건을 모두 외우지 않는다.

1. 각 절의 ASCII 흐름을 직접 종이에 다시 그린다.
2. **한 문장 기억법**을 자기 말로 바꿔 말한다.
3. 모르는 단어에 표시만 하고 끝까지 읽는다.

전체 길이 먼저 보이면 뒤의 단어가 들어갈 자리가 생긴다.

## 6.2 두 번째 읽기: 경계를 말한다

면접에서 깊이를 만드는 것은 어려운 명사보다 개념 경계를 정확히 나누는 능력이다.

다음 쌍을 설명해본다.

- Execute / retire / 다른 코어 visibility
- Cache line / C++ object
- TLB miss / page fault
- Coherence / ISA consistency / C++ memory model
- Atomicity / ordering
- Storage / object lifetime
- Lock-free step progress / wall-clock deadline
- System call / context switch
- Doorbell write / TX completion / wire timestamp
- Cancel request / cancel acknowledgement
- Message delivery / business acceptance
- Replay 결과 / venue truth

각 쌍을 “A는 무엇이고, B는 무엇이며, 둘이 왜 동시에 일어나지 않을 수 있는가?”로 말하면 된다.

## 6.3 세 번째 읽기: 30초와 3분 답변을 만든다

각 주제마다 두 버전을 준비한다.

```text
30초:
결론 + 핵심 메커니즘 + 실무 영향

3분:
30초 답변
+ 내부 단계
+ 구현별 차이
+ 실패 시나리오
+ 측정·검증 방법
```

처음부터 3분 답을 암기하면 한 문장을 잊었을 때 전체가 무너진다. 먼저 30초 구조를 자기 언어로 만들고 세부 내용을 가지처럼 붙인다.

## 6.4 추천 7일 순서

| 날짜 | 해설판 범위 | 해야 할 출력 |
| --- | --- | --- |
| 1일 | 0장 전체 시스템 | Packet-to-fill 경로를 안 보고 그리기 |
| 2일 | 1장 CPU와 memory | Load/store와 cache-line 이동 설명 |
| 3일 | 2.1~2.5 C++ lifetime | RAII, ownership, pool lifetime 설명 |
| 4일 | 2.6~2.12 concurrency | Happens-before와 SPSC를 그림으로 증명 |
| 5일 | 3장 Linux와 network | NIC에서 `recv()`까지 경로 그리기 |
| 6일 | 4장 측정 | 잘못된 benchmark 사례 세 개 찾기 |
| 7일 | 5장 trading | Gap, cancel/fill, crash recovery 상태 그리기 |

그 다음 [[Low Latency Trading] 면접 심화 질문과 답변](/posts/low-latency-trading-interview-deep-dive/)의 해당 질문을 읽고, 해설판을 보지 않은 채 30초 답변을 녹음한다.

# 7. 심화판 56문항과 해설판의 연결표

## 7.1 CPU·Cache·Memory·Compiler 15문항

| 심화판 질문 | 먼저 읽을 해설 |
| --- | --- |
| CPU Q1 Out-of-order 실행 | 1.1 CPU pipeline과 retire |
| CPU Q2 Store retire와 visibility | 1.2 Store buffer와 publication |
| CPU Q3 L1·L2·LLC와 inclusion | 1.3 Cache hierarchy와 line |
| CPU Q4 Cache miss의 fill 경로 | 1.4 Cache miss와 pointer chasing |
| CPU Q5 다른 core line에 store | 1.5 Coherence ownership |
| CPU Q6 False sharing | 1.5 False sharing |
| CPU Q7 Store forwarding와 disambiguation | 1.2 Store 단계, 1.4 dependency |
| CPU Q8 Branch prediction | 1.6 Branch와 speculation |
| CPU Q9 TLB·page fault·huge page | 1.7 Virtual memory |
| CPU Q10 NUMA | 1.8 NUMA topology |
| CPU Q11 NIC DMA와 cache | 1.8 DMA, 3.4 descriptor ownership |
| CPU Q12 Coherence·consistency·C++ | 1.9 세 규칙 층 |
| CPU Q13 Volatile과 barrier | 1.9 Compiler와 hardware 경계 |
| CPU Q14 Undefined behavior | 1.9 UB와 최적화 |
| CPU Q15 ABI와 calling convention | 1.9 ABI와 compiler output |

## 7.2 Modern C++·동시성 18문항

| 심화판 질문 | 먼저 읽을 해설 |
| --- | --- |
| C++ Q1 RAII | 2.1 Ownership과 cleanup |
| C++ Q2 생성자·소멸자 exception | 2.1 부분 생성, 2.2 destructor |
| C++ Q3 Exception guarantee | 2.2 실패 뒤 상태 |
| C++ Q4 Rule of Zero/Five와 move | 2.3 Special member와 ownership |
| C++ Q5 `unique_ptr`·`shared_ptr` | 2.3 Smart pointer |
| C++ Q6 Storage와 object lifetime | 2.4 Pool slot의 lifetime |
| C++ Q7 Allocator tail latency | 2.5 Preallocation과 exhaustion |
| C++ Q8 Data race와 happens-before | 2.6 Happens-before 다리 |
| C++ Q9 Atomicity·coherence·ordering | 2.6 Atomicity와 ordering |
| C++ Q10 Relaxed와 release/acquire | 2.7 Memory order |
| C++ Q11 Seq_cst와 fence | 2.7 Total order와 규칙 층 |
| C++ Q12 Compare-and-exchange | 2.8 CAS loop |
| C++ Q13 Progress guarantee | 2.9 Lock-free와 wall clock |
| C++ Q14 ABA | 2.10 값과 history |
| C++ Q15 Hazard·epoch·RCU | 2.10 Safe reclamation |
| C++ Q16 SPSC 증명 | 2.11 Index ownership과 publication |
| C++ Q17 MPSC·MPMC | 2.12 Reservation과 publication |
| C++ Q18 Mutex·futex·spinlock | 2.12 대기 전략 |

## 7.3 Linux·Network·측정·Trading 23문항

| 심화판 질문 | 먼저 읽을 해설 |
| --- | --- |
| System Q1 System call과 context switch | 3.1 Mode 전환과 task 교체 |
| System Q2 CPU affinity와 jitter | 3.2 허용 CPU와 isolation |
| System Q3 `mlockall`과 page fault | 1.7 Virtual memory, 3.2 pre-touch |
| System Q4 IRQ·softirq·NAPI | 3.3 Packet path |
| System Q5 NIC에서 `recv()`까지 | 3.3 전체 receive 경로 |
| System Q6 RX/TX descriptor ownership | 3.4 Buffer handoff |
| System Q7 RSS·RPS·RFS·XPS | 3.5 Packet steering |
| System Q8 Offload와 moderation | 3.6 Batching trade-off |
| System Q9 UDP loss 진단 | 3.7 경계별 counter |
| System Q10 Kernel bypass | 3.8 전용 data path의 대가 |
| System Q11 평균과 percentile | 4.1 Latency distribution |
| System Q12 Coordinated omission | 4.2 Load generator 오류 |
| System Q13 TSC·clock·PTP | 4.3 Clock domain과 경계 |
| System Q14 Microbenchmark | 4.4 통제된 실험 |
| System Q15 단계 p99와 PMU | 4.5 End-to-end correlation |
| System Q16 Incremental order book | 5.1 Sequence와 invalidation |
| System Q17 Snapshot merge | 5.2 Recovery handoff |
| System Q18 Cancel과 fill race | 5.3 직교 상태 머신 |
| System Q19 Duplicate execution | 5.4 Event identity와 position |
| System Q20 Pre-trade risk | 5.5 Exposure reservation |
| System Q21 Fixed-point | 5.6 단위와 overflow |
| System Q22 Bounded queue full | 5.7 Message별 overload 정책 |
| System Q23 Crash와 exactly-once | 5.8 Identity와 reconciliation |

# 8. 반복해서 확인할 핵심 용어

세부 용어는 각 절의 `용어부터 풀기`에서 처음 등장한 문맥과 함께 설명했다. 아래 표는 여러 분야에서 반복해서 사용하는 공통 용어만 다시 모은 것이다.

| 용어 | 이 글에서의 뜻 |
| --- | --- |
| Core | 실제 instruction을 실행하는 CPU의 처리 단위 |
| Hardware thread | SMT로 한 core가 제공할 수 있는 logical CPU context |
| Cache line | Cache와 coherence가 data를 옮기고 소유권을 관리하는 대표 단위 |
| Tail latency | 분포의 느린 쪽에 있는 드문 긴 latency |
| Ownership | 지금 어떤 thread/core/device가 상태를 변경할 권한이 있는가 |
| Publication | 완성한 data를 다른 실행 주체가 안전하게 읽도록 공개하는 일 |
| Atomic | 한 atomic object의 연산이 중간 상태로 관찰되지 않게 하는 언어 기능 |
| Ordering | 여러 memory operation의 관찰 순서에 대한 규칙 |
| Invariant | 어떤 event 전후에도 반드시 지켜져야 하는 상태 조건 |
| State machine | Event에 따라 허용된 상태 전이를 명시한 모델 |
| Sequence | Event의 누락, 중복과 순서를 판단하는 번호·규칙 |
| Backpressure | Consumer가 느릴 때 그 영향이 upstream으로 전달되는 현상 |
| Reconciliation | Local state와 venue 같은 외부 truth를 비교해 다시 맞추는 과정 |
| Replay | 기록된 event를 정해진 순서로 다시 적용해 상태를 재구성하는 과정 |

# 9. 완료 기준

다음 질문에 해설판을 보지 않고 답할 수 있으면 심화판으로 넘어간다.

- CPU가 instruction을 순서와 다르게 실행하면서 결과는 어떻게 순서대로 보이는가?
- Store retire와 다른 core visibility는 왜 다른가?
- Cache line 하나가 두 core 사이를 왕복하는 과정을 그릴 수 있는가?
- TLB miss와 page fault를 한 문장으로 구분할 수 있는가?
- C++ data race를 coherence로 설명하면 왜 부족한가?
- Payload publication의 happens-before edge를 그릴 수 있는가?
- SPSC에서 producer와 consumer가 각각 무엇을 소유하는가?
- System call과 context switch가 동시에 일어나지 않을 수 있는 예를 들 수 있는가?
- Packet이 NIC에서 socket queue까지 지나는 queue를 말할 수 있는가?
- 평균만 보고 저지연이라고 결론 내리면 왜 안 되는가?
- Sequence gap 뒤 book을 계속 사용하면 왜 안 되는가?
- Cancel pending 중 fill을 어떻게 처리해야 하는가?
- Replace-up과 replace-down의 risk reservation 시점이 왜 다른가?
- Crash 뒤 주문 처리 여부가 unknown일 때 무엇과 reconcile해야 하는가?

# 10. 다음에 읽을 글

## 전체 학습 경로

- [[Low Latency Trading] 초저지연 트레이딩 시스템 BFS 학습 로드맵](/posts/low-latency-trading-bfs-roadmap/)
- [[Low Latency Trading] 초저지연 트레이딩 시스템 전체 구조](/posts/low-latency-trading-system-overview/)
- [[Low Latency Trading] 면접 심화 질문과 답변](/posts/low-latency-trading-interview-deep-dive/)

## CPU·Memory·동시성

- [[Low Latency Trading] CPU Pipeline·Out-of-Order·Branch Prediction과 ABI](/posts/low-latency-cpu-pipeline-branch-abi/)
- [[Low Latency Trading] Cache-Friendly Data Layout과 Memory Pool](/posts/low-latency-cache-layout-memory-pool/)
- [[Low Latency Trading] CPU Interconnect와 Uncore](/posts/low-latency-cpu-interconnect-uncore/)
- [[C언어] Atomic Operation 3. Atomic은 실제로 어떻게 동작하는가](/posts/c-atomic-operation-3/)
- [[C언어] Atomic Operation 4. Memory Order 이해하기](/posts/c-atomic-operation-4/)
- [[Low Latency Trading] SPSC Ring Buffer와 Thread-per-Core 설계](/posts/low-latency-spsc-thread-per-core/)

## Linux·Network·측정

- [[Low Latency Trading] Linux 실행 환경과 CPU 격리](/posts/low-latency-linux-execution/)
- [[Low Latency Trading] Market Data가 NIC에서 Application까지 오는 길](/posts/low-latency-network-path/)
- [[Low Latency Trading] Latency를 올바르게 측정하는 방법](/posts/low-latency-measurement/)

## Trading state와 recovery

- [[Low Latency Trading] Market Data Sequencing과 Recovery](/posts/low-latency-market-data-sequencing/)
- [[Low Latency Trading] Order Book Engine](/posts/low-latency-order-book-engine/)
- [[Low Latency Trading] Order Gateway와 FIX·Binary Session Recovery](/posts/low-latency-order-gateway-session/)
- [[Low Latency Trading] Fixed-Point Trading Numerics와 O(1) Pre-Trade Risk](/posts/low-latency-trading-numerics-risk/)
- [[Low Latency Trading] Binary Event Log·Deterministic Replay와 Fault Injection](/posts/low-latency-event-log-replay/)

# 참고 자료

- [ISO C++ Working Draft — Object Lifetime](https://eel.is/c++draft/basic.life)
- [ISO C++ Working Draft — Atomics Order](https://eel.is/c++draft/atomics.order)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [Intel® 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html)
- [AMD Software Optimization Guide for AMD EPYC Processors](https://docs.amd.com/v/u/en-US/56305)
- [Linux Kernel — NAPI](https://docs.kernel.org/networking/napi.html)
- [Linux Kernel — Scaling in the Linux Networking Stack](https://docs.kernel.org/networking/scaling.html)
- [Linux Kernel — Dynamic DMA Mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)
- [Linux Kernel — CPU Isolation](https://docs.kernel.org/admin-guide/cpu-isolation.html)
- [Linux Kernel — Timestamping](https://docs.kernel.org/networking/timestamping.html)
- [FIX Trading Community — FIX Session Layer](https://www.fixtrading.org/standards/fix-session-layer-online/)
- [FIX Trading Community — Order State Change Matrices](https://www.fixtrading.org/online-specification/order-state-changes/)
- [Nasdaq — TotalView-ITCH 5.0 Specification](https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf)
- [Nasdaq — OUCH 5.0 Specification](https://nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/Ouch5.0.pdf)
