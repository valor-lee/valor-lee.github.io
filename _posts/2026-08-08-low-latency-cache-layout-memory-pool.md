---
title: '[Low Latency Trading] Cache-Friendly Data Layout과 Memory Pool'
date: 2026-08-08 00:20:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    cache locality,
    data layout,
    memory pool,
    C++ allocator
  ]
---

# 개요

초저지연 Trading System에서는 같은 Big-O 복잡도를 가진 두 구현도 전혀 다른 latency를 보일 수 있다. CPU가 실제로 읽는 byte의 위치, 다음 주소를 알 수 있는 시점과 cache line을 여러 core가 주고받는 횟수가 다르기 때문이다.

```text
Message 처리 시간
  = 계산 비용
  + Data를 가까운 곳으로 가져오는 비용
  + Allocation과 Reclamation 비용
  + Core 사이에서 Ownership을 이동하는 비용
```

Heap Allocation을 없애거나 `alignas(64)`를 붙이는 것만으로 문제가 끝나지는 않는다. 사용하지 않는 field까지 cache에 올리면 working set이 커지고, 잘못 나눈 pool은 capacity 고갈이나 memory 낭비를 만든다.

이 글에서는 접근 패턴에서 Data Layout을 결정하고 Object Lifetime을 보존하는 bounded Memory Pool을 설계한 뒤, correctness와 hardware counter로 효과를 검증한다.

# 선수 글과 BFS 위치

엄격한 선수 글은 다음 두 글이다.

- [[CPU] Memory Access와 Cache Hierarchy](/posts/cpu-memory-cache-hierarchy/)
- [[Low Latency Trading] Hot Path를 위한 Modern C++ 설계 원칙](/posts/low-latency-cpp-hot-path/)

CPU가 이 access stream을 실제로 실행하는 과정은 다음 글을 먼저 읽는다.

- [[Low Latency Trading] CPU Pipeline·Out-of-Order·Branch Prediction과 ABI](/posts/low-latency-cpu-pipeline-branch-abi/)

다음 글은 같은 Memory Path를 더 깊게 이해하기 위한 보강 자료다.

- [[Memory] Virtual Memory와 TLB](/posts/virtual-memory-tlb/)
- [[Memory] NUMA 구조와 Local·Remote Memory](/posts/numa-memory-architecture/)

이 글의 BFS 위치는 **Level 2-H: CPU Microarchitecture와 Memory Layout**이다. 다음 단계에서는 이 원칙을 [SPSC Queue](/posts/low-latency-spsc-thread-per-core/), [Feed Handler](/posts/low-latency-feed-handler/), [Order Book](/posts/low-latency-order-book-engine/)과 [O(1) Risk Gate](/posts/low-latency-trading-numerics-risk/)에 적용한다.

# 1. Source Code가 아니라 Memory Access Stream을 본다

다음 두 질문은 서로 다르다.

```text
논리적 질문: Best Ask의 Price와 Quantity는 무엇인가?
물리적 질문: 그 값을 얻기 위해 몇 cache line과 pointer를 읽는가?
```

Compiler는 runtime의 address, data dependency와 공유 방식까지 없애주지는 못한다. Hot Path를 검토할 때 다음 항목을 적는다.

- 한 Message에서 실제로 읽고 쓰는 field
- 접근 순서와 stride
- 동시에 살아 있는 element 수
- Data가 연속인지 node와 pointer로 흩어졌는지
- 각 field의 writer와 reader core
- 삽입, 삭제와 재사용 빈도
- 최대 capacity와 초과 시 정책

자료구조 이름보다 이 access stream이 cache behavior를 더 직접적으로 결정한다.

# 2. Working Set을 계산한다

Working set은 전체 allocation이 아니라 특정 시간 구간에 반복해서 접근하여 cache와 TLB에 머물러야 하는 data 집합이다.

```text
활성 Order 100,000개 × Hot Record 32 byte ≈ 3.05 MiB
활성 Order 100,000개 × 전체 Record 256 byte ≈ 24.41 MiB
```

두 값에는 container index, allocator metadata, queue와 다른 feed의 state가 빠져 있으므로 다음 항목까지 합쳐 추정한다.

