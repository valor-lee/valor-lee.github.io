---
title: '[Low Latency Trading] Hot Path를 위한 Modern C++ 설계 원칙'
date: 2026-08-07 00:40:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    Modern C++,
    hot path,
    performance engineering,
    correctness
  ]
---

# 개요

초저지연 Trading System에서는 C++ 문법을 많이 아는 것만으로 충분하지 않다.
Market Data 한 건을 처리하고 주문을 만드는 동안 어떤 Object가 살아 있는지, 누가 Memory를 소유하는지, 어느 연산이 실패할 수 있는지를 예측할 수 있어야 한다.
```text
Market Data
    → Decode
    → Order Book Update
    → Strategy Decision
    → Pre-Trade Risk
    → Order Encode
```
이 경로를 Hot Path라고 하자.
Hot Path의 목표는 단순히 instruction 수를 줄이는 것이 아니다.
```text
Correctness
  + Bounded Work
  + Predictable Memory Access
  + Measured Optimization
  = 운영 가능한 Low-Latency Code
```
잘못된 가격이나 수량으로 빠르게 주문을 보내는 시스템은 느리지만 정확한 시스템보다 위험하다.
이 글에서는 correctness를 지키면서 Hot Path의 비용을 통제하는 Modern C++ 설계 원칙을 정리한다.

# 선수 글과 BFS 위치

엄격한 선수 글은 `[Low Latency Trading] 초저지연 트레이딩 시스템 전체 구조`다.

Cache, TLB, Endianness와 NUMA 글은 선행 조건이 아니라 같은 Level에서 함께 읽는 보강 자료다.

이 글의 BFS 위치는 **Level 1-C: Modern C++ 공통 기반**이다.
다음 단계에서는 이 원칙을 이용해 Feed Handler, Order Book, Order Gateway와 SPSC Pipeline을 구현한다.

# 1. Correctness가 첫 번째 성능 조건이다

Trading System의 오류는 일반적인 Server Error보다 직접적인 상태 변화를 만들 수 있다.

- Price와 Quantity를 뒤바꾼다.
- 음수 또는 범위를 벗어난 Quantity를 허용한다.
- 이미 끝난 Order Object를 참조한다.
- Integer overflow로 Risk Limit을 우회한다.
- Decode되지 않은 Packet의 일부를 정상 값으로 사용한다.
- Data Race 때문에 같은 Order를 두 번 보낸다.

이런 문제는 평균 latency가 낮아도 허용할 수 없다.
먼저 Hot Path의 invariant를 적어야 한다.
```text
Price는 유효한 Tick 단위다.
Quantity는 0보다 크고 상품 한도 이하다.
Sequence Number는 Stream 규칙에 따라 전진한다.
Client Order ID는 Process 안에서 중복되지 않는다.
Risk Check를 통과하지 않은 Order는 Encode할 수 없다.
Borrow한 Buffer는 Owner보다 오래 살지 않는다.
```
최적화는 이 invariant를 보존한다는 증거가 있을 때만 적용한다.

# 2. Domain 값을 Primitive Type으로만 표현하지 않는다

다음 함수는 compile되지만 인자를 바꾸어 전달하기 쉽다.
```cpp
void send_order(std::int64_t price, std::int64_t quantity);
std::int64_t price = 101250;
std::int64_t quantity = 10;
send_order(quantity, price); // Compile은 되지만 의미가 틀렸다.
```
Price와 Quantity가 같은 underlying type을 사용하더라도 C++ type은 분리하는 것이 좋다.
```cpp
class Price {
public:
    static std::optional<Price> from_ticks(std::int64_t value) noexcept {
        return value > 0 ? std::optional{Price{value}} : std::nullopt;
    }
    std::int64_t ticks() const noexcept { return ticks_; }
private:
    explicit Price(std::int64_t value) noexcept : ticks_{value} {}
    std::int64_t ticks_;
};

class Quantity {
public:
    static std::optional<Quantity> from_units(std::int32_t value) noexcept {
        return value > 0 && value <= 1'000'000
            ? std::optional{Quantity{value}} : std::nullopt;
    }
    std::int32_t units() const noexcept { return units_; }
private:
    explicit Quantity(std::int32_t value) noexcept : units_{value} {}
    std::int32_t units_;
};
```
유효하지 않은 값을 생성 시점에서 거부하면 이후 경로의 조건문과 해석 가능성이 줄어든다.
Strong Type은 반드시 큰 class hierarchy여야 하는 것이 아니다.
작고 trivially copyable한 value type으로도 의미를 충분히 표현할 수 있다.

