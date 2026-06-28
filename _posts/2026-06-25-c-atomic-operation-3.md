---
title: '[C언어] Atomic Operation 3. Atomic은 실제로 어떻게 동작하는가'
date: 2026-06-25 00:30:00 +09:00
categories: [Language, C]
tags:
  [
    C,
    atomic,
    CPU,
    cache coherence,
    lock-free
  ]
---

# 개요

이전 글에서는 `<stdatomic.h>`의 기본 API를 정리했다.

```c
atomic_load(&value);
atomic_store(&value, 10);
atomic_fetch_add(&counter, 1);
atomic_compare_exchange_weak(&state, &expected, desired);
```

이 함수들은 atomic 연산을 표현하지만, 함수 내부에서 실제로 무슨 일이 일어나는지는 코드만 보고 알기 어렵다.

`atomic_fetch_add`는 정말 하나의 CPU 명령으로 실행될까?

여러 CPU core가 각자의 cache를 가지고 있다면 어떻게 같은 값을 안전하게 변경할까?

atomic 타입이라고 해서 항상 lock-free일까?

이번 글에서는 다음 내용을 중심으로 atomic operation의 실제 동작 방식을 살펴본다.

```text
1. C 표준이 보장하는 atomicity와 실제 구현 방식을 구분한다.
2. 컴파일러가 atomic 연산을 CPU 명령으로 바꾸는 과정을 본다.
3. 여러 core가 cache coherence로 같은 값을 맞춰 보는 방식을 이해한다.
4. atomic 연산이 느려지는 지점과 false sharing을 살펴본다.
5. atomic, lock-free, mutex의 관계를 정리한다.
```

글의 흐름을 한 문장으로 줄이면 이렇다.

```text
C는 atomicity라는 보장을 제공한다.
컴파일러는 그 보장을 target CPU 명령으로 낮춘다.
CPU는 cache line 단위의 coherence로 여러 core의 접근을 조정한다.
그 과정에서 경합, 정렬, cache line 배치에 따라 비용이 달라진다.
그래서 atomic과 mutex는 성능만이 아니라 보호할 불변식의 범위로 선택해야 한다.
```

# 1. C의 보장과 실제 구현은 다르다

## 1.1 C 표준이 보장하는 것

atomic을 이해할 때는 **언어가 보장하는 동작**과 **그 보장을 구현하는 방법**을 구분해야 한다.

C 코드에서 atomic 연산을 사용하면 C의 abstract machine 관점에서 해당 연산은 나뉘지 않는 하나의 연산으로 취급된다.

예를 들어 다음 코드를 보자.

```c
int old = atomic_fetch_add(&counter, 1);
```

여러 thread가 동시에 실행하더라도 각 연산은 하나씩 순서가 정해진 것처럼 관찰된다.

```text
counter = 10

Thread A: 10을 반환하고 counter를 11로 변경
Thread B: 11을 반환하고 counter를 12로 변경
```

두 thread가 모두 10을 읽고 11을 저장하는 lost update는 발생하지 않는다.

## 1.2 C 표준이 보장하지 않는 것

하지만 C 표준이 "반드시 특정 CPU 명령 하나를 사용하라"고 요구하는 것은 아니다.

구현 방법은 target CPU, 데이터 타입, 정렬 상태, 컴파일러와 컴파일 옵션에 따라 달라질 수 있다.

```text
C가 보장하는 것
    -> 연산의 atomicity와 memory ordering 규칙

컴파일러와 하드웨어가 결정하는 것
    -> CPU 명령, fence, 재시도 loop, runtime library, 내부 lock
```

따라서 다음 두 문장은 같은 의미가 아니다.

```text
이 연산은 atomic하다.
이 연산은 하나의 CPU 명령으로 실행된다.
```

첫 번째는 C 프로그램에서 요구할 수 있는 보장이다.

두 번째는 특정 환경에서 선택될 수 있는 구현 방식이다.

# 2. 컴파일러는 atomic을 CPU 명령으로 바꾼다

## 2.1 일반 증가와 atomic 증가

프로그래머가 작성한 C 코드는 바로 CPU에서 실행되지 않는다.

```text
C source
    -> compiler intermediate representation
    -> target CPU instruction
    -> CPU와 cache coherence protocol이 실행
```