| 항목 | 확인할 내용 |
| --- | --- |
| Domain Record | `sizeof(T)`, alignment와 padding |
| Index | Bucket, tree node, sparse slot와 load factor |
| Queue | Element, sequence marker와 producer/consumer cursor |
| Pool | Free-list metadata, bitmap와 unused capacity |
| Thread State | Counter, scratch buffer와 per-thread cache |
| Page Mapping | Page size와 TLB가 다루는 page 수 |

Working set이 cache 경계를 넘는 지점에서 latency가 불연속적으로 달라질 수 있으므로 정확한 크기와 공유 구조를 대상 CPU에서 확인한다.

# 3. Cache Line은 전송과 Coherence의 단위다

CPU는 일반적으로 cache line 단위로 data를 이동한다. 인접 data가 곧 필요하면 유리하지만 필요 없는 byte가 대부분이면 bandwidth와 capacity를 낭비한다.

많은 x86-64 CPU에서 64-byte line을 볼 수 있지만 보편적 상수로 가정하지 말고 Target CPU의 manual, CPUID 또는 운영체제 topology를 확인한다.

`std::hardware_destructive_interference_size`는 구현의 권장 간격이지만 모든 hardware의 절대적 보장은 아니다. 배포 대상과 ABI를 고정하고 실제 address와 counter로 검증한다.

# 4. AoS와 SoA는 접근 패턴으로 선택한다

Order Book level의 Price(P), Quantity(Q), Count(C), Venue(V), Time(T)를 표현한다고 하자.
AoS는 한 element의 field를 함께 두고 SoA는 같은 field를 모은다.

```text
AoS: [P Q C V T][P Q C V T][P Q C V T]
SoA: [P P P] [Q Q Q] [C C C] [V V V] [T T T]
```

| 접근 | 유력한 후보 | 이유 |
| --- | --- | --- |
| 한 level의 모든 field 처리 | AoS | 필요한 값이 가까이 있음 |
| 모든 price 또는 quantity만 scan | SoA | 사용하지 않는 field를 덜 가져옴 |
| 한 field에 SIMD 적용 | SoA | 연속 vector load가 쉬움 |
| record 단위 전달과 복사 | AoS | API와 ownership이 단순함 |
| 일부 field만 매우 자주 사용 | Hybrid | hot group만 함께 배치 가능 |

한 update에서 모든 배열을 읽으면 SoA의 stream과 cache line 수가 오히려 늘 수 있다. 추가·삭제 시에는 배열의 size와 index가 같은 invariant도 유지해야 한다.

# 5. Hot-Cold Split으로 자주 쓰는 byte를 모은다

Order state에는 매 Message마다 쓰는 field와 audit 때만 읽는 field가 섞일 수 있다.

```text
Hot : Order ID, State, Leaves Quantity, Price, Side
Cold: Text Symbol, Original Request, Timestamps, Reject Detail, Audit Metadata
```

Hot-Cold Split은 Hot Record를 작게 만들지만 Cold Record용 pointer나 index와 두 record의 lifetime 관리가 추가된다.

분리 기준은 profile이다. 오류용 field도 정상 경로의 metric이 계속 읽는다면 실제로는 cold가 아니다.

# 6. Flat Structure와 Node-Based Structure

연속 array나 flat container는 예측 가능한 stride로 순회하므로 hardware prefetcher가 다루기 쉽다.

```text
Flat: [0][1][2][3][4][5]
Node: [Node] --pointer--> [Node] --pointer--> [Node]
```

Node의 다음 주소를 현재 node에서 읽어야 하는 dependency를 pointer chasing이라고 한다. Node가 여러 page에 흩어지면 cache와 TLB miss 가능성이 커진다.

그렇다고 flat structure가 항상 정답은 아니다.

- Sorted vector의 중간 삽입은 element 이동이 필요하다.
- Dense price array는 price domain이 넓으면 memory를 낭비한다.
- Tree는 안정적인 iterator나 순서가 있는 동적 삽입에 유리할 수 있다.
- Hash table은 load factor, collision과 rehash 정책에 따라 tail이 달라진다.

Order Book이라면 tick 범위, update 분포, best-price 탐색과 snapshot scan을 같은 workload로 비교하며 평균 복잡도 표 하나로 선택하지 않는다.

# 7. Alignment와 Padding을 목적별로 구분한다