# 3. 가격과 금액은 Fixed-Point로 표현한다

Binary floating-point는 많은 decimal fraction을 정확히 표현하지 못한다.
```cpp
double price = 0.1 + 0.2;
```
Trading Domain에서는 다음 정책을 명시해야 한다.

- Tick Size
- 표시 소수 자릿수
- 반올림 방식
- Currency와 Contract Multiplier
- Intermediate Result의 범위

가격을 Tick 개수 또는 정해진 Scale의 Integer로 저장할 수 있다.
```text
Tick Size: 0.01
표시 가격: 101.25
내부 값: 10125 ticks
```
Fixed-Point가 overflow를 자동으로 없애는 것은 아니다.
Price와 Quantity가 양수라는 invariant를 이용하면 곱셈 전에 범위를 검사할 수 있다.
```cpp
std::optional<std::int64_t> checked_notional(
    Price price, Quantity quantity) noexcept {
    const auto p = price.ticks();
    const auto q = static_cast<std::int64_t>(quantity.units());
    const auto max = std::numeric_limits<std::int64_t>::max();
    if (q > max / p) {
        return std::nullopt;
    }
    return p * q;
}
```
Signed integer overflow는 C++에서 Undefined Behavior가 될 수 있다.
Risk 계산에서 “overflow한 뒤 검사”하면 이미 늦다.

# 4. Object Lifetime을 Memory 주소와 혼동하지 않는다

Memory 영역이 존재한다고 해서 그곳에 특정 C++ Object가 살아 있다는 뜻은 아니다.
```text
Storage 확보
    → Object Lifetime 시작
    → Object 사용
    → Lifetime 종료
    → Storage 재사용 또는 해제
```
아래 방식으로 Network Buffer를 Structure Pointer로 바꾸면 여러 문제가 생길 수 있다.
```cpp
const auto* message =
    reinterpret_cast<const WireMessage*>(buffer.data());
```
- Buffer alignment가 `WireMessage` 요구사항을 만족하지 않을 수 있다.
- Protocol byte order와 Host byte order가 다를 수 있다.
- Structure padding과 Wire Layout이 다를 수 있다.
- Buffer 길이가 부족할 수 있다.
- Object lifetime과 aliasing 규칙을 위반할 수 있다.

Wire Format과 Domain Object를 분리하고 byte 단위로 검증하며 Decode하는 편이 안전하다.

# 5. Allocation과 Construction을 구분한다

다음 두 작업은 같은 개념이 아니다.
```text
Allocation
  → Object를 둘 Storage를 확보한다.
Construction
  → 그 Storage에서 Object Lifetime을 시작한다.
```
Heap Allocation은 allocator lock, metadata 접근, page fault와 불규칙한 실행 시간을 만들 수 있다.
하지만 `new` 문자열만 지운다고 문제가 해결되는 것은 아니다.
다음 작업도 내부적으로 allocation할 수 있다.

- Capacity를 넘겨 성장하는 `std::vector`
- 긴 문자열을 만드는 `std::string`
- Node를 추가하는 associative container
- 일부 callback wrapper 또는 type erasure 사용
- 임시 Log Message 조립
- 마지막 Owner가 해제되는 `shared_ptr`

Allocation 위치는 Source Code의 한 줄이 아니라 실행되는 전체 호출 경로에서 확인해야 한다.

# 6. Hot Path 이전에 Memory를 준비한다

일반적인 실행 단계를 다음처럼 나눌 수 있다.
```text
Initialization
  → Config 검증
  → Buffer와 Queue Allocate
  → Container Capacity 확정
  → Memory Page Pre-Touch
  → Connection과 Session 준비
Warm-up
  → 대표 Message로 Code와 Data Path 준비
Hot Path
  → 이미 확보한 Storage 안에서 Bounded Work 수행
Shutdown
  → Resource 정리와 Audit 완료
```
`std::vector::reserve()`는 유용하지만 약속을 문서화해야 한다.
```cpp
std::vector<BookLevel> levels;
levels.reserve(max_levels);
```
`reserve()` 이후에도 `size() == capacity()`에서 `push_back()`하면 다시 allocation할 수 있다.