일반 변수의 증가와 atomic 증가를 비교해보자.

```c
void normal_add(int* value) {
    (*value)++;
}

void atomic_add(atomic_int* value) {
    atomic_fetch_add_explicit(value, 1, memory_order_relaxed);
}
```

두 함수 모두 C 코드에서는 값을 1 증가시킨다.

하지만 컴파일러는 `normal_add`를 다른 core와의 동기화를 요구하지 않는 일반 memory 연산으로 처리할 수 있다.

반면 `atomic_add`는 다른 thread와 동시에 실행되어도 read-modify-write 전체가 atomic해야 한다.

그래서 컴파일러는 target CPU가 제공하는 atomic 명령이나 명령 조합을 선택한다.

## 2.2 x86-64의 read-modify-write

Clang으로 위 `atomic_fetch_add`를 x86-64 대상으로 최적화하여 컴파일하면 다음과 같은 형태의 명령을 볼 수 있다.

```asm
movl    $1, %eax
lock xaddl %eax, (%rdi)
```

실제 출력은 컴파일러와 옵션에 따라 달라질 수 있지만, 핵심은 `lock xadd`다.

`xadd`는 register의 값과 memory의 값을 교환하면서 더하는 read-modify-write 명령이다.

여기에 `lock` prefix가 붙으면 다른 processor가 같은 memory 위치에 동시에 접근하는 상황에서도 해당 read-modify-write가 atomic하게 실행되도록 한다.

개념적으로는 다음 세 단계를 한 연산처럼 만든다.

```text
memory에서 기존 값 읽기
기존 값에 1 더하기
결과를 memory에 저장하기
```

여기서 `lock`이라는 이름 때문에 매번 system bus 전체를 잠근다고 생각하기 쉽다.

하지만 현대 x86 CPU에서 cache 가능한 일반 memory를 다룰 때는 보통 cache coherence protocol을 통해 해당 cache line에 대한 배타적인 소유권을 확보하는 방식으로 처리한다.

여기서 "더 비싼 방식"은 일반적인 cache line 소유권 조정만으로 끝나는 경우보다 비용이 큰 처리를 뜻한다.

예를 들어 접근 주소가 정렬되어 있지 않거나 하나의 atomic 접근이 cache line 경계를 걸치면 CPU가 한 cache line만 배타적으로 확보해서 처리하기 어려울 수 있다.

그런 경우에는 더 넓은 범위의 bus lock, microcode 처리, 여러 cache line에 대한 조정, 또는 compiler/runtime의 보조 경로가 필요할 수 있다.

여기서 비교 대상은 **정렬된 주소의 단일 cache line에 대해 hardware atomic 명령이 cache coherence만으로 처리되는 일반적인 fast path**다.

즉 `lock` prefix는 소스 코드의 mutex와 같은 lock 객체를 생성한다는 의미가 아니다.

CPU가 해당 memory read-modify-write의 atomicity를 보장하도록 지시하는 명령의 일부다.

## 2.3 ARM64의 LSE와 exclusive loop

ARM64도 atomic 연산을 지원하지만 x86-64와 같은 명령 형태만 사용하는 것은 아니다.

ARMv8.1-A의 LSE(Large System Extensions)를 사용할 수 있는 환경에서는 atomic add가 다음과 같은 명령으로 만들어질 수 있다.

```asm
mov     w8, #1
ldadd   w8, w0, [x0]
```

`ldadd`는 memory의 기존 값을 읽고 값을 더하는 atomic read-modify-write 명령이다.

LSE를 사용하지 않는 환경에서는 load-exclusive와 store-exclusive를 이용한 loop가 사용될 수 있다.

개념적인 형태는 다음과 같다.

```asm
retry:
    ldxr    w1, [x0]
    add     w2, w1, #1
    stxr    w3, w2, [x0]
    cbnz    w3, retry
```

각 명령의 역할은 다음과 같다.

```text
ldxr
    값을 읽고 해당 memory 위치에 대한 exclusive reservation을 만든다.

stxr
    reservation이 유지되고 있을 때만 저장에 성공한다.

cbnz
    저장에 실패했다면 다시 시도한다.
```

