---
title: '[Low Latency Trading] CPU Pipeline·Out-of-Order·Branch Prediction과 ABI'
date: 2026-08-08 00:10:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    CPU pipeline,
    out-of-order execution,
    branch prediction,
    ABI
  ]
---

# 개요

초저지연 Trading System의 Hot Path는 짧아 보인다.

```text
Packet Decode
  → Order Book Update
  → Strategy Decision
  → Risk Check
  → Order Encode
```

하지만 C++ statement 하나와 CPU가 수행하는 일 하나는 일대일로 대응하지 않는다. Compiler는 source를 바꾸어 배치하고, CPU는 여러 instruction을 겹쳐 실행하며, 잘못 예측한 경로의 작업을 버리기도 한다. 함수 호출 경계에서는 ABI가 argument, register 보존과 stack 모양을 정한다. 따라서 다음과 같은 단정은 성능 설명이 될 수 없다.

```text
한 줄이므로 빠르다.
branch가 있으므로 느리다.
instruction 하나이므로 1 cycle이다.
함수를 inline하면 항상 빨라진다.
```

이 글에서는 source code에서 시작해 assembly와 PMU counter까지 내려가며 이 단정들을 검증하는 방법을 정리한다.

# 선수 글과 BFS 위치

이 글의 BFS 위치는 **Level 2-H — CPU와 Memory 심화**다. 먼저 다음 글의 용어를 알고 있으면 좋다.

1. [[CPU] Memory Access와 Cache Hierarchy](/posts/cpu-memory-cache-hierarchy/)
2. [[Memory] Virtual Memory와 TLB](/posts/virtual-memory-tlb/)
3. [[Computer Architecture] Endianness와 Alignment](/posts/endianness-alignment/)
4. [[Memory] NUMA 구조와 Local·Remote Memory](/posts/numa-memory-architecture/)
5. [[Low Latency Trading] Hot Path를 위한 Modern C++ 설계 원칙](/posts/low-latency-cpp-hot-path/)

이 글은 Cache와 TLB를 다시 설명하기보다, 준비된 instruction과 data를 core가 어떻게 진행시키는지에 집중한다. 다음 글에서 core 밖의 coherence fabric, memory controller와 PCIe 경로를 연결한 뒤 data 구조에 적용한다.

- [[Low Latency Trading] CPU Interconnect와 Uncore — Core·Memory·PCIe 연결 경로](/posts/low-latency-cpu-interconnect-uncore/)
- [[Low Latency Trading] Cache-Friendly Data Layout과 Memory Pool](/posts/low-latency-cache-layout-memory-pool/)

# 1. 먼저 세 개의 층을 분리한다

성능을 분석할 때 source, ISA와 microarchitecture를 섞지 않아야 한다.

```text
C++ Source
  → Compiler Optimization
ISA Instruction
  → Decode와 내부 변환
Micro-operation과 Execution Resource
```

C++ abstract machine은 program의 observable behavior를 정의한다. ISA는 machine code가 사용할 register, instruction과 memory model을 정의한다. Microarchitecture는 특정 CPU가 그 ISA를 어떻게 구현하는지 정한다. 같은 x86-64 binary도 서로 다른 CPU model에서 다른 front-end, execution port, cache와 predictor를 만날 수 있다. 같은 CPU에서도 compiler, option, link-time optimization과 입력 분포가 바뀌면 실행 경로가 달라진다. 그러므로 이 글의 pipeline 그림은 개념 지도이지 모든 CPU의 정확한 block diagram이 아니다.

# 2. Pipeline의 개념적 단계

현대 CPU core를 다음처럼 단순화할 수 있다.

```text
Fetch
  → Decode
  → Rename / Allocate
  → Schedule / Issue
  → Execute
  → Complete
  → Retire
```

실제 CPU는 일부 단계를 합치거나 더 세분화한다. 한 cycle에 처리할 수 있는 폭도 단계마다 다르다. Pipeline의 목적은 instruction 하나를 반드시 더 짧게 만드는 것이 아니다. 서로 다른 instruction의 단계를 겹쳐 전체 throughput을 높이는 데 있다.

## 2.1 Fetch

Fetch 단계는 instruction byte를 instruction cache 쪽에서 가져온다. 다음 fetch address는 아직 branch 결과가 나오지 않았더라도 필요하다. Branch predictor가 다음 program counter를 추측하는 이유다. 다음 문제는 front-end가 instruction 공급을 늦출 수 있다.

- Instruction cache miss
- TLB miss
- 복잡하거나 긴 instruction decode
- Branch target을 제때 찾지 못함
- 큰 code footprint로 인한 code locality 저하

Data가 L1 data cache에 있어도 instruction 공급이 막히면 execution unit은 놀 수 있다.

## 2.2 Decode

