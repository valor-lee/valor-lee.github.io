---
title: '[C언어] Atomic Operation 2. stdatomic.h 기본 사용법'
date: 2026-06-24 00:39:00 +09:00
categories: [Language, C]
tags:
  [
    C,
    atomic,
    stdatomic,
    concurrency,
    thread
  ]
---

# 개요

이전 글에서는 C에서 atomic operation이 왜 필요한지 정리했다.

핵심은 `counter++` 같은 단순한 코드도 멀티스레드 환경에서는 안전하지 않을 수 있다는 점이었다.

```c
counter++;
```

위 코드는 실제로는 read, modify, write 단계로 나뉜다.

여러 thread가 동시에 실행하면 lost update가 발생할 수 있고, C에서는 동기화 없는 공유 변수 접근이 data race가 될 수 있다.

이번 글에서는 C11에서 제공하는 `<stdatomic.h>`의 기본 사용법을 정리한다.

아래 내용을 중심으로 본다.

```text
1. atomic 타입 선언하기
2. atomic_load / atomic_store
3. atomic_fetch_add
4. atomic_exchange
5. atomic_compare_exchange_strong / weak
6. 기본 memory order
```

이번 글의 목표는 atomic API를 실제 코드에서 읽고 쓸 수 있게 되는 것이다.

# stdatomic.h

C11부터 표준 라이브러리에는 atomic 연산을 위한 `<stdatomic.h>`가 추가되었다.

```c
#include <stdatomic.h>
```

이 헤더를 사용하면 atomic 타입과 atomic 연산 함수를 사용할 수 있다.

가장 간단한 예시는 다음과 같다.

```c
#include <stdatomic.h>

atomic_int counter = 0;

void increase(void) {
    atomic_fetch_add(&counter, 1);
}
```

`atomic_int`는 atomic하게 접근할 수 있는 `int` 타입이다.

`atomic_fetch_add`는 값을 읽고, 더하고, 다시 저장하는 과정을 atomic하게 수행한다.

# Atomic 타입 선언하기

C에서 atomic 타입을 선언하는 방식은 크게 두 가지가 있다.

## _Atomic 사용하기

첫 번째 방법은 `_Atomic` 타입 지정자를 사용하는 것이다.

```c
#include <stdbool.h>

_Atomic int counter = 0;
_Atomic bool done = false;
```

`_Atomic int`는 "atomic하게 접근해야 하는 int"라는 의미다.

이 방식은 원래 타입이 눈에 잘 보인다는 장점이 있다.

```c
_Atomic long total_bytes = 0;
_Atomic unsigned int flags = 0;
```

## atomic_int 같은 별칭 사용하기

두 번째 방법은 `<stdatomic.h>`에서 제공하는 별칭을 사용하는 것이다.

```c
atomic_int counter = 0;
atomic_bool done = false;
atomic_long total_bytes = 0;
```

자주 쓰는 정수 타입은 이렇게 별칭이 준비되어 있다.

대표적으로 다음과 같은 타입을 볼 수 있다.

```text
atomic_bool
atomic_char
atomic_int
atomic_uint
atomic_long
atomic_ulong
atomic_size_t
```

실제 코드에서는 `atomic_int`, `atomic_bool`, `atomic_size_t`를 자주 보게 된다.

두 방식은 표현만 다를 뿐, atomic 객체를 만든다는 점에서는 같은 목적을 가진다.

```c
_Atomic int a = 0;
atomic_int b = 0;
```

개인적으로는 예제나 간단한 counter에서는 `atomic_int`가 읽기 쉽고, 타입을 더 명확히 드러내고 싶을 때는 `_Atomic`을 쓰는 편이 좋다고 생각한다.

# 초기화하기

atomic 변수도 일반 변수처럼 초기값을 줄 수 있다.

```c
atomic_int counter = 0;
atomic_bool ready = false;
```

동적으로 초기화해야 한다면 `atomic_init`을 사용할 수 있다.

```c
#include <stdatomic.h>

typedef struct {
    atomic_int count;
} Counter;

void counter_init(Counter* counter) {
    atomic_init(&counter->count, 0);
}
```

초기화가 끝난 뒤 여러 thread가 접근하기 시작해야 한다.