`ldxr`와 `stxr` 사이에 다른 core가 관련 memory를 변경하면 `stxr`가 실패할 수 있다.

그러면 loop는 최신 값을 다시 읽고 연산을 재시도한다.

이 구조는 이전 글에서 `atomic_compare_exchange_weak`의 spurious failure를 설명할 때 살펴본 LL/SC 계열의 동작과 연결된다.

중요한 점은 x86-64와 ARM64의 명령 형태가 달라도 C 프로그램이 요구한 atomicity를 만족해야 한다는 것이다.

## 2.4 Atomic load와 store

atomic 연산이라고 해서 항상 복잡한 read-modify-write 명령이 필요한 것은 아니다.

```c
int load_value(atomic_int* value) {
    return atomic_load_explicit(value, memory_order_relaxed);
}

void store_value(atomic_int* value, int next) {
    atomic_store_explicit(value, next, memory_order_relaxed);
}
```

대상 CPU가 해당 크기와 정렬의 load, store를 원래 atomic하게 처리할 수 있다면 일반 load, store 명령이 사용될 수 있다.

ARM64에서는 위 relaxed 연산이 다음처럼 컴파일될 수 있다.

```asm
ldr     w0, [x0]
str     w1, [x0]
```

이때 `ldr`와 `str`가 평범해 보인다고 해서 C의 atomic 의미가 사라진 것은 아니다.

컴파일러는 이 접근을 atomic 객체에 대한 연산으로 알고 있으며, C memory model이 허용하지 않는 방식으로 없애거나 합치지 않아야 한다.

memory order가 달라지면 명령도 달라질 수 있다.

예를 들어 ARM64의 acquire load와 release store에는 다음과 같은 명령이 사용될 수 있다.

```asm
ldapr   w0, [x0]
stlr    w1, [x0]
```

target architecture에 적절한 명령이 없다면 compiler barrier나 CPU fence가 추가될 수도 있다.

atomicity와 ordering은 관련되어 있지만 같은 개념은 아니다.

```text
atomicity
    -> 하나의 연산이 중간 상태로 관찰되지 않게 한다.

ordering
    -> 해당 연산 전후의 다른 memory 연산이 관찰되는 순서를 제한한다.
```

memory ordering은 다음 글에서 별도로 다룬다.

# 3. 여러 core는 cache coherence로 값을 맞춘다

## 3.1 변수보다 중요한 단위는 cache line

현대 CPU의 각 core는 일반적으로 자신과 가까운 cache를 가지고 있다.

두 core가 같은 변수 `counter`를 읽었다고 해보자.

```text
Core A cache: counter가 포함된 cache line
Core B cache: counter가 포함된 cache line
Main memory:  counter가 포함된 data
```

여기서 중요한 단위는 변수 하나가 아니라 **cache line**이다.

cache는 보통 memory에서 `int` 하나만 가져오는 것이 아니라, 그 주변 data를 포함한 고정 크기 block을 가져온다.

이 block을 cache line이라고 한다.

cache line의 크기는 architecture와 CPU에 따라 다르지만 64 byte인 시스템을 흔히 볼 수 있다.

## 3.2 Cache coherence가 하는 일

각 core가 같은 cache line의 복사본을 가지고 있다면 한 core의 변경을 다른 core가 알아야 한다.

이를 위해 여러 core의 cache 상태를 일관되게 관리하는 cache coherence protocol이 사용된다.

구체적인 protocol과 상태 이름은 CPU마다 다르지만, 개념적으로 다음과 같은 일이 일어난다.

```text
1. Core A가 counter를 변경하려 한다.
2. Core A는 counter가 들어 있는 cache line의 쓰기 권한을 얻는다.
3. 다른 core가 가진 같은 cache line의 복사본은 무효화되거나 갱신 대상이 된다.
4. Core A가 값을 변경한다.
5. Core B가 이후 해당 값을 읽으려면 유효한 최신 cache line을 다시 가져온다.
```

atomic read-modify-write는 이 coherence mechanism과 CPU의 atomic 명령을 이용해 다른 core가 같은 cache line을 동시에 수정하지 못하도록 조정한다.

따라서 atomic 연산은 단순히 register 내부에서 빠르게 계산하고 끝나는 연산이 아니다.