Decode 단계는 ISA instruction을 CPU 내부에서 처리할 operation으로 해석한다. x86-64 구현에서는 instruction 하나가 하나 이상의 micro-operation, 즉 uop으로 변환될 수 있다. 반대로 특정 instruction 조합이 합쳐져 더 적은 uop처럼 처리되는 구현도 있다. 따라서 다음 등식은 일반적으로 성립하지 않는다.

```text
Source statement 수 = ISA instruction 수 = uop 수 = cycle 수
```

구현에 따라 decoded uop을 별도 cache에서 재사용하기도 한다. 이 세부 사항은 CPU model별 optimization manual과 측정으로 확인해야 한다.

## 2.3 Rename와 Allocate

Source와 ISA는 제한된 이름의 architectural register를 사용한다. CPU는 이를 더 많은 physical register에 연결할 수 있다. 이를 register renaming이라고 한다. Renaming은 이름만 같아서 생기는 거짓 dependency를 줄인다.

```text
WAW: 이전 write와 다음 write의 이름이 같음
WAR: 이전 read와 다음 write의 이름이 같음
RAW: 다음 operation이 이전 결과를 실제로 읽음
```

WAW와 WAR는 physical register를 다르게 배정해 제거할 수 있다. RAW는 값 자체가 필요하므로 renaming만으로 없앨 수 없다. Allocate 단계에서는 in-flight operation을 추적할 resource도 예약한다. 대표적으로 reorder buffer, scheduler entry와 load/store 관련 entry가 있다. 이 resource는 유한하다. 긴 cache miss를 다른 작업으로 숨기려 해도 독립 작업과 추적 공간이 부족하면 결국 진행이 멈춘다.

## 2.4 Schedule와 Issue

Out-of-order core는 program 순서의 다음 instruction만 고집하지 않는다. Operand와 execution resource가 준비된 operation을 골라 issue할 수 있다.

```text
Program order:  A → B → C → D

A: 먼 memory load를 기다림
B: A 결과가 필요함
C: A와 독립적인 integer add
D: C 결과가 필요함

가능한 진행: C와 D를 먼저 실행, A가 오면 B 실행
```

이것이 out-of-order execution의 핵심이다. 아무 operation이나 순서를 바꾸는 것은 아니다. Data dependency, memory ordering, exception과 architectural behavior를 지켜야 한다.

## 2.5 Execute와 Complete

Issued operation은 적절한 execution unit에서 실행된다. 예를 들면 integer ALU, branch unit, vector unit과 address generation unit이 있다. Operation이 사용할 수 있는 unit과 port 수는 microarchitecture마다 다르다. 결과가 만들어지면 dependent operation이 그 결과를 사용할 수 있다. 하지만 결과가 계산되었다는 사실과 architectural state에 최종 반영되었다는 사실은 다르다.

## 2.6 Retire

CPU는 보통 program order에 맞춰 완료된 instruction을 retire한다. Retirement는 speculative한 결과를 architectural state로 확정하는 경계로 이해할 수 있다. 앞선 instruction에서 exception이나 branch misprediction이 발견되면 잘못된 경로의 operation은 retire하지 않는다. 이 구조는 out-of-order execution과 precise exception을 함께 가능하게 한다.

# 3. Reorder Buffer를 queue 크기 숫자로만 보지 않는다

Reorder buffer, 즉 ROB는 실행 중인 operation의 program order와 상태를 추적하는 핵심 구조다. 개념적으로 다음 역할을 한다.

- 완료 여부 추적
- Program order retirement
- Exception과 fault의 정확한 지점 유지
- Branch misprediction 뒤 잘못된 경로 복구
- 일부 register 상태 복구에 필요한 정보 관리

정확한 entry 수와 내부 구현은 CPU model별로 다르다. 공개된 숫자가 있더라도 source instruction 수와 단순 비교하면 안 된다. 한 instruction이 여러 uop이 될 수 있고, 구현에 따라 추적 단위도 다를 수 있기 때문이다. ROB가 크면 모든 memory latency가 사라지는 것도 아니다.

```text
Latency를 겹쳐 숨기려면
  독립적인 다음 작업
  + 충분한 ROB와 scheduler 공간
  + 충분한 load/store 추적 공간
  + 필요한 execution bandwidth
가 함께 있어야 한다.
```

# 4. Dependency와 Instruction-Level Parallelism

Instruction-Level Parallelism, 즉 ILP는 동시에 진행할 수 있는 독립 operation의 양이다. 다음 loop는 하나의 accumulator가 이전 iteration 결과를 계속 요구한다.