입력의 최대 크기, 초과 시 동작과 복구 정책이 함께 있어야 한다.
초과 입력을 무한히 받아들이는 것보다 명확하게 거부하고 Metric을 올리는 편이 예측 가능하다.

# 7. Bounded Container를 사용한다

최대 크기를 Protocol이나 Risk Policy에서 알 수 있다면 type에 반영할 수 있다.
```cpp
template <typename T, std::size_t Capacity>
class FixedBuffer {
public:
    bool push(const T& value) noexcept {
        if (size_ == Capacity) return false;
        data_[size_++] = value;
        return true;
    }
    const T& operator[](std::size_t i) const noexcept { return data_[i]; }
    std::size_t size() const noexcept { return size_; }
private:
    std::array<T, Capacity> data_{};
    std::size_t size_{};
};
```
이 예제의 `operator[]`는 bounds check를 하지 않는다.
호출자가 `index < size()`를 보장하는 invariant가 필요하다.
Debug Build에서는 assertion을 추가하고 Release Build에서도 외부 입력 경계는 반드시 검증해야 한다.

# 8. RAII는 Low Latency와 충돌하지 않는다

RAII는 Resource Lifetime을 Object Lifetime에 묶는 원칙이다.
Resource는 Heap Memory만 의미하지 않는다.

- Socket File Descriptor
- Memory Mapping
- NIC Queue Handle
- Lock
- Packet Capture File
- PTP Clock File Descriptor

RAII를 사용하면 early return과 오류 경로에서도 해제를 빠뜨릴 가능성이 줄어든다.
```cpp
class UniqueFd {
public:
    explicit UniqueFd(int fd = -1) noexcept : fd_{fd} {}
    ~UniqueFd() { if (fd_ >= 0) ::close(fd_); }
    UniqueFd(const UniqueFd&) = delete;
    UniqueFd& operator=(const UniqueFd&) = delete;
    UniqueFd(UniqueFd&& other) noexcept : fd_{other.fd_} {
        other.fd_ = -1;
    }
    int get() const noexcept { return fd_; }
private:
    int fd_;
};
```
중요한 것은 destructor의 비용이 언제 실행되는지 아는 것이다.
마지막 Owner의 해제나 File Close가 latency-sensitive 구간에서 실행되지 않도록 Lifetime 경계를 설계한다.

# 9. Ownership을 Type과 API에 드러낸다

Pointer가 있다고 해서 누가 Object를 해제해야 하는지는 알 수 없다.
일반적인 구분은 다음과 같다.

| 표현 | 의미 |
| --- | --- |
| Value | Object 자체를 소유한다. |
| `std::unique_ptr<T>` | 단일 Owner를 이동시킨다. |
| `std::span<T>` | 연속 영역을 빌리며 소유하지 않는다. |
| `T&` | 존재해야 하는 Object를 빌린다. |
| `T*` | Nullable Observer 또는 저수준 Handle이다. |
| `std::shared_ptr<T>` | 공유 Lifetime이 실제 요구사항일 때 사용한다. |

`shared_ptr`가 무조건 나쁜 것은 아니다.

다만 reference count 갱신, 마지막 해제 위치와 불명확한 Lifetime이 Hot Path 요구와 맞는지 확인해야 한다.
Market Data Packet을 Decode할 때는 Owner를 복제하기보다 제한된 Scope에서 `std::span<const std::byte>`를 빌릴 수 있다.

# 10. Wire Buffer를 Allocation 없이 Decode한다

아래 예제는 길이와 값을 검증하고 Domain Type을 만든다.
```cpp
template <typename UInt>
UInt read_be(std::span<const std::byte> bytes,
             std::size_t offset) noexcept {
    static_assert(std::is_unsigned_v<UInt>);
    UInt value = 0;
    for (std::size_t i = 0; i < sizeof(UInt); ++i) {
        value = (value << 8) | std::to_integer<std::uint8_t>(bytes[offset + i]);
    }
    return value;
}

struct BookUpdate {
    std::uint64_t sequence;
    Price price;
    Quantity quantity;
};

std::optional<BookUpdate> decode_update(
    std::span<const std::byte> bytes) noexcept {
    constexpr std::size_t wire_size = 20;
    if (bytes.size() < wire_size) return std::nullopt;
    const auto sequence = read_be<std::uint64_t>(bytes, 0);
    const auto raw_price = read_be<std::uint64_t>(bytes, 8);
    const auto raw_quantity = read_be<std::uint32_t>(bytes, 16);
    if (raw_price > std::uint64_t{std::numeric_limits<std::int64_t>::max()} ||
        raw_quantity > std::uint32_t{std::numeric_limits<std::int32_t>::max()})
        return std::nullopt;
    const auto price = Price::from_ticks(static_cast<std::int64_t>(raw_price));
    const auto quantity = Quantity::from_units(static_cast<std::int32_t>(raw_quantity));
    if (!price || !quantity) return std::nullopt;
    return BookUpdate{sequence, *price, *quantity};
}
```
`std::span`과 `std::optional`은 이 사용 방식에서 Heap Allocation을 요구하지 않는다.