Alignment는 Object 시작 주소의 배수 조건이고 padding은 compiler가 member alignment와 array stride를 맞추려고 넣는 빈 byte다.

Member 순서를 바꾸면 size가 줄 수 있지만 ABI와 storage 호환성을 확인해야 하며, padding byte를 Protocol에 전송하거나 `memcmp`으로 의미적 동등성을 검사하면 안 된다.

`alignas`는 다음 두 목적에 다르게 사용한다.

- SIMD 또는 hardware 명령이 요구하는 정렬을 만족한다.
- 서로 다른 writer의 state를 별도 coherence unit에 두어 false sharing을 줄인다.

과도한 alignment는 working set을 키운다. 직접 준비한 backing storage도 `alignof(T)`를 만족하는지 확인해야 한다.

# 8. False Sharing은 주소가 아니라 Writer 관계의 문제다

서로 다른 thread가 서로 다른 variable만 수정해도 두 variable이 같은 coherence unit에 있으면 line ownership이 core 사이를 왕복할 수 있다.

```text
Core 2 writes producer_cursor ─┐ 같은 cache line
Core 6 writes consumer_cursor ─┘
```

Data race 없이도 생기는 이 성능 문제를 false sharing이라고 한다.

Queue의 producer cursor와 consumer cursor, thread별 counter, 독립 shard의 통계가 흔한 후보이다.
해결은 무조건 padding을 추가하는 것이 아니라 다음 순서로 진행한다.

1. 각 field의 writer와 reader를 표시한다.
2. 실제 address가 어느 cache line에 놓이는지 확인한다.
3. Thread를 별도 physical core에 고정한다.
4. Padding 전후 throughput과 tail latency를 비교한다.
5. 대상 CPU가 지원하면 cache-to-cache 또는 HITM 관련 counter를 확인한다.

같이 읽는 field까지 멀리 떼면 locality가 나빠지므로 read-only sharing과 여러 writer의 sharing을 구분한다.

# 9. `reserve()`와 `resize()`는 다른 계약이다

`std::vector<T>`의 `size()`는 살아 있는 element 수이고 `capacity()`는 재할당 없이 담을 수 있는 element 수다.

| Operation | Size | Capacity와 Allocation | Object Lifetime |
| --- | --- | --- | --- |
| `reserve(n)` | 바뀌지 않음 | 현재 capacity보다 크면 재할당 가능 | 새 element를 만들지 않음 |
| `resize(n)` 증가 | `n`으로 증가 | capacity 부족 시 재할당 가능 | 새 element를 construction |
| `resize(n)` 감소 | `n`으로 감소 | 보통 capacity를 줄이지 않음 | 뒤 element를 destruction |
| `clear()` | 0 | capacity는 유지 | 모든 element를 destruction |

Initialization에서 `reserve(max_orders)`를 호출해도 다음 문제는 남는다.

- `max_orders`를 넘으면 이후 insertion이 다시 allocate할 수 있다.
- `reserve()` 자체가 기존 storage를 옮겨 pointer와 reference를 무효화할 수 있다.
- `resize()`는 element constructor나 initialization 비용을 수행한다.
- `shrink_to_fit()`은 non-binding request이며 Hot Path 정책으로 쓰기 어렵다.

Hot Path에서 vector를 사용하려면 최대 size, growth 금지 방식, iterator invalidation과 exhaustion 동작을 API 계약으로 만든다.

# 10. Allocation 경로를 끝까지 본다

C++ `new` expression은 대략 storage allocation과 object construction을 결합한다.
일반 allocator의 실제 경로는 구현과 크기에 따라 달라질 수 있다.

```text
operator new
  → Thread-local allocator cache에서 충족
  → Size-class central structure 접근
  → 다른 arena 또는 page 확보
  → OS mapping과 physical page의 first touch
```

매 allocation이 system call이나 global lock을 수행한다고 단정하면 안 된다.
반대로 평소 빠른 thread cache hit만 측정하고 worst path가 없다고 결론 내릴 수도 없다.

Deallocation도 공짜가 아니다.
마지막 owner의 destructor, allocator metadata 갱신, remote free, page 반환과 coalescing이 예상하지 못한 thread에서 실행될 수 있다.
Source Code에서 `new`를 검색하는 것만으로 다음 숨은 allocation을 찾을 수 없는 이유다.