```cpp
#include <cstddef>
#include <cstdint>
#include <span>

std::uint64_t serial_sum(std::span<const std::uint32_t> values) noexcept {
    std::uint64_t sum = 0;
    for (const auto value : values) {
        sum += value;
    }
    return sum;
}

std::uint64_t four_way_sum(std::span<const std::uint32_t> values) noexcept {
    std::uint64_t a = 0;
    std::uint64_t b = 0;
    std::uint64_t c = 0;
    std::uint64_t d = 0;

    std::size_t i = 0;
    for (; i + 3 < values.size(); i += 4) {
        a += values[i];
        b += values[i + 1];
        c += values[i + 2];
        d += values[i + 3];
    }
    for (; i < values.size(); ++i) {
        a += values[i];
    }
    return a + b + c + d;
}
```

두 번째 함수에는 독립적인 accumulator가 네 개 있다. Compiler와 CPU가 여러 add를 겹치기 쉬울 수 있다. 그러나 source만 보고 두 번째가 빠르다고 결론 내리면 안 된다. Compiler가 첫 번째 loop도 vectorize하거나 여러 accumulator로 바꿀 수 있다. 입력 크기가 작으면 tail 처리와 code size가 더 큰 비중을 차지할 수 있다. Memory bandwidth가 병목이면 add dependency를 줄여도 결과가 변하지 않을 수 있다. 정수형을 `std::uint64_t`로 둔 이유도 중요하다. Unsigned overflow는 modulo arithmetic으로 정의되지만 signed overflow는 Undefined Behavior가 될 수 있다. Trading domain에서 modulo 합이 맞다는 뜻은 아니므로 실제 notional 계산에는 별도의 overflow policy가 필요하다.

## 4.1 Latency와 Throughput

Instruction latency는 결과가 dependent operation에 사용 가능해질 때까지의 지연과 관련된다. Reciprocal throughput은 독립적인 같은 종류의 operation을 얼마나 자주 시작할 수 있는지와 관련된다. 두 숫자는 같지 않을 수 있다. 긴 dependency chain은 latency에 민감하다. 독립 operation이 많으면 execution bandwidth와 throughput에 더 민감할 수 있다. Trading request 한 건의 end-to-end latency와 instruction table의 latency도 같은 개념이 아니다.

## 4.2 ILP와 Memory-Level Parallelism

여러 독립 load miss를 동시에 진행할 수 있는 성질은 Memory-Level Parallelism, 즉 MLP로 따로 구분하면 유용하다. ILP가 있어도 모든 load가 하나의 pointer chain이라면 다음 address를 미리 알 수 없다.

```text
node = node->next
```

현재 node를 읽어야 다음 node address를 알 수 있으므로 dependency가 memory access 사이를 직렬화한다. 이 문제는 다음 글의 flat layout과 pointer chasing benchmark로 이어진다.

# 5. Load Queue와 Store Queue

Load와 store는 단순 ALU operation보다 ordering 문제가 더 많다. CPU는 load queue와 store queue 같은 구조로 in-flight memory operation을 추적한다. 구현에 따라 load/store queue를 합쳐 Load-Store Queue, 즉 LSQ라고 부르기도 한다. 이 구조들은 다음 동작에 관여할 수 있다.

- 아직 retire하지 않은 load와 store 추적
- 이전 store와 이후 load의 address 관계 확인
- 조건을 만족할 때 store data를 load로 전달
- Memory ordering violation 탐지와 replay
- Cache miss를 기다리는 operation 관리

## 5.1 Memory Disambiguation

다음 두 pointer가 같은 address인지 compiler나 CPU가 즉시 확정하지 못할 수 있다.

```cpp
#include <cstdint>

void update(std::uint64_t* out, const std::uint64_t* in) noexcept {
    *out = 7;
    const auto value = *in;
    *out = value + 1;
}
```

`out`과 `in`이 alias할 수 있으므로 memory operation의 순서는 observable result에 영향을 준다. CPU는 address가 확인되기 전 추측해 진행할 수 있지만 틀리면 관련 operation을 다시 실행해야 할 수 있다. Compiler에 거짓 alias 정보를 주어 최적화를 강제하면 성능 문제가 correctness 문제로 바뀐다.

## 5.2 Store-to-Load Forwarding

이전 store가 쓴 값을 이후 load가 읽는다면 cache에 완전히 반영되기를 기다리지 않고 내부 buffer에서 전달할 수 있다. 하지만 size, alignment와 address overlap 형태에 따라 forwarding이 원활하지 않거나 replay가 필요할 수 있다. 정확한 조건은 target CPU 문서와 benchmark로 확인한다. `reinterpret_cast`로 packed wire data를 억지로 읽는 것은 이 최적화를 얻기 위한 안전한 방법이 아니다. Object lifetime, alignment와 aliasing 규칙을 먼저 지켜야 한다.

# 6. Speculation은 확정 전에 일을 시작하는 방법이다

