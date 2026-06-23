---
title: '[C언어] Atomic Operation 1. 왜 Atomic이 필요한가'
date: 2026-06-23 20:56:00 +09:00
categories: [Language, C]
tags:
  [
    C,
    atomic,
    concurrency,
    thread,
    race condition
  ]
---

# 개요

C에서 멀티스레드 코드를 작성하다 보면 여러 thread가 같은 값을 함께 읽고 수정하는 상황이 생긴다.

예를 들어 여러 thread가 하나의 counter를 증가시키는 코드를 생각할 수 있다.

```c
counter++;
```

위 코드는 아래와 같은 여러 단계로 나누어 실행된다.

```text
1. counter 값을 메모리에서 읽는다.
2. 읽은 값에 1을 더한다.
3. 더한 값을 다시 counter에 저장한다.
```

문제는 여러 thread가 이 과정을 동시에 실행하면 중간 상태가 서로 겹칠 수 있다는 점이다.

이번 글에서는 C에서 atomic operation이 왜 필요한지 정리한다.

아직 `memory_order`, `compare_exchange`, CPU cache coherence 같은 내용은 깊게 다루지 않는다. 1편의 목표는 atomic을 사용해야 하는 문제 상황을 이해하는 것이다.

# 공유 변수를 증가시키는 코드

먼저 전역 변수 하나를 여러 thread가 증가시키는 예제를 보자.

```c
#include <pthread.h>
#include <stdio.h>

#define THREAD_COUNT 4
#define INCREMENT_COUNT 100000

int counter = 0;

void* increase(void* arg) {
    (void)arg;

    for (int i = 0; i < INCREMENT_COUNT; i++) {
        counter++;
    }

    return NULL;
}

int main(void) {
    pthread_t threads[THREAD_COUNT];

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threads[i], NULL, increase, NULL);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("counter = %d\n", counter);

    return 0;
}
```

기대하는 결과는 단순하다.

```text
4 * 100000 = 400000
```

하지만 실제 실행 결과는 항상 `400000`이라고 보장할 수 없다.

```text
counter = 273841
counter = 318502
counter = 399112
```

실행할 때마다 다른 값이 나올 수 있다.

더 조심해야 할 점은, 어떤 환경에서는 우연히 `400000`이 나올 수도 있다는 것이다. 그렇다고 올바른 코드가 되는 것은 아니다. 동시성 버그는 항상 터지는 버그가 아니라, 특정 타이밍에서만 드러나는 버그인 경우가 많다.

# counter++는 왜 안전하지 않은가

`counter++`는 하나의 C 문장이지만, 실행 관점에서는 read-modify-write 연산이다.

```text
read      현재 counter 값을 읽는다.
modify    읽은 값에 1을 더한다.
write     계산한 값을 다시 저장한다.
```

예를 들어 `counter`가 10인 상태에서 Thread A와 Thread B가 동시에 `counter++`를 실행한다고 해보자.

```text
Thread A: counter 읽기 -> 10
Thread B: counter 읽기 -> 10

Thread A: 10 + 1 계산 -> 11
Thread B: 10 + 1 계산 -> 11

Thread A: counter에 11 저장
Thread B: counter에 11 저장
```

두 thread가 각각 1씩 증가시켰으니 최종 결과는 12가 되어야 한다.

하지만 둘 다 같은 10을 읽고 11을 저장했기 때문에 최종 결과는 11이 된다.

증가 연산 하나가 사라진 것처럼 보인다.

이런 상황을 **lost update**라고 부를 수 있다.

# Race Condition

위 문제는 race condition의 한 예다.

race condition은 실행 결과가 thread의 실행 순서나 타이밍에 따라 달라지는 상황을 말한다.

단일 thread 프로그램에서는 보통 코드의 순서가 실행 결과를 결정한다.

하지만 멀티스레드 프로그램에서는 여러 thread의 실행이 섞인다.

```text
Thread A의 read
Thread B의 read
Thread A의 write
Thread B의 write
```

또는

```text
Thread A의 read
Thread A의 write
Thread B의 read
Thread B의 write
```

어떤 순서로 섞이느냐에 따라 결과가 달라질 수 있다.

공유 자원에 여러 thread가 동시에 접근하고, 그중 하나 이상이 값을 변경한다면 race condition을 의심해야 한다.

# C에서는 Data Race가 더 위험하다

C에서는 이 문제가 단순히 "결과가 조금 틀릴 수 있다" 정도에서 끝나지 않는다.

C 표준의 관점에서, 여러 thread가 같은 객체에 동시에 접근하고 그중 하나 이상이 write를 수행하는데 적절한 동기화가 없다면 data race가 발생한다.

그리고 C에서 data race는 undefined behavior다.

즉 컴파일러와 프로그램 실행 결과에 대해 아무것도 보장할 수 없다.

```c
int counter = 0;

void* increase(void* arg) {
    counter++;
    return NULL;
}
```

여러 thread가 위 코드를 동시에 실행하면 `counter`에 대한 동기화되지 않은 read/write가 발생한다.

이때 문제는 CPU 실행 타이밍뿐만이 아니다.

컴파일러도 "data race가 없는 정상적인 C 프로그램"이라는 전제 아래 최적화를 수행할 수 있다.

그래서 멀티스레드 환경에서 공유 변수에 접근할 때는 컴파일러와 CPU 모두에게 동기화 의도를 알려야 한다.

그 역할을 하는 도구 중 하나가 atomic operation이다.

# Atomic Operation이란

atomic operation은 중간에 쪼개진 상태를 다른 thread가 관찰할 수 없도록 보장되는 연산이다.

즉 atomic한 증가 연산은 아래 세 단계를 하나의 indivisible한 연산처럼 다룬다.