같은 cache line을 여러 core가 경쟁하면 core 사이에 cache line의 소유권과 coherence message가 오갈 수 있다.

## 3.3 두 core가 동시에 증가시키면

두 core가 같은 atomic counter를 증가시키는 상황을 단순화해보자.

초기값은 10이다.

```text
Core A: atomic_fetch_add(&counter, 1)
Core B: atomic_fetch_add(&counter, 1)
```

가능한 동작 흐름 중 하나는 다음과 같다.

```text
1. Core A가 cache line의 배타적인 쓰기 권한을 얻는다.
2. Core A가 10을 읽고 11을 저장한다.
3. Core A의 atomic 연산이 완료된다.
4. Core B가 같은 cache line의 쓰기 권한을 얻는다.
5. Core B가 11을 읽고 12를 저장한다.
6. Core B의 atomic 연산이 완료된다.
```

결과는 12다.

두 연산의 실제 시작 시점은 겹칠 수 있지만, 같은 atomic 객체에 대한 modification은 하나의 순서로 관찰된다.

ARM의 exclusive loop처럼 구현되었다면 한 core의 저장이 실패하고 다시 시도할 수도 있다.

```text
Core A: 10을 exclusive load
Core B: 10을 exclusive load
Core A: 11 저장 성공
Core B: 11 저장 시도 실패
Core B: 최신 값 11을 다시 읽음
Core B: 12 저장 성공
```

어느 방식이든 성공한 atomic 증가 두 번이 사라져서는 안 된다.

# 4. Atomic이 느려지는 지점

## 4.1 같은 atomic 객체에 경합이 집중될 때

atomic은 mutex보다 가벼운 도구라고 자주 설명된다.

작은 counter 하나를 변경하는 상황에서는 실제로 mutex의 lock과 unlock으로 critical section을 만드는 것보다 간결하고 빠를 수 있다.

하지만 atomic이라는 이름이 공짜라는 뜻은 아니다.

여러 core가 같은 atomic 변수를 계속 변경하면 해당 cache line이 core 사이를 반복해서 이동할 수 있다.

```text
Core A가 쓰기 권한 획득
    -> Core B의 복사본 무효화

Core B가 쓰기 권한 획득
    -> Core A의 복사본 무효화

Core A가 다시 쓰기 권한 획득
    -> 다시 coherence traffic 발생
```

이런 cache line 이동을 흔히 cache line ping-pong이라고 표현한다.

CAS loop를 사용한다면 경합 때문에 compare-exchange 실패와 재시도도 늘어난다.

따라서 thread 수를 늘릴수록 하나의 전역 atomic counter가 항상 선형적으로 빨라지는 것은 아니다.

경합이 심한 counter는 thread별 local counter로 나누고 나중에 합치는 방식이 더 나을 수 있다.

```text
하나의 global atomic counter
    -> 모든 thread가 같은 cache line을 경쟁

thread-local counter
    -> 각 thread가 자신의 counter를 변경
    -> 필요한 시점에 결과를 합산
```

atomic을 선택할 때는 연산 하나의 비용뿐 아니라 여러 core가 같은 memory 위치를 얼마나 자주 변경하는지도 봐야 한다.

## 4.2 서로 다른 변수도 같은 cache line에 있으면 느려질 수 있다

서로 다른 atomic 변수라도 같은 cache line에 들어 있다면 성능 문제가 생길 수 있다.

```c
typedef struct {
    atomic_int first;
    atomic_int second;
} Counters;
```

Thread A는 `first`만 변경하고 Thread B는 `second`만 변경한다고 해보자.

논리적으로 두 변수는 독립적이다.

하지만 두 변수가 같은 cache line에 있다면 coherence는 변수 단위가 아니라 cache line 단위로 동작한다.

```text
Thread A가 first 변경
    -> first와 second가 포함된 cache line의 쓰기 권한 필요

Thread B가 second 변경
    -> 같은 cache line의 쓰기 권한 필요
```

결국 실제 data를 공유하지 않는데도 같은 cache line을 서로 가져오며 경쟁할 수 있다.

이를 **false sharing**이라고 한다.

성능 측정에서 이런 현상이 확인된다면 자주 변경되는 값을 서로 다른 cache line에 배치하는 방법을 고려할 수 있다.