Out-of-order CPU는 미래를 완전히 알 수 없으므로 예측을 이용한다. 대표적인 예가 control-flow speculation과 memory dependency speculation이다. Speculative operation이 실행되었다고 해서 결과가 반드시 retire하는 것은 아니다. 예측이 틀리면 architectural state에 반영되지 않은 잘못된 경로의 작업을 버리고 올바른 지점에서 다시 시작한다. 다만 speculative execution은 security 관점도 가진다. Architectural state에 남지 않은 작업도 cache 같은 microarchitectural state에 흔적을 남길 수 있다. 성능 최적화와 speculative-execution 보안 완화 정책은 별개로 검토해야 한다.

# 7. Branch Prediction

Conditional branch의 조건은 pipeline 뒤쪽에서 계산될 수 있다. 그때까지 fetch를 멈추면 front-end가 자주 비게 된다. Predictor는 과거 실행과 branch 위치 등의 정보를 사용해 방향과 target을 예상한다.

```text
Prediction correct
  → 이미 가져온 경로를 계속 진행

Prediction wrong
  → 잘못된 경로의 speculative work 폐기
  → 올바른 target에서 fetch 재시작
```

Misprediction 비용을 모든 CPU에서 같은 cycle 숫자로 외우지 않는다. Pipeline 깊이, front-end, branch 종류, cache 상태와 뒤따르는 dependency가 비용을 바꾼다.

## 7.1 예측 가능한 branch와 불규칙한 branch

다음 조건의 source 모양은 같아도 입력 분포가 다르면 결과가 달라진다.

```cpp
#include <cstdint>
#include <span>

struct QuoteSample {
    std::uint32_t value;
    bool active;
};

std::uint64_t sum_active(std::span<const QuoteSample> samples) noexcept {
    std::uint64_t sum = 0;
    for (const auto& sample : samples) {
        if (sample.active) {
            sum += sample.value;
        }
    }
    return sum;
}
```

`active`가 긴 구간 동안 같은 값이면 predictor가 규칙을 배우기 쉬울 수 있다. 매 iteration마다 독립적으로 무작위에 가깝다면 방향 예측이 어려울 수 있다. 그렇다고 source에 `if`가 보인다는 사실만으로 실제 conditional branch가 존재한다고 단정할 수 없다. Compiler가 conditional move, mask 또는 vector instruction으로 바꿀 수 있다. 반대로 branchless source가 assembly에서는 branch를 만들 수도 있다.

## 7.2 Branchless가 항상 빠르지는 않다

Branchless 변환은 misprediction을 피할 수 있지만 다음 비용을 만들 수 있다.

- 선택하지 않을 값까지 load하거나 계산함
- 더 긴 data dependency chain
- 더 많은 instruction과 register pressure
- 큰 code footprint
- Fault 가능성이 있는 address를 불필요하게 읽음

예측이 매우 잘 되는 branch라면 조건부 실행보다 branch가 더 적은 일을 할 수 있다. 두 version을 실제 message 분포로 비교해야 한다. Synthetic random input 하나만으로 production의 burst, phase와 symbol별 편향을 대신하지 않는다.

# 8. Direct Call, Indirect Call과 Return

Direct call은 target address가 machine code에 직접 표현되거나 link 단계에서 결정되는 호출이다. Compiler는 target function을 알면 다음 최적화를 시도하기 쉽다.

- Inlining
- Constant propagation
- Dead argument 제거
- 호출 전후 code 재배치

Indirect call은 function pointer, virtual dispatch나 jump table처럼 runtime value로 target이 정해질 수 있다.

```text
Direct call:    call known_function
Indirect call:  call address_in_register
```

Indirect call도 target이 계속 같으면 CPU가 잘 예측할 수 있다. Link-time optimization이나 devirtualization으로 direct call이 될 수도 있다. 따라서 “virtual function은 무조건 느리다”는 결론은 지나치게 단순하다. 문제는 target의 안정성, inlining 기회, code layout과 실제 miss rate다. Return은 많은 CPU에서 return-address 예측용 구조의 도움을 받을 수 있다. 그러나 지나치게 깊거나 비정상적인 call/return 패턴, context 변화와 구현 세부 사항에 따라 예측이 흔들릴 수 있다.

# 9. ABI는 function 경계의 binary 계약이다

ABI, 즉 Application Binary Interface는 별도로 compile한 code가 binary 수준에서 서로 호출되도록 규칙을 정한다. ABI가 다룰 수 있는 항목은 다음과 같다.

- Argument와 return value 전달 위치
- Caller-saved와 callee-saved register
- Stack alignment와 frame 규칙
- Aggregate type 분류와 layout
- Symbol name, relocation과 object file 형식
- Exception unwinding 정보

C++ language standard만으로 특정 register 번호가 정해지지는 않는다. Target triple, operating system, compiler ABI와 type에 따라 규칙이 달라진다.

## 9.1 x86-64 System V의 단순한 integer argument

Linux의 일반적인 x86-64 user-space C/C++ 호출을 이해할 때 System V AMD64 psABI가 기준이 된다. 단순 INTEGER class argument 여섯 개는 보통 다음 순서로 전달된다.