atomic 객체라고 해도, 여러 thread가 동시에 초기화하거나 초기화 중인 객체에 접근하는 구조는 피해야 한다.

# atomic_load

atomic 변수의 값을 읽을 때는 `atomic_load`를 사용한다.

```c
atomic_int counter = 10;

int value = atomic_load(&counter);
```

일반 변수라면 아래처럼 읽을 수 있다.

```c
int value = counter;
```

하지만 atomic 변수는 atomic 연산으로 접근하는 습관을 들이는 것이 좋다.

특히 여러 thread가 같은 atomic 객체를 공유한다면, 읽기 역시 atomic하게 수행해야 한다.

예를 들어 현재 요청 수를 조회하는 코드를 생각해볼 수 있다.

```c
#include <stdatomic.h>

atomic_int request_count = 0;

int get_request_count(void) {
    return atomic_load(&request_count);
}
```

`atomic_load`는 값을 읽는 연산이다.

읽기만 한다고 해서 동기화가 필요 없는 것은 아니다. 다른 thread가 값을 쓰고 있다면, 읽는 쪽도 atomic 연산을 사용해야 data race를 피할 수 있다.

# atomic_store

atomic 변수에 값을 저장할 때는 `atomic_store`를 사용한다.

```c
atomic_bool done = false;

void finish(void) {
    atomic_store(&done, true);
}
```

읽는 쪽은 `atomic_load`를 사용한다.

```c
bool is_done(void) {
    return atomic_load(&done);
}
```

전체 예제는 다음과 같다.

```c
#include <stdbool.h>
#include <stdatomic.h>

atomic_bool done = false;

void finish_work(void) {
    atomic_store(&done, true);
}

bool is_finished(void) {
    return atomic_load(&done);
}
```

이런 flag는 atomic의 대표적인 사용처다.

다만 flag를 통해 "다른 데이터의 준비 완료"까지 표현하려면 memory order를 함께 이해해야 한다.

예를 들어 아래처럼 데이터를 쓴 뒤 `ready`를 true로 바꾸는 구조가 있다.

```c
data = 100;
atomic_store(&ready, true);
```

다른 thread가 `ready`를 보고 `data`를 읽는다면, 단순히 atomic bool만 썼다고 모든 문제가 자동으로 해결되는 것은 아니다.

이 부분은 release-acquire ordering과 관련이 있으므로 뒤의 memory ordering 글에서 따로 다룬다.

# atomic_fetch_add

`atomic_fetch_add`는 값을 더하는 atomic read-modify-write 연산이다.

```c
atomic_int counter = 0;

void increase(void) {
    atomic_fetch_add(&counter, 1);
}
```

이 함수는 기존 값을 반환한다.

```c
int old_value = atomic_fetch_add(&counter, 1);
```

예를 들어 `counter`가 10이었다면 다음과 같이 동작한다.

```text
old_value = 10
counter = 11
```

즉 반환값은 증가 후 값이 아니라 증가 전 값이다.

증가 후 값을 사용하고 싶다면 1을 더해주면 된다.

```c
int new_value = atomic_fetch_add(&counter, 1) + 1;
```

여러 thread가 동시에 호출해도 각 thread는 서로 다른 기존 값을 받는다.

```text
Thread A: atomic_fetch_add -> 0 반환, counter는 1
Thread B: atomic_fetch_add -> 1 반환, counter는 2
Thread C: atomic_fetch_add -> 2 반환, counter는 3
```

그래서 `atomic_fetch_add`는 counter뿐 아니라 간단한 sequence number를 만들 때도 사용할 수 있다.

```c
#include <stdatomic.h>

atomic_uint next_id = 1;

unsigned int issue_id(void) {
    return atomic_fetch_add(&next_id, 1);
}
```

반대로 값을 빼고 싶다면 `atomic_fetch_sub`를 사용할 수 있다.

```c
atomic_fetch_sub(&counter, 1);
```

# atomic_exchange

`atomic_exchange`는 atomic하게 값을 교체한다.

```c
atomic_int value = 10;

int old_value = atomic_exchange(&value, 20);
```

동작은 다음과 같다.