```text
read -> modify -> write
```

다른 thread 입장에서는 연산이 실행되기 전 상태 또는 실행된 후 상태만 볼 수 있다.

중간에 `read는 했지만 아직 write는 안 한 상태` 같은 틈을 이용해서 같은 값을 읽는 문제가 사라진다.

C에서는 C11부터 `<stdatomic.h>`를 통해 표준 atomic 기능을 제공한다.

# Atomic으로 counter 고치기

위 counter 예제를 atomic으로 바꾸면 다음과 같다.

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

#define THREAD_COUNT 4
#define INCREMENT_COUNT 100000

atomic_int counter = 0;

void* increase(void* arg) {
    (void)arg;

    for (int i = 0; i < INCREMENT_COUNT; i++) {
        atomic_fetch_add(&counter, 1);
    }

    return NULL;
}

int main(void) {
    pthread_t threads[THREAD_COUNT];

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threads[i], NULL, increase, NULL);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("counter = %d\n", atomic_load(&counter));

    return 0;
}
```

여기서 핵심은 두 가지다.

```c
atomic_int counter = 0;
```

`counter`를 atomic 타입으로 선언한다.

```c
atomic_fetch_add(&counter, 1);
```

그리고 증가 연산을 atomic 연산으로 수행한다.

이제 여러 thread가 동시에 증가시키더라도 각 증가 연산이 서로 덮어쓰이지 않는다.

최종 결과는 기대한 대로 `400000`이 된다.

# Atomic이 필요한 상황

atomic은 보통 작은 공유 상태를 여러 thread가 동시에 다룰 때 유용하다.

대표적인 예시는 다음과 같다.

- 여러 thread가 함께 증가시키는 counter
- 작업 완료 여부를 나타내는 flag
- reference count
- lock-free 자료구조의 일부 상태
- 한 번만 초기화되었는지 확인하는 상태값

예를 들어 단순 counter는 atomic과 잘 맞는다.

```c
atomic_int request_count = 0;

void on_request(void) {
    atomic_fetch_add(&request_count, 1);
}
```

작업 완료 여부를 표시하는 flag도 atomic을 사용할 수 있다.

```c
#include <stdbool.h>
#include <stdatomic.h>

atomic_bool done = false;

void finish_work(void) {
    atomic_store(&done, true);
}

bool is_done(void) {
    return atomic_load(&done);
}
```

다만 이런 예제에서도 실제 동기화 의미를 정확히 만들려면 memory order를 이해해야 한다.

이 부분은 뒤의 글에서 따로 정리한다.

# Atomic이 모든 문제를 해결하지는 않는다

atomic은 강력하지만 만능은 아니다.

특히 여러 값을 하나의 규칙으로 묶어서 보호해야 한다면 atomic 하나로는 부족할 수 있다.

예를 들어 계좌 이체를 생각해보자.

```text
A 계좌에서 1000원 차감
B 계좌에 1000원 증가
```

이 작업은 두 값이 함께 일관성을 가져야 한다.

`A.balance`만 atomic이고 `B.balance`도 atomic이라고 해서 전체 이체 과정이 자동으로 안전해지는 것은 아니다.

중간에 A에서는 돈이 빠졌지만 B에는 아직 들어가지 않은 상태를 다른 thread가 볼 수도 있다.

이런 경우에는 하나의 critical section으로 묶는 lock이 더 자연스럽다.

```c
pthread_mutex_lock(&mutex);

from->balance -= amount;
to->balance += amount;

pthread_mutex_unlock(&mutex);
```

즉 atomic은 작은 단위의 공유 상태에는 잘 맞지만, 복잡한 불변식을 보호해야 할 때는 lock이 더 적합할 수 있다.

# Atomic과 Lock의 첫 번째 차이

atomic과 lock은 둘 다 동시성 문제를 해결하기 위한 도구다.

하지만 접근 방식이 다르다.

```text
atomic
    특정 변수에 대한 개별 연산을 안전하게 만든다.

lock
    특정 코드 구간을 한 번에 하나의 thread만 실행하게 만든다.
```

counter 증가처럼 연산의 단위가 작고 명확하면 atomic이 간결하다.

```c
atomic_fetch_add(&counter, 1);
```

반대로 여러 줄의 코드가 하나의 의미를 가져야 한다면 lock이 더 읽기 쉽고 안전할 수 있다.

```c
pthread_mutex_lock(&mutex);

validate();
update_state();
write_log();

pthread_mutex_unlock(&mutex);
```

atomic이 lock보다 항상 빠른 것도 아니다.

경쟁이 심한 상황에서는 atomic 연산도 cache line을 계속 주고받으며 비용이 커질 수 있다.

따라서 "lock은 느리고 atomic은 빠르다"가 아니라, 보호하려는 상태의 성격에 맞는 도구를 선택해야 한다.

# 정리

이번 글에서는 C에서 atomic operation이 왜 필요한지 살펴보았다.

핵심은 `counter++` 같은 간단한 코드도 멀티스레드 환경에서는 안전하지 않을 수 있다는 점이다.

```text
counter++는 read -> modify -> write로 나뉜다.
여러 thread가 동시에 실행하면 lost update가 발생할 수 있다.
C에서 동기화 없는 공유 변수 접근은 data race가 될 수 있다.
data race는 undefined behavior다.
atomic operation은 이런 공유 변수 연산을 안전하게 표현하는 방법이다.
```

다음 글에서는 `<stdatomic.h>`의 기본 사용법을 정리한다.

특히 `_Atomic`, `atomic_int`, `atomic_load`, `atomic_store`, `atomic_fetch_add`, `atomic_exchange` 같은 기본 API를 예제로 살펴볼 예정이다.