```text
1: RDI
2: RSI
3: RDX
4: RCX
5: R8
6: R9
7 이후: ABI 분류와 stack 위치 확인
```

다음 함수를 x86-64 Linux target으로 별도 compile하면 일곱 번째 단순 integer argument의 stack 접근을 관찰할 수 있다.

```cpp
#include <cstdint>

extern "C" std::uint64_t combine7(
    std::uint64_t a,
    std::uint64_t b,
    std::uint64_t c,
    std::uint64_t d,
    std::uint64_t e,
    std::uint64_t f,
    std::uint64_t g) noexcept {
    return a + b + c + d + e + f + g;
}
```

이 표를 모든 type에 그대로 적용하면 안 된다. Floating-point와 vector argument는 보통 XMM register 쪽을 사용하지만 aggregate, variadic function과 복합 return type은 ABI의 type classification 절차를 따라야 한다. 작은 struct도 field 구성에 따라 여러 class로 나뉘거나 memory로 전달될 수 있다.

## 9.2 Caller-saved와 Callee-saved

System V AMD64에서 일반적인 general-purpose register 보존 관계를 단순화하면 다음과 같다.

```text
Caller-saved:
  RAX, RCX, RDX, RSI, RDI, R8, R9, R10, R11

Callee-saved:
  RBX, RBP, R12, R13, R14, R15

RSP:
  ABI 규칙에 맞게 유지
```

Caller는 caller-saved 값이 호출 뒤에도 필요하면 스스로 보존해야 한다. Callee는 callee-saved register를 바꾸면 원래 값을 복원한 뒤 return해야 한다. Register 저장과 복원이 실제로 필요한지는 compiler가 만든 전체 function을 봐야 한다. 호출이 inline되면 이 경계 자체가 사라질 수 있다.

System V AMD64의 vector register도 호출 경계에서 보존된다고 임의로 가정하지 말고 psABI의 volatile 규칙과 type classification을 함께 확인한다.

## 9.3 Stack Alignment와 Red Zone

System V AMD64 호출 규칙에서 caller는 `call` 직전 stack을 16-byte boundary에 맞춘다. `call`이 8-byte return address를 push하므로 일반적인 callee entry에서는 다음 관계를 관찰한다.

```text
(RSP + 8) mod 16 = 0
```

Function prologue가 stack을 다시 조정할 수 있다. User-space System V AMD64에는 현재 `RSP` 아래 128 byte를 leaf function이 사용할 수 있는 red zone 규칙도 있다. Kernel code, interrupt context와 다른 ABI에서는 같은 가정을 그대로 쓰면 안 된다. Compiler option으로 red zone 사용을 끄는 환경도 있다.

## 9.4 다른 ABI를 경계한다

Windows x64는 첫 integer argument register, 32-byte shadow space와 register 보존 규칙이 System V와 다르다. AArch64의 AAPCS64는 일반적인 argument에 `x0`부터 `x7`, SIMD와 floating-point argument에 `v0`부터 `v7`을 사용하며 별도의 보존 규칙을 가진다. Stack pointer의 16-byte alignment 규칙도 해당 ABI 문서에서 확인해야 한다. 이 목록은 예외를 포함한 완전한 ABI 요약이 아니다. Assembly를 읽기 전에 다음을 먼저 기록한다.

```text
Compiler와 version
Target triple
Operating system
Optimization option
ISA extension option
LTO/PGO 여부
```

# 10. Inlining의 이득과 Code Footprint

Inlining은 call과 return instruction만 없애는 최적화가 아니다. Caller 문맥이 callee 내부에 보이면 더 큰 이득이 생길 수 있다.

```text
Constant argument
  → 조건 제거
  → 불필요한 load와 store 제거
  → 더 작은 specialized code
```

반면 큰 function을 여러 call site에 복제하면 text size가 커진다. 그 결과 instruction cache, decoded-uop cache와 branch-target 관련 구조에 더 큰 압력을 줄 수 있다. Hot loop 안에서 드물게 실행되는 error handling까지 inline하면 hot code의 locality가 나빠질 수 있다. 작은 wrapper조차 register pressure나 compile 결과에 따라 예상과 다른 결과를 만들 수 있다. `inline` keyword는 optimizer에게 반드시 inline하라는 명령이 아니다. Compiler-specific `always_inline` 속성도 성능 보증이 아니다. Profile-guided optimization, link-time optimization과 hot/cold code layout을 함께 실험한다.

# 11. SIMD는 선택지이지 출발점이 아니다

SIMD는 하나의 instruction으로 여러 lane의 data를 처리할 수 있다. 다음 조건에서는 유용할 가능성이 있다.

- 같은 연산을 많은 element에 적용함
- Data가 연속적임
- Iteration 사이 dependency가 적음
- Batch 처리 지연이 허용됨