- Capacity를 넘은 container growth
- Node-based container insertion
- 긴 string과 formatted logging
- Type-erased callback의 일부 구현
- 마지막 `shared_ptr` owner의 release 경로

Allocation counter, allocator trace와 production과 같은 build의 profile을 함께 사용한다.

# 11. Memory Pool의 세 가지 기본 형태

## 11.1 Monotonic Arena

Monotonic arena는 큰 영역에서 pointer를 앞으로 이동하며 allocation한다.
개별 deallocation은 하지 않고 phase가 끝날 때 전체를 release한다.

동일 lifetime을 가진 snapshot decode, batch scratch와 replay phase에 잘 맞는다.
수명이 긴 object와 짧은 object를 같은 arena에 섞으면 짧은 object의 memory도 phase 끝까지 남는다.

## 11.2 Fixed-Size Pool

같은 크기와 alignment의 slot을 미리 준비하고 하나씩 대여한다.
Order, session message와 queue node처럼 type과 최대 개수가 정해진 곳에 적합하다.

```text
Slot: [live][free]──┐ [live][free]←┘
```

Allocation과 release를 bounded O(1) operation으로 만들 수 있지만 사용하지 않는 slot도 memory를 점유한다.

## 11.3 Size-Class Free List

여러 크기가 필요하면 64, 128, 256 byte처럼 class별 pool을 둘 수 있다.
내부 fragmentation과 class 선택 비용이 생기며 큰 예외 allocation의 정책도 필요하다.

Free list가 object 내부 pointer를 재사용하는 intrusive 방식이면 metadata는 줄지만 object가 free 상태일 때 그 byte의 의미가 달라진다.
여러 thread가 공유하면 ABA, synchronization과 reclamation 문제가 추가되므로 처음에는 single-owner 또는 per-thread pool을 선호한다.

# 12. Fixed Pool에서도 Object Lifetime을 지킨다

Raw byte storage를 확보했다고 `T` Object가 자동으로 생기지는 않는다.

```text
정렬된 Slot 확보
  → `std::construct_at`으로 T의 lifetime 시작
  → 살아 있는 동안만 T로 접근
  → `std::destroy_at`으로 lifetime 종료
  → Slot을 free list로 반환
```

다음 예제는 single-thread 전용 고정 크기 pool이다.
Index handle을 사용하므로 해제 뒤의 오래된 handle을 다시 쓰지 않는다는 상위 계약이 필요하다.

```cpp
#include <array>
#include <cassert>
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <memory>
#include <optional>
#include <utility>

template <typename T, std::size_t Capacity>
class FixedPool {
    static_assert(Capacity > 0);

    struct Slot {
        alignas(T) std::array<std::byte, sizeof(T)> bytes{};
    };

public:
    FixedPool() noexcept {
        for (std::size_t i = 0; i < Capacity; ++i) {
            next_[i] = i + 1;
        }
    }

    ~FixedPool() {
        for (std::size_t i = 0; i < Capacity; ++i) {
            if (active_[i]) {
                std::destroy_at(live_object(i));
            }
        }
    }

    FixedPool(const FixedPool&) = delete;
    FixedPool& operator=(const FixedPool&) = delete;

    template <typename... Args>
    std::optional<std::size_t> emplace(Args&&... args) {
        if (free_head_ == Capacity) {
            return std::nullopt;
        }
        const std::size_t index = free_head_;
        std::construct_at(raw_slot(index), std::forward<Args>(args)...);
        free_head_ = next_[index];
        active_[index] = true;
        return index;
    }

    T* get(std::size_t index) noexcept {
        if (index >= Capacity || !active_[index]) {
            return nullptr;
        }
        return live_object(index);
    }

    bool erase(std::size_t index) noexcept {
        T* const object = get(index);
        if (object == nullptr) {
            return false;
        }
        std::destroy_at(object);
        active_[index] = false;
        next_[index] = free_head_;
        free_head_ = index;
        return true;
    }

private:
    T* raw_slot(std::size_t index) noexcept {
        return reinterpret_cast<T*>(slots_[index].bytes.data());
    }

    T* live_object(std::size_t index) noexcept {
        return std::launder(raw_slot(index));
    }

    std::array<Slot, Capacity> slots_{};
    std::array<std::size_t, Capacity> next_{};
    std::array<bool, Capacity> active_{};
    std::size_t free_head_{0};
};

struct Order {
    Order(std::uint64_t id_value, std::int64_t price_value,
          std::int32_t quantity_value) noexcept
        : id{id_value}, price{price_value}, quantity{quantity_value} {}

    std::uint64_t id;
    std::int64_t price;
    std::int32_t quantity;
};

int main() {
    FixedPool<Order, 2> pool;
    const auto first = pool.emplace(1001, 10'125, 10);
    const auto second = pool.emplace(1002, 10'126, 20);
    const auto exhausted = pool.emplace(1003, 10'127, 30);

    assert(first.has_value());
    assert(second.has_value());
    assert(!exhausted.has_value());
    assert(pool.get(*first)->id == 1001);

    assert(pool.erase(*first));
    const auto reused = pool.emplace(1004, 10'128, 40);
    assert(reused.has_value());
    std::cout << pool.get(*reused)->id << '\n';
}
```