다만 무조건 padding을 추가하면 memory 사용량과 cache 효율이 나빠질 수 있다.

먼저 profile과 benchmark로 실제 병목인지 확인하는 것이 중요하다.

# 5. Atomic과 lock-free는 같은 말이 아니다

## 5.1 Atomic 타입이 항상 lock-free인 것은 아니다

atomic 타입을 사용했다고 해서 항상 hardware atomic 명령만 사용된다고 보장할 수는 없다.

target CPU가 특정 크기의 atomic 연산을 직접 지원하지 않으면 compiler runtime이 helper function을 호출할 수 있다.

구현은 필요한 보장을 제공하기 위해 내부 lock을 사용할 수도 있다.

```text
atomic
    -> C가 보장하는 연산의 의미

lock-free
    -> 해당 구현이 연산을 수행할 때 lock에 의존하지 않는 특성
```

즉 atomic과 lock-free는 동의어가 아니다.

C에서는 기본 atomic 타입의 lock-free 특성을 macro로 확인할 수 있다.

```c
#include <stdatomic.h>
#include <stdio.h>

int main(void) {
    printf("atomic int: %d\n", ATOMIC_INT_LOCK_FREE);
    printf("atomic long: %d\n", ATOMIC_LONG_LOCK_FREE);
    printf("atomic pointer: %d\n", ATOMIC_POINTER_LOCK_FREE);

    return 0;
}
```

각 macro의 값은 다음 의미를 가진다.

```text
0
    해당 타입의 atomic 연산은 lock-free가 아니다.

1
    일부 객체에서는 lock-free이고 일부 객체에서는 아닐 수 있다.

2
    해당 타입의 atomic 연산은 항상 lock-free다.
```

특정 atomic 객체는 `atomic_is_lock_free`로 확인할 수 있다.

```c
#include <stdatomic.h>
#include <stdio.h>

int main(void) {
    atomic_int counter = 0;

    if (atomic_is_lock_free(&counter)) {
        printf("counter is lock-free\n");
    } else {
        printf("counter is not lock-free\n");
    }

    return 0;
}
```

이 결과는 architecture, ABI, compiler와 객체의 타입 및 정렬 조건에 따라 달라질 수 있다.

한 개발 환경에서 lock-free였다는 결과를 모든 target에 그대로 가정하면 안 된다.

## 5.2 Lock-free가 항상 빠르거나 wait-free인 것도 아니다

lock-free는 구현과 progress를 이해하는 데 중요한 특성이지만, 그 자체로 성능을 보장하지는 않는다.

경합이 심한 CAS loop는 계속 실패하며 CPU 시간을 사용할 수 있다.

```c
int current = atomic_load(&value);

while (!atomic_compare_exchange_weak(&value, &current, current + 1)) {
    // 실패하면 current가 갱신되고 다시 시도한다.
}
```

다른 thread가 계속 값을 변경하면 한 thread는 여러 번 재시도할 수 있다.

시스템 전체에서는 누군가가 계속 성공하더라도 특정 thread 하나는 오랫동안 성공하지 못할 수 있다.

또한 다음 용어도 구분해야 한다.

```text
lock-free
    -> 전체적으로는 어떤 thread가 유한한 단계 안에 진행한다.

wait-free
    -> 각 thread가 유한한 단계 안에 자신의 연산을 완료한다.
```

wait-free가 lock-free보다 더 강한 progress 보장이다.

단순히 atomic API를 사용했다는 이유만으로 작성한 알고리즘 전체가 lock-free나 wait-free가 되는 것도 아니다.

자료구조가 어떤 progress guarantee를 가지는지는 전체 알고리즘과 사용한 모든 연산을 함께 분석해야 한다.

# 6. Atomic과 mutex는 어떻게 연결되는가

atomic과 mutex는 서로 완전히 분리된 세계가 아니다.

일반적인 mutex 구현도 빠른 경로에서는 atomic compare-and-exchange 같은 연산으로 lock 상태를 변경할 수 있다.

```text
mutex 획득 시도
    -> atomic 연산으로 unlocked에서 locked 상태 변경

경합이 없음
    -> user space에서 빠르게 성공

경합이 있음
    -> spin, 대기 queue 또는 OS system call 사용 가능
```