반면 message 한 건씩 이어지는 control-heavy path에서는 이득이 작을 수 있다. Gather와 scatter, tail 처리, data 재배치와 불필요한 lane 계산이 비용을 만든다. Batch를 기다려야 한다면 throughput은 좋아져도 첫 message latency는 나빠질 수 있다. 일부 vector instruction의 power와 frequency 영향은 CPU 세대와 instruction width에 따라 다르므로 target system에서 확인한다. Scalar baseline을 먼저 정확하게 만들고 compiler vectorization report와 assembly를 확인한다.

# 12. Compiler의 As-If Rule과 Undefined Behavior

C++ compiler는 observable behavior가 같다면 program을 자유롭게 변환할 수 있다. 이를 흔히 as-if rule이라고 부른다. Source line의 실행 순서, local variable의 존재와 특정 load의 발생 시점 자체가 항상 observable behavior인 것은 아니다. 따라서 benchmark 결과를 사용하지 않으면 계산 전체가 제거될 수 있다. `volatile`은 일반적인 동시성 도구나 성능 barrier가 아니다. Benchmark framework가 제공하는 결과 소비와 optimization barrier를 사용하고 생성된 assembly를 확인한다.

## 12.1 Undefined Behavior는 optimizer와 맺은 계약을 깨뜨린다

대표적인 Undefined Behavior는 다음과 같다.

- Signed integer overflow
- Array bounds 밖 접근
- Lifetime이 끝난 object 참조
- 잘못된 alignment의 object 접근
- 허용되지 않은 aliasing
- C++ memory model에서의 data race

Debug build에서 우연히 원하는 결과가 나왔다는 사실은 안전성을 증명하지 않는다. Sanitizer build, unit test와 release assembly 확인은 서로 다른 역할을 한다. Fast path를 위해 safety check를 지웠다면 입력 경계에서 같은 invariant가 반드시 보장되는지 증명해야 한다.

# 13. Source에서 PMU까지 검증한다

성능 가설은 다음 사슬로 검증한다.

```text
Source와 invariant
  → Release build
  → Assembly와 code layout
  → Representative workload
  → End-to-end latency distribution
  → PMU counter
  → 가설 수정
```

## 13.1 Exact Build를 남긴다

예를 들어 별도 translation unit을 assembly로 만들 수 있다.

```bash
clang++ -std=c++20 -O3 -DNDEBUG -S -masm=intel sum.cpp -o sum.s
```

`-masm=intel`은 x86 계열에서 사용하는 표기 선택이다. AArch64 target이나 다른 compiler에서는 맞는 option을 사용한다. Binary 전체의 call target과 layout은 disassembler로 본다.

```bash
objdump -drC ./bench
```

다음 항목을 확인한다.

1. 측정하려는 function이 실제로 남아 있는가?
2. Loop가 unroll 또는 vectorize되었는가?
3. Conditional branch가 남았는가?
4. Call이 direct, indirect 또는 inline 중 무엇인가?
5. Spill과 reload가 생겼는가?
6. Error path가 hot block 사이에 섞였는가?

Assembly 한 장면만 보고 runtime frequency를 알 수는 없다. 어떤 path가 얼마나 실행되는지는 profile과 입력 기록이 필요하다.

## 13.2 PMU Counter를 보조 증거로 쓴다

Linux `perf stat`으로 기본 event를 반복 측정할 수 있다.

```bash
perf stat -r 20 \
  -e cycles,instructions,branches,branch-misses,cache-misses \
  ./bench
```

Event 이름과 의미는 CPU vendor, model과 kernel 지원에 따라 달라진다. 원하는 hardware event를 사용할 수 없으면 multiplexing되거나 지원되지 않을 수 있다. 다음 정보를 보고서에 함께 남긴다.

- CPU model과 microcode
- Kernel, compiler와 binary hash
- Event의 정확한 이름
- Run 횟수와 workload seed
- CPU affinity와 frequency policy
- Counter multiplexing 여부

`cycles / instructions`로 계산한 CPI는 실행 구간의 평균 비율이다. Instruction 하나가 각각 그 수의 cycle을 사용했다는 뜻이 아니다. IPC가 높아도 한 request의 tail latency가 나쁠 수 있다. 반대로 memory를 기다리는 작은 request는 IPC가 낮아도 business requirement를 만족할 수 있다.

## 13.3 PMU의 한계

PMU counter는 원인을 자동으로 말해주지 않는다. Branch miss가 많아도 전체 시간에서 차지하는 비중이 작을 수 있다. Cache miss event도 어느 cache level, demand/prefetch, speculative/retired 중 무엇을 세는지 정의가 다르다. Sampling은 skid 때문에 정확한 source line과 약간 떨어진 위치를 가리킬 수 있다. 하나의 counter를 최종 판결로 사용하지 말고 latency, assembly와 여러 counter를 연결한다. 측정 환경 설계는 [[Low Latency Trading] Latency 측정과 Tail 분석](/posts/low-latency-measurement/)을 함께 참고한다.