이 예제는 production-ready concurrent allocator가 아니다.
Pool보다 오래 pointer를 보관하면 안 되고 slot 재사용을 구분해야 한다면 generation을 handle에 추가해야 한다.
큰 `Capacity`의 pool을 thread stack에 두는 것도 피하고 초기화 단계의 명시적 owner가 관리하게 한다.

Constructor가 throw하면 `free_head_`를 바꾸기 전에 stack unwinding되므로 slot은 유지된다.
Hot Path에서 사용할 type은 실패 계약을 더 단순하게 만들 수 있는지 검토하되 무조건 `noexcept`를 붙이지 않는다.

# 13. Placement Construction의 Caveat

Placement construction은 일반 allocation을 우회할 뿐 다음 책임을 없애지 않는다.

- Storage size와 alignment가 `T`에 충분해야 한다.
- Object가 살아 있을 때만 `T`로 접근해야 한다.
- Non-trivial destructor는 lifetime 종료 전에 실행해야 한다.
- 재사용 후 이전 pointer와 handle의 유효성을 관리해야 한다.
- Pool이 파괴될 때 남은 live object를 정리해야 한다.

C++20에서는 raw placement-new 표현을 직접 반복하기보다 `std::construct_at`과 `std::destroy_at`으로 의도를 드러낼 수 있다.
`std::launder`도 잘못된 lifetime을 고쳐주는 만능 함수가 아니다.
먼저 유효하게 construction된 object가 있어야 한다.

# 14. `std::pmr`가 주는 것과 주지 않는 것

`std::pmr` container는 allocation 정책을 `std::pmr::memory_resource`로 분리한다.
Component마다 arena 또는 pool을 주입하고 일반 container API를 유지하는 데 유용하다.

`std::pmr::monotonic_buffer_resource`는 allocation을 누적하고 resource destruction 또는 `release()`에서 한꺼번에 되돌리는 phase-lifetime resource다.
그러나 다음 caveat가 있다.

- Initial buffer가 부족하면 기본적으로 upstream resource에 추가 allocation을 요청할 수 있다.
- 완전히 bounded하게 만들려면 충분한 buffer와 `std::pmr::null_memory_resource()` 같은 실패 upstream을 명시해야 한다.
- 개별 `deallocate`는 storage를 재사용 가능한 상태로 즉시 반환하지 않는다.
- Resource는 그 resource를 사용하는 container와 element보다 오래 살아야 한다.
- `release()` 뒤에는 그 storage를 가리키는 object와 reference를 사용할 수 없다.
- `memory_resource` 경유 호출과 container operation 비용은 실제 구현을 측정해야 한다.
- Allocator propagation과 서로 다른 resource 사이의 move 동작을 추측하지 말고 계약을 확인해야 한다.

PMR은 “allocation이 사라진다”는 기능이 아니라 allocation의 위치와 lifetime을 명시하는 도구다.

# 15. Capacity 고갈은 정상적으로 설계해야 하는 상태다

Bounded pool은 언젠가 가득 찰 수 있다.
그때 일반 heap으로 조용히 fallback하면 평소 측정에 없던 latency와 memory pressure가 가장 나쁜 순간에 나타난다.

Component별 정책은 다를 수 있다.