```text
old_value = 10
value = 20
```

즉 기존 값을 반환하면서 새 값을 저장한다.

이 함수는 "이전 상태를 확인하면서 상태를 바꾸고 싶을 때" 유용하다.

예를 들어 한 번만 실행되어야 하는 작업을 생각해보자.

```c
#include <stdbool.h>
#include <stdatomic.h>

atomic_bool started = false;

bool try_start(void) {
    bool was_started = atomic_exchange(&started, true);

    return !was_started;
}
```

처음 호출한 thread는 `started`의 기존 값인 `false`를 받는다.

그래서 `try_start()`는 `true`를 반환한다.

그 이후 호출한 thread는 기존 값으로 `true`를 받기 때문에 `false`를 반환한다.

```text
첫 번째 호출: old = false, started = true, return true
두 번째 호출: old = true,  started = true, return false
```

이런 식으로 간단한 one-shot 상태 전환을 만들 수 있다.

# compare_exchange

atomic API 중 처음 볼 때 가장 헷갈리는 함수가 `compare_exchange`다.

대표 함수는 두 가지다.

```c
atomic_compare_exchange_strong
atomic_compare_exchange_weak
```

이 연산은 흔히 CAS라고 부른다.

CAS는 Compare-And-Swap의 약자다.

동작은 이름 그대로다.

```text
현재 값이 내가 기대한 값과 같으면 새 값으로 바꾼다.
현재 값이 기대한 값과 다르면 바꾸지 않는다.
```

예를 들어 아래 상태를 생각해보자.

```c
atomic_int state = 0;
int expected = 0;

bool success = atomic_compare_exchange_strong(&state, &expected, 1);
```

현재 `state`가 0이고 `expected`도 0이면 교체에 성공한다.

```text
success = true
state = 1
expected = 0
```

반대로 현재 `state`가 이미 2였다면 교체에 실패한다.

```text
success = false
state = 2
expected = 2
```

주의할 점은 실패했을 때 `expected` 값이 현재 atomic 객체의 값으로 바뀐다는 것이다.

이 특성 때문에 `compare_exchange`를 반복문에서 사용할 때는 `expected`를 다시 설정해야 하는 경우가 있다.

간단한 예제로 0에서 1로 상태를 바꾸는 함수를 만들어보자.

```c
#include <stdatomic.h>
#include <stdbool.h>

atomic_int state = 0;

bool try_change_state(void) {
    int expected = 0;

    return atomic_compare_exchange_strong(&state, &expected, 1);
}
```

이 함수는 `state`가 0일 때만 1로 바꾼다.

이미 다른 thread가 먼저 1로 바꿨다면 실패한다.

```text
state == 0 -> state를 1로 변경하고 true 반환
state != 0 -> 변경하지 않고 false 반환
```

# strong과 weak의 차이

`atomic_compare_exchange_strong`과 `atomic_compare_exchange_weak`는 비슷하지만 차이가 있다.

`strong`은 비교하는 순간 atomic 객체의 값과 `expected`가 같다면 교체에 성공한다.

반면 `weak`는 값이 같더라도 실패할 수 있다.

이처럼 비교값이 일치하고 다른 thread가 값을 변경한 것도 아닌데 CAS가 실패하는 현상을 **spurious failure**, 즉 허위 실패라고 한다.

```text
실제 실패
    counter != expected
    다른 thread가 먼저 값을 변경했으므로 CAS 실패

spurious failure
    counter == expected
    값은 일치하지만 weak CAS가 실패
```

## spurious failure는 왜 허용할까

일부 CPU는 CAS를 직접 실행하는 명령어 대신 LL/SC 계열의 명령어로 구현한다.

개념적으로는 다음 두 단계로 동작한다.

```text
1. 값을 읽으면서 해당 메모리에 대한 예약을 만든다.
2. 예약이 유지되고 있을 때만 새 값을 저장한다.
```

두 단계 사이에 context switch가 발생하거나, 다른 CPU의 메모리 접근 등으로 예약이 사라지면 값 자체는 변하지 않았어도 저장이 실패할 수 있다.

`weak`는 이런 하드웨어 수준의 실패를 감추기 위해 내부에서 계속 재시도하지 않고 그대로 실패를 반환할 수 있다.