# 14. 실습 1 — Dependency Chain 비교

첫 번째 실습은 `serial_sum`과 `four_way_sum`을 비교한다.

## 14.1 가설

```text
독립 accumulator가 늘면 add dependency chain이 짧아져
충분히 큰 cache-resident input에서 throughput이 좋아질 수 있다.
```

## 14.2 통제할 조건

- 같은 값과 같은 element 수
- 정답 비교
- 같은 compiler option
- Data를 L1, LLC와 memory working set으로 각각 구성
- Warm-up과 측정 iteration 분리
- CPU affinity 고정
- 결과가 제거되지 않도록 benchmark framework 사용

## 14.3 관찰할 것

- Assembly의 unroll과 vectorization
- Instructions와 cycles
- Cache miss 변화
- Element당 시간
- p50뿐 아니라 반복 간 분산

Compiler가 두 version을 같은 형태로 만들었다면 source 차이가 사라진 것이다. 그 결과도 유효한 결론이다.

# 15. 실습 2 — Branch 분포 비교

`sum_active`의 input을 세 가지로 만든다.

```text
A: active가 항상 true
B: true가 긴 구간으로 뭉침
C: true/false가 seed 고정 난수로 섞임
```

Data 생성은 측정 구간 밖에서 수행한다. 각 input의 true 개수와 최종 합이 동일하도록 구성하면 계산량 비교가 더 명확하다. Branch version과 compiler가 만든 conditional-move 또는 vectorized version을 비교한다. 관찰 항목은 다음과 같다.

- `branches`
- `branch-misses`
- Instructions와 cycles
- 전체 latency distribution
- Assembly의 실제 control flow

Production trace의 비밀 정보와 고객 data는 benchmark에 그대로 복사하지 않는다. 분포 특성을 보존한 synthetic trace나 승인된 replay data를 사용한다.

# 16. 실습 3 — Call Boundary와 ABI 확인

`combine7`을 별도 source file에 두고 caller와 분리해 compile한다. 먼저 LTO 없이 build해 ABI 경계를 관찰한다. 다음에는 LTO를 켜고 call이 inline 또는 constant-fold되는지 비교한다. 확인할 질문은 다음과 같다.

1. 첫 여섯 integer argument는 어느 register에 있는가?
2. 일곱 번째 argument는 어디에서 읽는가?
3. Caller가 호출 전후 보존하는 값은 무엇인가?
4. Callee가 callee-saved register를 실제로 사용하는가?
5. Stack pointer는 호출 경계에서 어떻게 정렬되는가?
6. LTO 뒤 function symbol과 call이 남아 있는가?

이 실습을 Windows x64 또는 AArch64에서 반복하면 ABI를 source language와 분리해 이해할 수 있다.

# 17. Trading Hot Path 분석 순서

실제 handler를 최적화할 때 다음 순서를 사용한다.

```text
1. Correctness invariant와 입력 분포 고정
2. End-to-end latency와 tail baseline 수집
3. Hot function과 call path 확인
4. Release assembly에서 실제 branch/load/call 확인
5. Dependency와 working set 가설 작성
6. 관련 PMU event로 보조 증거 수집
7. 한 가지 변경 적용
8. 정답, overload와 tail regression 재검증
```

평균이 줄어도 다음 항목이 나빠지면 배포 후보가 아닐 수 있다.

- p99.9와 max
- Packet drop과 sequence gap
- Queue depth와 backpressure
- Risk check correctness
- CPU 사용률과 sibling workload 간섭
- Capacity 초과 시 동작

# 18. 완료 기준

다음 질문에 답하고 실험 결과를 남기면 이 글의 학습을 완료한 것으로 본다.

1. Fetch, decode, rename, issue, execute와 retire를 서로 구분할 수 있는가?
2. Register renaming이 WAR/WAW는 줄여도 RAW dependency는 없애지 못하는 이유를 설명할 수 있는가?
3. ROB가 out-of-order execution과 in-order retirement를 어떻게 연결하는가?
4. Load/store queue가 필요한 이유와 memory disambiguation을 설명할 수 있는가?
5. Branch misprediction을 보편적인 한 cycle 숫자로 말하면 안 되는 이유를 설명할 수 있는가?
6. Direct call과 indirect call의 실제 assembly를 찾을 수 있는가?
7. System V AMD64의 단순 integer argument와 caller/callee-saved 규칙을 확인할 수 있는가?
8. Inlining이 instruction cache에 불리할 수 있는 경우를 말할 수 있는가?
9. Source, assembly, PMU와 latency를 하나의 가설로 연결할 수 있는가?
10. `cycle = instruction`이 아닌 이유를 설명할 수 있는가?

Level 2-H 산출물에는 최소한 다음을 포함한다.