반대로 앞에서 본 것처럼 일부 C atomic 타입은 target에서 lock-free가 아니어서 내부 lock을 사용할 수 있다.

따라서 다음과 같이 단순하게 나누면 정확하지 않다.

```text
atomic = hardware만 사용
mutex  = operating system만 사용
```

실제 구현은 hardware atomic, compiler runtime, spinning, scheduler와 operating system의 대기 기능을 조합할 수 있다.

프로그래머 입장에서 더 중요한 차이는 표현할 수 있는 범위다.

```text
atomic
    -> 한 객체에 대한 load, store, read-modify-write와 상태 전환

mutex
    -> 여러 객체와 여러 단계의 작업을 하나의 critical section으로 보호
```

atomic이 내부적으로 어떻게 구현되는지 아는 것은 성능을 이해하는 데 도움이 된다.

하지만 코드에서 atomic과 mutex를 선택할 때는 먼저 보호해야 하는 불변식의 범위를 봐야 한다.

# 7. 실무에서 확인할 점

atomic 연산을 사용할 때는 다음 항목을 확인하는 것이 좋다.

## 7.1 정말 하나의 값만 보호하면 되는가

counter, flag, reference count처럼 하나의 값에 대한 독립적인 연산이라면 atomic과 잘 맞는다.

여러 값이 함께 바뀌어야 한다면 mutex가 더 자연스러울 수 있다.

## 7.2 target에서 lock-free여야 하는가

signal handler나 실시간 처리처럼 내부 lock 사용 가능성 자체가 문제가 되는 환경이라면 `atomic_is_lock_free`와 target 문서를 확인해야 한다.

## 7.3 같은 atomic 객체에 경합이 집중되는가

여러 core가 하나의 변수만 계속 갱신하면 coherence traffic이 병목이 될 수 있다.

## 7.4 false sharing이 있는가

서로 독립적인 값이라도 같은 cache line에 배치되면 성능이 영향을 받을 수 있다.

## 7.5 정렬과 객체의 lifetime이 올바른가

atomic 객체는 정상적인 타입 선언과 초기화 방식을 사용하고, 일반 타입 pointer로 강제 변환해 접근하지 않는 것이 좋다.

## 7.6 실제 assembly와 benchmark를 확인했는가

성능이 중요한 코드는 예상만으로 판단하기보다 compiler explorer, `clang -S`, `gcc -S`, profiler와 benchmark로 target 환경의 결과를 확인해야 한다.

# 정리

이번 글에서는 C atomic operation이 compiler와 CPU 수준에서 어떻게 구현될 수 있는지 살펴봤다.

핵심은 다음과 같다.

```text
C의 atomic 보장과 실제 CPU 구현 방식은 구분해야 한다.
atomic 연산이 반드시 하나의 CPU 명령이라는 보장은 없다.
x86-64에서는 lock prefix가 붙은 read-modify-write 명령을 사용할 수 있다.
ARM64에서는 LSE 명령이나 exclusive load/store loop를 사용할 수 있다.
여러 core의 atomic 접근은 cache coherence를 통해 조정된다.
같은 atomic 객체에 경합이 집중되면 cache line 이동과 재시도 비용이 커진다.
서로 다른 변수도 같은 cache line에 있으면 false sharing이 발생할 수 있다.
atomic 타입이 항상 lock-free인 것은 아니다.
lock-free 여부는 macro와 atomic_is_lock_free로 확인할 수 있다.
lock-free라고 해서 항상 빠르거나 wait-free인 것은 아니다.
```

atomic은 "CPU가 알아서 한 번에 처리해주는 빠른 연산"이라고만 이해하면 부족하다.

언어의 memory model, compiler의 명령 선택, CPU의 atomic instruction, cache coherence가 함께 동작해 하나의 보장을 만든다.

다음 글에서는 `memory_order_relaxed`, `acquire`, `release`, `acq_rel`, `seq_cst`를 중심으로 atomicity와 memory ordering의 차이를 정리한다.

# 참고 자료

- [C11 draft N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [GCC - Built-in Functions for Memory Model Aware Atomic Operations](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)
- [Arm Architecture Reference Manual - Exclusive Access Instructions](https://developer.arm.com/documentation/ddi0487/latest/)
- [Intel 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