반대로 `strong`은 값이 실제로 다르지 않은데 하드웨어 연산이 실패했다면 구현 내부에서 재시도해 spurious failure가 호출자에게 보이지 않도록 보장한다.

따라서 차이는 atomicity의 강도가 아니다.

```text
strong과 weak 모두 성공한 CAS 연산은 atomic하다.
차이는 weak가 spurious failure를 허용하는지 여부다.
```

## 언제 strong을 사용할까

한 번만 CAS를 시도하고 결과에 따라 바로 분기한다면 `strong`이 적합하다.

```c
bool try_start(void) {
    int expected = 0;

    return atomic_compare_exchange_strong(&state, &expected, 1);
}
```

이 함수가 `false`를 반환하면 `state`가 0이 아니었다고 해석하고 싶다.

여기서 `weak`를 사용하면 `state`가 0인데도 spurious failure 때문에 `false`가 반환될 수 있다. 호출자가 이를 실제 상태 불일치로 오해할 수 있으므로 단발성 시도에는 `strong`이 이해하기 쉽다.

조건부 상태 전환, 한 번만 수행해야 하는 초기화 시도, 성공 여부를 즉시 반환하는 `try_*` 함수가 대표적인 예다.

## 언제 weak를 사용할까

실패하면 어차피 값을 다시 읽고 재시도하는 CAS 루프에서는 `weak`를 사용할 수 있다.

```c
#include <stdatomic.h>

atomic_int counter = 0;

void add_if_non_negative(int value) {
    int current = atomic_load(&counter);

    while (current >= 0) {
        int next = current + value;

        if (atomic_compare_exchange_weak(&counter, &current, next)) {
            return;
        }
    }
}
```

이 코드에서 CAS가 실패하는 이유는 두 가지다.

```text
1. 다른 thread가 counter를 변경해 current와 달라졌다.
2. 값은 같지만 spurious failure가 발생했다.
```

첫 번째 경우에는 실패하면서 `current`가 `counter`의 최신 값으로 갱신된다. 루프는 그 값을 기준으로 `next`를 다시 계산한다.

두 번째 경우에는 `counter`와 `current`가 여전히 같으므로 같은 값으로 한 번 더 시도하면 된다.

```text
실제 값 불일치로 실패
    -> current가 counter의 현재 값으로 갱신됨
    -> next를 다시 계산하여 재시도

spurious failure
    -> 비교값은 여전히 유효함
    -> 루프에서 다시 CAS 시도
```

어느 경우든 루프가 실패를 처리하므로 `weak`의 허위 실패가 최종 결과를 바꾸지 않는다.

일부 플랫폼에서는 `strong`과 `weak`가 동일한 명령어로 구현되어 성능 차이가 없을 수도 있다. 따라서 `weak`가 항상 더 빠르다고 단정하기보다는, **허위 실패를 자연스럽게 재시도할 수 있는 코드인지**를 선택 기준으로 삼는 것이 좋다.

일반적으로 단발성 시도에는 `strong`, 반복문에서는 `weak`를 사용하는 패턴을 볼 수 있다.

다만 처음 atomic을 사용할 때는 `compare_exchange` 자체가 필요한 상황인지 먼저 생각하는 편이 좋다.

단순 증가라면 `atomic_fetch_add`가 더 읽기 쉽다.

단순 값 교체라면 `atomic_exchange`가 더 명확하다.

# 기본 Memory Order

지금까지 사용한 함수들은 memory order를 따로 넘기지 않았다.

```c
atomic_load(&counter);
atomic_store(&done, true);
atomic_fetch_add(&counter, 1);
atomic_exchange(&started, true);
atomic_compare_exchange_strong(&state, &expected, 1);
```

이런 기본 함수들은 기본적으로 `memory_order_seq_cst`를 사용한다.

`seq_cst`는 sequentially consistent의 줄임말이다.

처음에는 가장 보수적이고 이해하기 쉬운 기본값이라고 생각하면 된다.

코드 전체에서 atomic 연산들이 하나의 전역 순서로 실행되는 것처럼 이해할 수 있기 때문이다.