- CPU와 compiler가 명시된 benchmark repository
- Branch 분포별 latency와 PMU 결과
- Dependency-chain 비교의 assembly diff
- Direct/indirect call과 ABI register를 표시한 disassembly
- 틀린 가설과 수정된 결론까지 포함한 짧은 보고서

# 19. 자주 생기는 오해

## 19.1 Out-of-order CPU는 program 결과도 순서를 바꾼다

CPU는 dependency와 architectural rule을 지키며 내부 실행을 겹친다. 일반적으로 retirement를 program order로 관리해 precise state를 제공한다. C++ thread 사이 ordering은 별도의 language memory model과 synchronization 규칙으로 다뤄야 한다.

## 19.2 Branch 하나는 항상 비싸다

잘 예측된 branch는 잘못 예측된 branch와 비용 구조가 다르다. Branchless version도 추가 계산, load와 dependency를 만들 수 있다.

## 19.3 Instruction 하나는 1 cycle이다

한 instruction은 여러 uop이 될 수 있고 latency와 throughput도 다르다. 여러 instruction이 같은 cycle에 겹쳐 진행되거나 하나가 여러 cycle의 dependency를 만들 수 있다.

## 19.4 IPC가 높으면 request latency가 반드시 낮다

IPC는 측정 구간의 aggregate metric이다. Queueing, cache miss, kernel preemption과 rare slow path가 tail latency를 지배할 수 있다.

## 19.5 Indirect call은 항상 mispredict한다

Target이 안정적이면 예측될 수 있고 optimizer가 devirtualize할 수도 있다. Target 분포와 실제 assembly를 확인해야 한다.

## 19.6 `inline`을 붙이면 code가 반드시 inline된다

C++의 `inline`은 One Definition Rule과도 관련된 language specifier이며 강제 성능 명령이 아니다. 최종 결정과 결과는 compiler, option과 call context에 달려 있다.

## 19.7 ABI 표를 알면 모든 argument 위치를 안다

Aggregate, vector, variadic function과 return type은 별도의 classification 규칙을 가진다. Operating system과 architecture가 바뀌면 ABI도 달라질 수 있다.

## 19.8 PMU event 이름이 같으면 모든 CPU에서 의미도 같다

Event encoding과 counted condition은 model-specific할 수 있다. Vendor manual과 `perf list`에서 현재 system의 정의를 확인한다.

# 20. 다음 학습 순서

이 글에서 core 내부의 instruction 흐름을 살펴봤다. 이제 data가 그 core에 어떤 순서와 모양으로 도착하는지 연결한다.

1. [[Low Latency Trading] CPU Interconnect와 Uncore — Core·Memory·PCIe 연결 경로](/posts/low-latency-cpu-interconnect-uncore/)
2. [[Low Latency Trading] Cache-Friendly Data Layout과 Memory Pool](/posts/low-latency-cache-layout-memory-pool/)
3. Bounded SPSC ring buffer와 backpressure
4. NIC RX/TX ring, RSS, NAPI와 busy polling
5. Feed Handler와 Order Book의 end-to-end profiling

특히 다음 글에서는 contiguous array, AoS와 SoA, pointer chasing, false sharing과 allocation 경로를 benchmark한다.

# 정리

Pipeline은 여러 instruction의 단계를 겹쳐 throughput을 높인다. Register renaming은 거짓 dependency를 줄이고, scheduler는 준비된 operation을 먼저 issue할 수 있게 한다. ROB는 speculative한 out-of-order 실행을 program-order retirement와 연결한다. Load/store queue는 memory dependency와 ordering을 추적한다. Branch predictor는 front-end를 계속 움직이게 하지만 잘못된 예측은 이미 진행한 작업을 버리게 한다. ABI는 function argument, register 보존과 stack을 정하는 target별 binary 계약이다. Inlining과 SIMD는 유용한 선택지지만 code footprint, data layout과 실제 workload를 함께 봐야 한다. 가장 중요한 원칙은 다음과 같다.

```text
Source를 믿고 끝내지 않는다.
Assembly로 compiler의 결정을 확인한다.
PMU를 latency 결과의 보조 증거로 사용한다.
정확한 CPU, ABI와 workload 안에서만 결론을 말한다.
```

# 참고 자료

- [Intel® 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html)
- [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Intel VTune Profiler: Instructions Retired Event](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-0/instructions-retired-event.html)
- [Intel: Refined Speculative Execution Terminology](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/refined-speculative-execution-terminology.html)
- [AMD Software Optimization Guide for AMD EPYC Processors](https://docs.amd.com/v/u/en-US/56305)
- [System V Application Binary Interface AMD64 Architecture Processor Supplement](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Arm AAPCS64](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [ISO C++ Current Status](https://www.open-std.org/jtc1/sc22/wg21/docs/standards)
- [Linux perf-stat Manual](https://man7.org/linux/man-pages/man1/perf-stat.1.html)