| Component | 가능한 Exhaustion Policy |
| --- | --- |
| Pre-Trade Risk | Fail closed하고 새 주문을 거부 |
| Order Gateway | 신규 주문을 중단하고 cancel용 reserve는 보호 |
| Market Data | Gap/overload 상태로 전환하고 recovery 시작 |
| Telemetry | 낮은 우선순위 event drop 후 counter 증가 |
| Replay Tool | 명시적 오류로 run 중단 또는 사전 capacity 재계산 |

어떤 정책이든 다음 값은 관측 가능해야 한다.

- 현재 사용량과 high-water mark
- allocation failure 횟수
- pool별 capacity
- fallback 여부
- recovery 또는 operator action

Cancel과 risk-control message가 일반 traffic 고갈 때문에 생성되지 못하지 않도록 별도 reserve를 둘 수 있다.
단, reserve 분리 자체가 starvation을 만들지 않는지 fault injection으로 확인한다.

# 16. Pre-Touch와 NUMA Placement

Virtual address를 reserve하거나 heap에서 큰 영역을 얻었다고 physical page가 모두 준비된 것은 아니다.
첫 접근에서 page fault와 physical page allocation이 일어날 수 있다.

```text
Initialization thread on NUMA Node 0
  → Pool touch → Page가 Node 0에 배치될 수 있음
Trading worker on Node 1 → Remote access 가능
```

일반적인 준비 순서는 다음과 같다.

1. Worker를 목표 CPU 또는 NUMA node에 배치한다.
2. Memory policy와 NIC topology를 확인한다.
3. 실제 owner thread가 pool의 각 page를 write하여 pre-touch한다.
4. 필요하면 memory locking과 huge page 정책을 별도 실험한다.
5. Minor fault, page 위치, RSS와 latency를 warm-up 뒤 확인한다.

Pre-touch는 cache를 영구히 warm하게 만들지 않는다.
Page table과 physical page 준비를 앞당기는 작업이며, 이후 cache eviction과 NUMA traffic은 여전히 발생한다.
Compiler가 의미 없는 초기화를 제거하지 않았는지와 실제 page가 배치됐는지도 측정한다.

First-touch 동작, automatic NUMA balancing과 explicit binding은 운영체제 설정에 따라 달라질 수 있다.
자세한 배치는 [[Memory] NUMA 구조와 Local·Remote Memory](/posts/numa-memory-architecture/)의 실험과 함께 검증한다.

# 17. Fragmentation과 Reclamation

Fragmentation은 두 형태로 나눌 수 있다.

| 종류 | 의미 | 예시 |
| --- | --- | --- |
| Internal | Slot 안에서 사용하지 못하는 공간 | 65-byte object를 128-byte class에 저장 |
| External | Free 공간 총량은 충분하지만 필요한 연속 block이 없음 | 크기가 다른 allocation과 free가 섞임 |

Fixed-size pool은 external fragmentation을 줄이는 대신 unused capacity와 internal fragmentation을 만든다.
Size-class를 지나치게 많이 만들면 pool별 여유 공간과 운영 복잡성이 커진다.

Reclamation 시점도 latency의 일부다.
수천 object의 destructor와 page 반환을 한 번에 수행하면 phase 경계에서 spike가 생길 수 있다.
그 시간이 Hot Path 밖인지, shutdown budget에 들어가는지와 다음 session 시작 전에 끝나는지 명시한다.

여러 thread가 object를 읽는 동안 writer가 slot을 즉시 재사용하면 use-after-free가 된다.
Hazard pointer, epoch-based reclamation과 RCU 계열은 안전한 reclamation을 제공할 수 있지만 추가 metadata와 지연된 회수를 만든다.
가능하면 single-writer ownership, message passing 또는 명시적 safe point로 문제 자체를 단순화한 뒤 고급 reclamation을 검토한다.

# 18. Layout Microbenchmark 예제

다음 프로그램은 모든 field를 가진 AoS와 scan에 필요한 두 field만 연속으로 둔 SoA를 비교한다.
완전한 Trading Workload가 아니라 layout 가설을 분리하는 출발점이다.