하지만 “Standard Library Type이므로 빠르다”가 아니라 Compiler Output과 실제 측정으로 확인해야 한다.

# 11. Contiguous Layout을 기본 후보로 둔다

연속된 Memory Layout은 spatial locality와 hardware prefetch에 유리할 수 있다.
```text
Contiguous Array
[Level 0][Level 1][Level 2][Level 3]
Node-Based Structure
[Node] -> [Node] -> [Node] -> [Node]
```
Pointer Chasing은 다음 Node 주소를 알아야 다음 Load를 시작할 수 있다.
Order Book이라고 해서 항상 `std::map`이 정답인 것은 아니다.
가격 범위, Tick 수, update pattern과 필요한 operation을 바탕으로 sorted vector, flat structure, dense array와 tree를 비교해야 한다.
AoS와 SoA도 접근 패턴으로 결정한다.
```text
AoS: [price, qty][price, qty][price, qty]
SoA: [price, price, price] [qty, qty, qty]
```
한 번에 모든 field를 사용하면 AoS가 단순할 수 있다.
한 field만 순회하거나 SIMD를 사용하면 SoA가 유리할 수 있다.

# 12. Exception, Virtual Function, STL에 대한 오해

## 12.1 Exception은 존재하기만 해도 항상 느리다

구현과 ABI에 따라 throw하지 않는 정상 경로의 직접 비용은 작을 수 있다.
하지만 실제 throw는 stack unwinding과 불규칙한 제어 흐름을 만들며 latency budget 안에 두기 어렵다.
외부 입력 오류처럼 예상 가능한 결과는 `std::optional`, Result Type 또는 Error Code로 표현할 수 있다.
Exception 사용 정책은 Component 경계와 Build 설정까지 포함해 정한다.

## 12.2 Virtual Function은 무조건 금지해야 한다

Virtual Dispatch는 간접 분기이며 inlining을 막을 수 있다.
반대로 호출 대상이 안정적이거나 Compiler가 devirtualize하면 영향이 작을 수도 있다.
먼저 해당 호출이 Hot Path에 있는지, call frequency와 branch behavior가 어떤지 측정한다.

## 12.3 STL은 느리므로 모두 직접 구현해야 한다

Standard Library는 하나의 자료구조가 아니다.

`std::array`, `std::span`, `std::vector`와 algorithm은 명확한 Lifetime과 연속 Layout을 표현하는 데 유용하다.

반면 성장, allocation, hashing, node allocation 같은 개별 Operation의 계약은 확인해야 한다.
직접 만든 Container도 invariant, iterator invalidation, exception safety와 test가 없으면 더 위험하다.

# 13. Compiler Optimization은 C++ 규칙을 전제로 한다

Compiler는 Observable Behavior를 유지하는 범위에서 Code를 바꿀 수 있다.
Undefined Behavior가 있으면 Source Code에서 기대한 의미를 보존할 의무가 없다.
특히 확인할 항목은 다음과 같다.

- Signed integer overflow
- Out-of-bounds access
- Use-after-free와 dangling reference
- Uninitialized value
- Misaligned access
- Strict aliasing 위반
- Object lifetime 위반
- Data race

Debug Build에서 우연히 동작하던 Code가 `-O2` 또는 LTO에서 달라질 수 있다.
이는 Compiler가 거래 시스템을 이해하지 못해서가 아니라 Program이 C++의 유효한 의미를 제공하지 않았기 때문일 수 있다.

`volatile`은 일반 Thread Synchronization이나 Undefined Behavior 해결책이 아니다.

# 14. Debug Profile과 Release Profile의 역할을 나눈다