물론 실제 CPU와 컴파일러가 내부적으로 항상 그렇게 단순하게 동작한다는 뜻은 아니다.

하지만 프로그래머가 이해하는 모델로는 가장 직관적이다.

memory order를 명시하고 싶다면 `_explicit` 버전의 함수를 사용한다.

```c
atomic_load_explicit(&counter, memory_order_relaxed);
atomic_store_explicit(&done, true, memory_order_release);
atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
```

이 글에서는 `_explicit` 함수는 깊게 다루지 않는다.

처음에는 기본 atomic 함수로 코드를 명확하게 만들고, 성능이나 동기화 의미가 중요해질 때 memory order를 따로 공부하는 흐름이 좋다.

# atomic API로 의도를 명확히 하기

atomic 객체를 사용할 때 주의할 점이 있다.

같은 객체에 대해 접근 방식이 섞이면 코드를 읽기 어려워진다.

예를 들어 atomic 객체는 아래처럼 단순 대입 형태로 읽을 수도 있다.

```c
atomic_int counter = 0;

void increase(void) {
    atomic_fetch_add(&counter, 1);
}

int get_count(void) {
    return counter;
}
```

`counter`가 atomic 객체이기 때문에 이 접근도 atomic load로 처리될 수 있다.

하지만 코드의 의도를 명확히 하기 위해 atomic 객체에는 atomic API를 사용하는 편이 좋다.

```c
int get_count(void) {
    return atomic_load(&counter);
}
```

특히 프로젝트에서 atomic 접근과 일반 접근이 섞이면 나중에 코드를 읽는 사람이 동기화 의도를 파악하기 어려워진다.

atomic을 쓰기로 했다면 해당 객체의 read/write 경로를 일관되게 atomic 연산으로 표현하는 것이 좋다.

또한 atomic 객체를 억지로 일반 타입 포인터로 변환해서 접근하는 방식은 피해야 한다.

```c
int* raw_counter = (int*)&counter;  // 피해야 하는 방식
```

이런 코드는 atomic 객체를 사용하는 의미를 흐리고, 구현에 따라 예상하기 어려운 문제를 만들 수 있다.

# 언제 어떤 함수를 쓰면 좋을까

기본 API를 용도별로 정리하면 다음과 같다.

```text
값을 읽고 싶다
    -> atomic_load

값을 저장하고 싶다
    -> atomic_store

counter를 증가시키고 싶다
    -> atomic_fetch_add

counter를 감소시키고 싶다
    -> atomic_fetch_sub

기존 값을 얻으면서 새 값으로 바꾸고 싶다
    -> atomic_exchange

현재 값이 기대한 값일 때만 바꾸고 싶다
    -> atomic_compare_exchange_strong

CAS를 반복문에서 사용하고 싶다
    -> atomic_compare_exchange_weak
```

처음에는 `load`, `store`, `fetch_add`만 익숙해져도 많은 atomic 코드를 읽을 수 있다.

`exchange`와 `compare_exchange`는 lock-free 알고리즘이나 상태 전환 코드에서 자주 등장한다.

# 정리

이번 글에서는 `<stdatomic.h>`의 기본 사용법을 정리했다.

핵심은 다음과 같다.

```text
atomic 타입은 _Atomic 또는 atomic_int 같은 별칭으로 선언할 수 있다.
atomic_load는 값을 atomic하게 읽는다.
atomic_store는 값을 atomic하게 저장한다.
atomic_fetch_add는 read-modify-write 증가 연산을 atomic하게 수행한다.
atomic_exchange는 기존 값을 반환하면서 새 값으로 교체한다.
atomic_compare_exchange는 현재 값이 기대한 값일 때만 교체한다.
기본 atomic 함수는 memory_order_seq_cst를 사용한다.
```

atomic API는 단순히 "thread-safe한 변수"를 만드는 문법이 아니다.

어떤 연산을 atomic하게 수행할 것인지 명확하게 표현하는 도구다.

다음 글에서는 atomic operation이 실제로 CPU와 컴파일러 수준에서 어떻게 처리되는지 정리한다.

특히 read-modify-write 명령, cache line, cache coherence, lock-free 여부를 중심으로 살펴볼 예정이다.