```cpp
#include <chrono>
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <utility>
#include <vector>

struct QuoteAoS {
    std::int64_t price;
    std::int32_t quantity;
    std::uint32_t venue;
    std::uint64_t timestamp;
};

struct QuoteSoA {
    std::vector<std::int64_t> prices;
    std::vector<std::int32_t> quantities;
    std::vector<std::uint32_t> venues;
    std::vector<std::uint64_t> timestamps;
};

template <typename Function>
std::pair<std::int64_t, std::int64_t> measure(Function&& function) {
    const auto begin = std::chrono::steady_clock::now();
    const std::int64_t checksum = function();
    const auto end = std::chrono::steady_clock::now();
    const auto elapsed = std::chrono::duration_cast<std::chrono::nanoseconds>(
        end - begin);
    return {checksum, elapsed.count()};
}

int main() {
    constexpr std::size_t count = 1U << 20U;
    constexpr int repetitions = 20;
    std::vector<QuoteAoS> aos(count);
    QuoteSoA soa{std::vector<std::int64_t>(count),
                 std::vector<std::int32_t>(count),
                 std::vector<std::uint32_t>(count),
                 std::vector<std::uint64_t>(count)};

    for (std::size_t i = 0; i < count; ++i) {
        const auto price = static_cast<std::int64_t>(10'000 + (i % 256));
        const auto quantity = static_cast<std::int32_t>(1 + (i % 100));
        const auto venue = static_cast<std::uint32_t>(i % 4);
        const auto timestamp = static_cast<std::uint64_t>(i);
        aos[i] = QuoteAoS{price, quantity, venue, timestamp};
        soa.prices[i] = price;
        soa.quantities[i] = quantity;
        soa.venues[i] = venue;
        soa.timestamps[i] = timestamp;
    }

    const auto aos_result = measure([&] {
        std::int64_t total = 0;
        for (int repeat = 0; repeat < repetitions; ++repeat) {
            for (const QuoteAoS& quote : aos) {
                if (quote.price >= 10'128) total += quote.quantity;
            }
        }
        return total;
    });
    const auto soa_result = measure([&] {
        std::int64_t total = 0;
        for (int repeat = 0; repeat < repetitions; ++repeat) {
            for (std::size_t i = 0; i < count; ++i) {
                if (soa.prices[i] >= 10'128) total += soa.quantities[i];
            }
        }
        return total;
    });
    if (aos_result.first != soa_result.first) return 1;
    std::cout << "AoS ns=" << aos_result.second
              << " SoA ns=" << soa_result.second
              << " checksum=" << aos_result.first << '\n';
}
```

이 예제 한 번의 숫자로 결론 내리면 안 된다.
초기화는 측정 구간 밖에 있지만 allocation strategy, compiler auto-vectorization과 CPU frequency가 결과에 영향을 준다.
AoS와 SoA 실행 순서를 번갈아 배치하고 여러 run의 distribution을 남긴다.

# 19. Benchmark를 설계하는 순서

1. Scan, lookup, insert, cancel과 snapshot의 production 비율을 고정하고 layout 외 algorithm은 바꾸지 않는다.
2. 동일 replay의 최종 state, checksum, constructor/destructor 수와 invariant를 비교한다.
3. Element 수를 L1, L2, Last-Level Cache와 DRAM 범위까지 단계적으로 키운다.
4. Cold start와 pre-touch·warm-up 뒤 steady state를 분리하고 initialization·reclamation도 보고한다.
5. p50, p99, p99.9, max, operations/s와 allocation failure를 함께 기록한다.
6. Cache·TLB·branch counter가 시간 차이의 mechanism을 뒷받침하는지 확인한다.

Linux `perf`를 사용할 수 있는 환경의 후보 event 예시는 다음과 같다.

```bash
perf stat -r 10 \
  -e cycles,instructions,cache-references,cache-misses \
  -e dTLB-loads,dTLB-load-misses,branches,branch-misses \
  ./layout_bench
```

Event 이름, 지원 범위와 multiplexing 여부는 CPU와 kernel마다 다르다.
Counter가 가설을 지지하지 않으면 frequency, code generation과 측정 순서도 확인한다.

# 20. Test Matrix