Sanitizer Build는 정확성을 찾기 위한 Build다.
```bash
clang++ -std=c++20 -O1 -g -fno-omit-frame-pointer \
  -fsanitize=address,undefined app.cpp
clang++ -std=c++20 -O1 -g -fno-omit-frame-pointer \
  -fsanitize=thread app.cpp
```
ASan/UBSan과 TSan은 별도 실행 대상으로 두는 편이 일반적이다.
Sanitizer가 추가한 instrumentation 때문에 이 Build의 latency를 Production 수치로 사용하면 안 된다.
성능 측정용 Profile은 배포 후보와 같은 Compiler, 표준 라이브러리, 최적화, Link 설정을 사용한다.
```bash
clang++ -std=c++20 -O2 -g -DNDEBUG \
  -fno-omit-frame-pointer -march=<target-cpu> app.cpp
```
`-march=native`로 Build한 Binary를 다른 CPU에 배포하면 지원 instruction과 tuning이 달라질 수 있다.

Target CPU를 명시하고 Build Artifact와 Flag를 함께 기록한다.
LTO, PGO, `-O3`, 강제 inline과 branch hint는 Benchmark로 이득을 확인한 뒤 적용한다.

# 완료 기준

다음을 만족하면 이 단계의 학습을 완료한 것으로 본다.

1. Price, Quantity와 Order ID를 서로 바꿔 쓸 수 없는 Type으로 표현한다.
2. Fixed-Point 곱셈의 overflow와 외부 입력 범위를 Test한다.
3. Wire Buffer를 `reinterpret_cast` 없이 allocation-free로 Decode한다.
4. Hot Path의 Owner와 Borrower Lifetime을 Diagram으로 설명한다.
5. Warm-up 이후 Heap Allocation 0회를 계측으로 확인한다.
6. Contiguous Layout과 대안 자료구조를 같은 Workload로 비교한다.
7. ASan/UBSan/TSan Test와 최적화 Release Test를 모두 통과한다.
8. 적용한 Compiler Flag마다 성능 또는 운영상의 근거를 남긴다.

# 자주 생기는 오해

## Strong Type은 Abstraction 비용 때문에 느리다

작고 단순한 Value Type은 inlining 후 underlying integer와 같은 Code로 최적화될 수 있다.
실제 Compiler Output을 확인해야 하며 Type Safety를 추측만으로 제거하면 안 된다.

## Heap을 사용하지 않으면 Page Fault가 없다

미리 확보한 Memory도 처음 접근할 때 Physical Page가 배치될 수 있다.
Preallocation과 함께 pre-touch, page policy와 memory locking 요구를 검토해야 한다.

## `noexcept`를 붙이면 자동으로 빨라진다

`noexcept`는 실패 계약이다.

실제로 throw하면 `std::terminate`가 호출되므로 실행 환경과 호출 계약을 확인하지 않고 성능 hint처럼 붙이면 안 된다.

## Release에서 Assertion을 지우면 입력 검증도 지워도 된다

내부 invariant를 위한 Debug Assertion과 신뢰할 수 없는 외부 입력 검증은 역할이 다르다.
Protocol 길이, 값 범위와 Sequence 검증은 Release에서도 필요하다.

# 정리

Hot Path용 C++ 설계는 문법적 묘기가 아니다.
```text
Strong Domain Types
  → 잘못된 상태를 표현하기 어렵게 만든다.
Explicit Lifetime과 Ownership
  → dangling과 예측하지 못한 해제를 막는다.
Preallocation과 Bounded Work
  → Memory와 실행 시간의 상한을 만든다.
Contiguous Layout
  → CPU가 실제로 접근하는 경로를 단순하게 만든다.
Sanitizer + Optimized Profile
  → 정확성과 성능을 서로 다른 도구로 검증한다.
```
가장 빠른 Code가 아니라 정확한 의미를 유지하며 비용을 측정할 수 있는 Code를 먼저 만든다.
다음 글에서는 그 비용을 왜 평균 하나로 표현하면 안 되는지, Clock과 Histogram을 이용해 Latency를 올바르게 측정하는 방법을 살펴본다.

# 참고 자료

- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [ISO C++ Working Group 문서와 Working Draft](https://www.open-std.org/jtc1/sc22/wg21/docs/standards)
- [GCC Optimize Options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- [Clang AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)
- [Clang UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
- [Clang ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html)
- [Intel 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