| 대상 | 필수 Test |
| --- | --- |
| Layout | 동일 결과, sequential·random access, hot-only·all-field scan |
| Boundary | 1개, capacity 직전, capacity와 capacity 초과 |
| Pool lifetime | Slot 재사용, stale handle 정책, destructor 정확히 1회, throwing constructor |
| Alignment | Over-aligned type의 실제 address와 `sizeof`·working set |
| NUMA | Touch·사용 thread의 node, local·remote placement |
| False sharing | SMT sibling·별도 core, padding 전후 tail·throughput·coherence counter |
| Tool | ASan, UBSan과 공유 pool을 만들었다면 concurrency test |

# 완료 기준

다음을 만족하면 이 단계의 학습을 완료한 것으로 본다.

1. Hot Path의 access stream과 working set을 byte 단위로 추정한다.
2. 같은 workload에서 AoS, SoA와 hybrid layout의 결과를 비교한다.
3. Flat structure와 node-based structure의 locality와 update 비용을 함께 설명한다.
4. False sharing을 writer 관계, address와 counter로 재현한다.
5. `reserve()`와 `resize()`의 size, capacity와 lifetime 차이를 설명한다.
6. Warm-up 이후 예상하지 않은 general allocation이 0회인지 계측한다.
7. Fixed pool에서 construction, destruction, exhaustion과 stale handle을 test한다.
8. PMR resource의 upstream과 lifetime 정책을 문서화한다.
9. Owner thread에서 pre-touch하고 NUMA page 위치와 page fault를 확인한다.
10. p50, p99, p99.9, max와 cache·TLB counter를 포함한 비교 보고서를 작성한다.

# 자주 생기는 오해

## Cache line에 맞추면 무조건 빨라진다

Alignment는 필요한 data를 더 가깝게 만들 수도 있지만 padding 때문에 working set을 키울 수도 있다.
False sharing 방지와 SIMD 요구처럼 목적을 정하고 전후를 측정한다.

## SoA는 AoS보다 항상 빠르다

일부 field만 순회할 때는 SoA가 유리할 수 있다.
한 record의 모든 field를 함께 쓰면 AoS가 더 적은 stream과 단순한 API를 제공할 수 있다.

## `reserve()`를 호출했으므로 allocation은 끝났다

Capacity를 넘는 insertion은 재할당할 수 있다.
최대 크기와 초과 정책을 지키는 test와 runtime counter가 필요하다.

## Memory Pool을 사용하면 Memory 문제가 사라진다

Pool도 capacity 고갈, internal fragmentation, stale handle, 잘못된 NUMA placement와 reclamation spike를 만들 수 있다.
Pool은 비용을 없애는 것이 아니라 시점과 상한을 통제하는 설계다.

## Placement New는 Constructor를 생략한다

Placement construction은 caller가 제공한 storage에서 constructor를 실행한다.
Object lifetime과 destructor 책임은 그대로 남는다.

## Pre-Touch하면 모든 접근이 Cache Hit다

Pre-touch는 page fault와 page placement를 준비하는 데 도움을 준다.
Cache residency, coherence와 이후 eviction을 보장하지 않는다.

# 정리

Cache-Friendly Design은 structure member를 기계적으로 재배열하는 작업이 아니다.

```text
Access Pattern을 기록한다.
  → Working Set과 Writer Ownership을 계산한다.
  → AoS·SoA·Flat·Node 후보를 만든다.
  → Lifetime에 맞는 Arena 또는 Pool을 선택한다.
  → Capacity와 Exhaustion Policy를 고정한다.
  → Pre-Touch와 NUMA Placement를 준비한다.
  → Correctness, Tail과 Hardware Counter로 검증한다.
```

좋은 layout은 가장 작은 `sizeof(T)`가 아니라 목표 operation이 필요한 byte를 예측 가능한 순서로 가져오는 layout이다.
좋은 pool도 가장 빠른 allocation 하나가 아니라 정상 처리, 고갈과 reclamation까지 bounded contract로 만드는 pool이다.

# 참고 자료

- [ISO C++ Working Draft — Object Lifetime](https://eel.is/c++draft/basic.life)
- [ISO C++ Working Draft — Dynamic Memory Management](https://eel.is/c++draft/support.dynamic)
- [ISO C++ Working Draft — Memory Resources](https://eel.is/c++draft/mem.res)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html)
- [AMD Software Optimization Guide for AMD Family 19h Processors](https://docs.amd.com/v/u/en-US/57228)
- [Arm Cortex-A Series Programmer's Guide](https://developer.arm.com/documentation/den0013/latest/)
