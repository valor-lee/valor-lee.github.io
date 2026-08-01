---
title: '[C언어] Atomic Operation 4. Memory Order 이해하기'
date: 2026-06-26 00:30:00 +09:00
categories: [Language, C]
tags:
  [
    C,
    atomic,
    memory order,
    acquire release,
    happens-before
  ]
---

# 개요

이전 글에서는 C의 atomic operation이 compiler와 CPU에서 어떻게 구현되는지 살펴봤다.

하지만 atomic 연산을 사용할 때는 연산 자체가 나뉘지 않는다는 것만으로 충분하지 않을 수 있다.

다음처럼 한 thread가 데이터를 준비하고, 다른 thread가 준비 완료 여부를 확인하는 상황을 생각해보자.

```c
int data = 0;
atomic_bool ready = false;

// Thread A
data = 42;
atomic_store(&ready, true);

// Thread B
if (atomic_load(&ready)) {
    printf("data = %d\n", data);
}
```

`ready`의 load와 store는 atomic하다.

그렇다면 Thread B가 `true`를 읽었을 때 `data == 42`도 보장될까?

이 질문에 답하려면 atomicity와 memory ordering을 구분해야 한다.

이번 글에서는 다음 내용을 중심으로 C의 memory order를 정리한다.

```text
1. atomicity와 ordering의 차이
2. happens-before와 synchronizes-with
3. memory_order_relaxed
4. memory_order_release와 memory_order_acquire
5. memory_order_acq_rel
6. memory_order_seq_cst
7. memory_order_consume과 fence
8. 연산별로 사용할 수 있는 memory order
```

글 전체의 핵심은 다음과 같다.

```text
Atomicity
    -> 해당 atomic 객체의 연산이 중간에 찢어지지 않게 한다.

Memory ordering
    -> 그 atomic 연산의 앞뒤에 있는 다른 memory 접근이
       thread 사이에서 어떤 순서로 관찰되는지를 정한다.
```

# 1. Atomicity와 ordering의 차이

## 1.1 Atomicity의 범위

atomic 연산은 해당 atomic 객체에 대한 접근을 안전하게 만든다.

```c
atomic_int counter = 0;

atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
```

여러 thread가 이 코드를 동시에 실행해도 `counter`의 증가는 lost update 없이 처리된다.

여기서 `memory_order_relaxed`를 사용해도 read-modify-write 자체의 atomicity는 유지된다.

즉 memory order를 약하게 지정한다고 atomic 연산이 일반 연산으로 바뀌는 것은 아니다.

## 1.2 Ordering의 범위

문제는 atomic 객체 주변에 일반 변수가 함께 있을 때다.

```c
int data = 0;
atomic_bool ready = false;
```

Thread A는 `data`를 쓴 뒤 `ready`를 변경하고, Thread B는 `ready`를 확인한 뒤 `data`를 읽는다.

```text
Thread A                         Thread B

data = 42                       ready 읽기
ready = true                    data 읽기
```

우리가 원하는 의미는 다음과 같다.

```text
Thread B가 ready == true를 관찰했다면
Thread A가 그 전에 수행한 data = 42도 관찰해야 한다.
```

이 관계는 `ready`가 atomic이라는 사실만으로 자동으로 만들어지지 않는다.

두 thread 사이에 적절한 memory ordering이 필요하다.

# 2. Happens-before 관계

## 2.1 관계의 종류와 범위

C memory model의 관계를 이해할 때는 각 관계가 어디까지 연결되는지 먼저 구분해야 한다.

| 관계 | 적용 범위 | 의미 |
|---|---|---|
| `sequenced-before` | 하나의 thread 내부 | C abstract machine에서 두 평가의 순서를 나타낸다. |
| `modification order` | 하나의 atomic 객체 | 해당 객체에 수행된 모든 modification의 일관된 순서를 나타낸다. |
| `reads-from` | 특정 store와 load | load가 어느 store의 값을 읽었는지 나타낸다. |
| `synchronizes-with` | 서로 다른 thread | release와 acquire 같은 synchronization operation을 연결한다. |
| `happens-before` | thread 내부와 thread 사이 | sequenced-before와 synchronizes-with 등을 전이적으로 연결한 논리적 선후 관계다. |

`reads-from`은 load가 어느 store의 값을 읽었는지 설명하기 위해 이 글에서 사용하는 표현이다.

중요한 점은 `reads-from` 자체가 synchronization을 의미하지 않는다는 것이다.

```text
relaxed store  ------ reads-from ------>  relaxed load

synchronizes-with 관계는 만들어지지 않음
```

반면 acquire load가 대응하는 release store의 값을 읽으면 synchronizes-with가 만들어진다.

```text
release store  ---- synchronizes-with ---->  acquire load
```

각 관계의 역할을 짧게 정리하면 다음과 같다.

```text
sequenced-before
    -> 같은 thread 안의 언어상 순서

modification order
    -> atomic 객체 하나의 변경 순서

reads-from
    -> load가 어떤 store의 값을 읽었는지 나타내는 연결

synchronizes-with
    -> synchronization operation이 만드는 thread 사이의 연결

happens-before
    -> sequenced-before와 synchronizes-with 등을 연결해 도출하는 선후 관계
```

따라서 다음 두 상황은 서로 다르다.

```text
relaxed load가 relaxed store의 값을 읽음
    -> reads-from 관계는 존재
    -> synchronizes-with는 없음

acquire load가 release store의 값을 읽음
    -> reads-from 관계가 존재
    -> synchronizes-with도 존재
```

## 2.2 Thread 내부의 언어 순서

먼저 같은 thread 안에서 앞의 평가가 뒤의 평가보다 먼저 순서 지어져 있으면 `sequenced-before` 관계가 있다.

```c
data = 42;  // (1)
atomic_store_explicit(&ready, true, memory_order_release);  // (2)
```

이 코드에서는 `(1)`이 `(2)`보다 sequenced-before다.

```text
data = 42
    |
    | sequenced-before
    v
release store: ready = true
```

여기서 sequenced-before는 CPU가 반드시 두 명령을 이 순서대로 실행한다는 뜻이 아니다.

sequenced-before는 C abstract machine에서 두 평가 사이에 정의되는 **언어 수준의 관계**다.

```text
sequenced-before
    != 실제 CPU 명령의 실행 순서
    != 다른 core가 두 연산을 관찰하는 순서
```

compiler는 C memory model에서 허용되는 결과를 바꾸지 않는 범위에서 명령을 재배치할 수 있다.

CPU 역시 out-of-order execution과 store buffer 등을 사용할 수 있다.

따라서 소스 코드에 sequenced-before가 있더라도, 그 관계가 항상 같은 기계 명령 순서나 다른 core의 관찰 순서로 나타나는 것은 아니다.

sequenced-before는 한 thread 안의 관계만 설명한다.

Thread A의 연산과 Thread B의 연산을 연결하려면 thread 사이의 동기화 관계가 필요하다.

## 2.3 Thread 사이의 동기화

Thread A가 `release store`로 값을 저장하고 Thread B의 `acquire load`가 그 값을 읽으면, 두 atomic 연산 사이에 `synchronizes-with` 관계가 만들어진다.

```c
// Thread A
data = 42;
atomic_store_explicit(&ready, true, memory_order_release);

// Thread B
if (atomic_load_explicit(&ready, memory_order_acquire)) {
    printf("data = %d\n", data);
}
```

관계를 그림으로 나타내면 다음과 같다.

```text
Thread A                                      Thread B

data = 42
    |
    | sequenced-before
    v
release store: ready = true  ---------------->  acquire load: ready
                              synchronizes-with         |
                                                        | sequenced-before
                                                        v
                                                    data 읽기

data 쓰기  ---------------------------------------->  data 읽기
                          happens-before
```

Thread A의 `data = 42`는 release store보다 sequenced-before다.

release store는 그 값을 읽은 acquire load와 동기화된다.

acquire load 뒤에는 Thread B의 `data` 읽기가 있다.

이 관계들을 연결하면 Thread A의 `data` 쓰기가 Thread B의 `data` 읽기보다 happens-before가 된다.

따라서 Thread B가 `ready == true`를 읽은 경로에서는 `data`를 안전하게 읽을 수 있고 `42`를 관찰한다.

중요한 조건은 acquire load가 release store가 저장한 값을 실제로 읽어야 한다는 점이다.

`ready == false`를 읽었다면 이 release와 acquire 사이에는 위와 같은 동기화가 만들어지지 않는다.

# 3. Relaxed ordering

## 3.1 단일 객체의 atomicity

`memory_order_relaxed`는 가장 약한 memory order다.

```c
atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
```

relaxed 연산도 다음 보장은 유지한다.

```text
해당 atomic 객체에 대한 연산은 atomic하다.
해당 atomic 객체의 modification order를 따른다.
```

하나의 atomic 객체에 수행된 모든 변경에는 그 객체만의 일관된 modification order가 존재한다.

예를 들어 여러 thread가 relaxed `fetch_add`를 수행해도 각 증가는 하나씩 순서가 정해지고 최종 counter 값도 올바르게 누적된다.

## 3.2 언어 순서와 관찰 순서

relaxed를 이해할 때 가장 헷갈리기 쉬운 지점은 sequenced-before와 실제 관찰 순서의 차이다.

먼저 data race 없이 가능한 결과를 분석하기 위해 `data`와 `ready`를 모두 atomic으로 선언해보자.

```c
atomic_int data = 0;
atomic_bool ready = false;

// Thread A
atomic_store_explicit(&data, 42, memory_order_relaxed);   // A1
atomic_store_explicit(&ready, true, memory_order_relaxed); // A2

// Thread B
bool r1 = atomic_load_explicit(&ready, memory_order_relaxed);
int r2 = atomic_load_explicit(&data, memory_order_relaxed);
```

Thread A 안에서는 `A1`이 `A2`보다 sequenced-before다.

Thread B 안에서도 `ready`의 load가 `data`의 load보다 sequenced-before다.

```text
Thread A                              Thread B

data.store(42)                        ready.load()
    |                                      |
    | sequenced-before                     | sequenced-before
    v                                      v
ready.store(true)                     data.load()
```

Thread B의 `ready` load가 Thread A의 `ready.store(true)` 값을 읽었다고 가정해보자.

두 연산 사이에는 reads-from 관계가 있지만, 둘 다 relaxed이므로 synchronizes-with는 만들어지지 않는다.

```text
A1: data.store(42)
    |
    | sequenced-before
    v
A2: ready.store(true)  ------ reads-from ------>  B1: ready.load()
                                                    |
                         synchronizes-with 없음    | sequenced-before
                                                    v
                                                B2: data.load()
```

따라서 `A1 -> A2`의 sequenced-before 관계를 `B1 -> B2`까지 이어주는 thread 간 연결이 없다.

따라서 다음 결과가 허용될 수 있다.

```text
r1 == true
r2 == 0
```

소스 코드에는 `data.store(42)` 다음에 `ready.store(true)`가 있다.

그런데도 compiler가 허용되는 범위에서 주변 연산을 재배치하거나, 약한 memory model의 CPU에서 서로 다른 주소의 store가 다른 core에 서로 다른 순서로 관찰될 수 있다.

```text
Thread A의 소스 코드 순서          Thread B의 관찰 결과

data = 42                          ready == true
ready = true                       data == 0
```

하드웨어 관점에서는 Thread A의 store가 store buffer와 cache coherence를 거쳐 다른 core에 전달된다.

이때 Thread B가 Thread A의 store buffer를 직접 읽는 것은 아니다.

Thread B는 `ready == true`를 관찰했지만 `data == 42`가 아직 자신에게 관찰 가능한 상태가 아니어서 `data`의 이전 값을 읽을 수 있다.

relaxed에는 서로 다른 atomic 객체의 관찰 순서를 연결하는 barrier가 없기 때문이다.

```text
data의 modification order     0 -> 42
ready의 modification order    false -> true

두 modification order를 연결하는 thread 간 순서     없음
```

특정 x86-64 환경에서는 강한 hardware ordering과 compiler의 명령 선택 때문에 이 결과가 실제로 나타나지 않을 수 있다.

하지만 그것이 relaxed에 이식 가능한 thread 간 순서 보장이 있다는 뜻은 아니다.

즉 relaxed에서 sequenced-before가 사라지는 것이 아니다.

```text
sequenced-before는 언어 모델상 존재한다.
실제 명령이 반드시 그 순서로 실행되어야 하는 것은 아니다.
다른 core가 반드시 그 순서로 관찰해야 하는 것도 아니다.
```

## 3.3 Thread 동기화의 부재

relaxed 연산은 다른 memory 접근을 thread 사이에서 동기화하지 않는다.

앞의 publication 예제를 relaxed로 바꾸면 문제가 생긴다.

```c
int data = 0;
atomic_bool ready = false;

// Thread A
data = 42;
atomic_store_explicit(&ready, true, memory_order_relaxed);

// Thread B
if (atomic_load_explicit(&ready, memory_order_relaxed)) {
    printf("data = %d\n", data);
}
```

`ready`의 접근 자체는 안전하다.

하지만 relaxed store와 relaxed load는 `data`의 write와 read 사이에 happens-before 관계를 만들지 않는다.

```text
Thread A                                  Thread B

data = 42                                relaxed load: ready
    |                                           |
    | sequenced-before                          | sequenced-before
    v                                           v
relaxed store: ready = true              data 읽기

             thread 사이를 잇는 동기화가 없음
```

따라서 Thread B가 `ready == true`를 읽더라도 일반 변수 `data`에 대한 접근은 C memory model에서 안전하게 동기화되지 않는다.

Thread B가 `ready == true`인 분기로 들어가 `data`를 읽으면, Thread A의 write와 Thread B의 read는 서로 충돌하는 접근이면서 happens-before로 정렬되지 않는다.

이 예제는 앞에서 모든 값을 atomic으로 선언한 예제보다 더 위험하다.

모든 값을 atomic으로 선언한 예제에서는 `ready == true`, `data == 0`이 허용 가능한 atomic 관찰 결과다.

여기서는 `data`가 일반 변수이므로 Thread A의 write와 Thread B의 read 사이에 data race가 발생해 프로그램 전체가 undefined behavior가 된다.

특정 x86-64 환경에서 항상 `42`가 출력되더라도 올바른 C 프로그램이라는 뜻은 아니다.

## 3.4 독립적인 통계 값

relaxed는 다른 데이터를 보호할 필요가 없는 독립적인 값에 잘 맞는다.

```c
atomic_ulong request_count = 0;

void record_request(void) {
    atomic_fetch_add_explicit(
        &request_count,
        1,
        memory_order_relaxed
    );
}
```

여기서 필요한 조건이 다음과 같다고 해보자.

```text
증가 연산이 유실되지 않아야 한다.
조회 시점까지 반영된 대략적인 통계 값을 읽으면 된다.
counter를 이용해 다른 데이터의 공개 여부를 판단하지 않는다.
```

이 경우에는 atomicity만 필요하므로 relaxed가 적절할 수 있다.

```c
unsigned long current_request_count(void) {
    return atomic_load_explicit(
        &request_count,
        memory_order_relaxed
    );
}
```

반대로 counter의 특정 값을 보고 다른 객체의 초기화 완료 여부를 판단한다면 단순한 relaxed counter 문제가 아니다.

# 4. Release와 acquire

## 4.1 Release store

`memory_order_release`는 주로 atomic store에 사용한다.

```c
data = 42;
atomic_store_explicit(&ready, true, memory_order_release);
```

release는 이 연산보다 앞에 있는 memory 접근을 다른 thread에 공개하는 경계 역할을 한다.

개념적으로는 앞의 연산이 release 뒤로 넘어가 동기화 의미를 깨뜨리지 못하게 한다.

```text
공개할 데이터 쓰기
공개할 데이터 쓰기
-------------------- release
ready = true
```

release store 하나만으로 다른 thread가 데이터를 획득하는 것은 아니다.

반대편에서 같은 atomic 객체를 적절한 acquire 연산으로 읽어야 한다.

## 4.2 Acquire load

`memory_order_acquire`는 주로 atomic load에 사용한다.

```c
if (atomic_load_explicit(&ready, memory_order_acquire)) {
    use(data);
}
```

acquire는 이 연산보다 뒤에 있는 memory 접근이 동기화 의미를 깨뜨리며 앞으로 넘어가지 못하게 하는 경계 역할을 한다.

```text
ready 읽기
-------------------- acquire
공개된 데이터 읽기
공개된 데이터 읽기
```

acquire load가 대응하는 release store의 값을 읽으면 release 이전의 작업을 acquire 이후에서 관찰할 수 있다.

이를 짧게 표현하면 다음과 같다.

```text
release: 이전 작업을 내보낸다.
acquire: 공개된 작업을 받아들인다.
```

다만 release와 acquire가 모든 thread를 한 번에 멈추게 하는 전역 장벽이라는 뜻은 아니다.

동기화는 어떤 atomic 연산이 어떤 값을 읽었는지에 따라 형성된다.

## 4.3 데이터 publication

release와 acquire의 대표적인 활용은 초기화한 데이터를 다른 thread에 공개하는 것이다.

여기서 publication은 한 thread가 만든 데이터의 사용 권한을 다른 thread에 넘기는 패턴을 뜻한다.

```text
Producer
    데이터 생성과 초기화
    release store로 준비 완료를 알림

Consumer
    acquire load로 준비 완료를 확인
    초기화된 데이터 사용
```

atomic flag가 실제 데이터를 운반하는 것은 아니다.

데이터는 원래의 memory 위치에 있고, flag는 producer의 작업과 consumer의 작업을 연결하는 synchronization 지점으로 사용된다.

### Publication 코드

```c
#include <stdatomic.h>
#include <stdbool.h>

typedef struct {
    int id;
    int price;
    int quantity;
} Order;

Order shared_order;
atomic_bool order_ready = false;

void publish_order(void) {
    shared_order.id = 1001;        // P1
    shared_order.price = 72000;    // P2
    shared_order.quantity = 3;     // P3

    atomic_store_explicit(
        &order_ready,
        true,
        memory_order_release
    );                             // P4
}

bool try_read_order(Order* output) {
    if (!atomic_load_explicit(
            &order_ready,
            memory_order_acquire)) { // C1
        return false;
    }

    *output = shared_order;          // C2
    return true;
}
```

`shared_order`는 atomic 객체가 아니다.

그런데도 이 코드가 안전할 수 있는 이유는 `order_ready`의 release/acquire가 일반 변수 접근 사이에 happens-before를 만들기 때문이다.

### Producer의 release

`publish_order`는 먼저 `shared_order`의 field를 초기화한다.

```text
P1: shared_order.id 쓰기
P2: shared_order.price 쓰기
P3: shared_order.quantity 쓰기
P4: order_ready에 true를 release store
```

같은 thread 안에서 P1, P2, P3는 P4보다 sequenced-before다.

```text
P1, P2, P3
     |
     | sequenced-before
     v
P4: release store
```

P4는 "이 flag만 true로 바꾼다"는 의미에 그치지 않는다.

P4 이전에 수행한 `shared_order`의 초기화를 acquire하는 consumer와 연결할 수 있는 publication 지점이 된다.

release store만 실행했다고 모든 consumer가 즉시 데이터를 읽을 수 있게 되는 것은 아니다.

consumer가 P4가 저장한 `true`를 acquire load로 읽어야 동기화가 완성된다.

### Consumer의 acquire

`try_read_order`는 먼저 C1에서 `order_ready`를 읽는다.

```text
C1이 false를 읽음
    -> 아직 publication을 확인하지 못함
    -> shared_order를 읽지 않고 반환

C1이 P4의 true를 읽음
    -> P4 synchronizes-with C1
    -> C1 이후에 shared_order를 읽을 수 있음
```

이 예제에서는 `order_ready`를 `true`로 저장하는 thread가 producer 하나뿐이다.

따라서 C1이 `true`를 읽었다면 P4가 저장한 값을 읽은 것으로 판단할 수 있다.

그 결과 다음 관계가 형성된다.

```text
Producer                                      Consumer

P1, P2, P3: shared_order 초기화
       |
       | sequenced-before
       v
P4: release store  ---------------------->  C1: acquire load
                       synchronizes-with             |
                                                     | sequenced-before
                                                     v
                                                C2: shared_order 복사
```

관계를 하나의 경로로 이어보면 다음과 같다.

```text
shared_order 초기화
    sequenced-before
release store
    synchronizes-with
acquire load
    sequenced-before
shared_order 복사
```

따라서 P1, P2, P3는 C2보다 happens-before다.

```text
P1, P2, P3  -------- happens-before -------->  C2
```

consumer가 `shared_order`를 읽을 때 producer의 초기화와 충돌하는 정렬되지 않은 접근이 남지 않는다.

그래서 `shared_order`가 일반 non-atomic 객체여도 이 publication 경로에서는 data race가 발생하지 않는다.

### Relaxed flag의 차이

flag를 relaxed로 바꾸면 flag 자체의 load와 store는 여전히 atomic하다.

```c
// Producer
shared_order.id = 1001;
atomic_store_explicit(
    &order_ready,
    true,
    memory_order_relaxed
);

// Consumer
if (atomic_load_explicit(
        &order_ready,
        memory_order_relaxed)) {
    use(shared_order.id);
}
```

하지만 relaxed store와 relaxed load 사이에는 synchronizes-with가 만들어지지 않는다.

```text
shared_order 쓰기
    sequenced-before
relaxed store  -------- reads-from -------->  relaxed load
                         synchronizes-with 없음        |
                                                        | sequenced-before
                                                        v
                                                   shared_order 읽기
```

producer의 일반 write에서 consumer의 일반 read까지 이어지는 happens-before가 없으므로 `shared_order`에는 data race가 발생한다.

이는 단순히 consumer가 예전 값을 읽을 수 있다는 수준이 아니라 C에서 undefined behavior다.

### 하드웨어의 ordering

release/acquire는 `shared_order`를 flag 안으로 복사하거나 cache 전체를 main memory로 flush하지 않는다.

대신 compiler와 CPU가 다음 결과를 허용하지 않도록 필요한 ordering을 구현한다.

```text
Consumer가 order_ready == true를 확인함
그런데 shared_order의 초기화 이전 값을 읽음
```

CPU는 store buffer와 cache coherence를 계속 사용한다.

다만 acquire load가 release store의 값을 관찰한 경우, consumer가 release 이전의 write를 관찰할 수 있도록 명령 재배치와 memory 접근 순서를 제한한다.

어떤 target에서는 release/acquire 전용 명령을 사용하고, 어떤 target에서는 기본 load/store만으로 필요한 ordering을 만족할 수 있다.

중요한 것은 특정 cache를 비웠는지가 아니라 C memory model에서 요구한 관찰 결과가 보장되는지다.

### Publication의 안전 조건

이 예제는 다음 조건에서 안전하다.

```text
1. Producer가 shared_order를 모두 초기화한 뒤 release store를 수행한다.
2. Consumer는 acquire load로 true를 확인한 뒤에만 shared_order를 읽는다.
3. Consumer가 읽은 true는 producer의 release store에서 온 값이다.
4. Publication 이후에는 shared_order를 동기화 없이 다시 수정하지 않는다.
```

publication 이후 여러 consumer가 `shared_order`를 읽기만 하는 것은 가능하다.

하지만 producer가 publication 이후 `shared_order`를 다시 수정하면 기존 release/acquire 한 번으로 그 수정까지 보호할 수 없다.

```text
초기화
release publication
consumer의 읽기       -> 보호됨

producer의 추가 수정   -> 별도의 동기화 필요
consumer의 추가 읽기
```

또한 `order_ready`를 다시 `false`로 바꿨다가 재사용하는 것만으로 반복 publication protocol이 완성되지는 않는다.

여러 번 갱신하거나 writer가 여러 명이라면 mutex, sequence number, double buffering처럼 상태 전환 전체를 설명할 수 있는 별도 protocol이 필요하다.

release/acquire 한 쌍은 publication 이전의 작업과 acquire 이후의 작업을 연결한다.

객체의 남은 lifetime 전체를 자동으로 보호하는 장치는 아니다.

# 5. Acquire-release ordering

## 5.1 Read-modify-write의 양방향 역할

`memory_order_acq_rel`은 acquire와 release를 결합한 order다.

이 order는 값을 읽으면서 동시에 새로운 값을 쓰는 read-modify-write 연산에 사용한다.

```c
atomic_fetch_add_explicit(
    &state,
    1,
    memory_order_acq_rel
);
```

하나의 연산이 두 역할을 수행한다.

```text
acquire
    -> 이전 thread가 release한 작업을 받아들인다.

read-modify-write
    -> 현재 atomic 값을 읽고 새 값으로 변경한다.

release
    -> 현재 thread가 앞에서 수행한 작업을 다음 thread에 공개한다.
```

따라서 이전 상태에 딸린 데이터를 읽은 뒤 새로운 상태와 데이터를 다음 thread에 넘기는 상태 전환에 사용할 수 있다.

단순 통계 counter처럼 주변 데이터를 동기화하지 않는 연산에는 보통 acq_rel까지 필요하지 않다.

## 5.2 Compare-exchange의 성공과 실패

`compare_exchange`는 성공할 때와 실패할 때 하는 일이 다르다.

먼저 compare-exchange의 동작을 일반 코드처럼 표현해보자.

```c
if (state == expected) {
    state = desired;
    return true;
}

expected = state;
return false;
```

실제 compare-exchange는 위 비교와 변경을 하나의 atomic operation으로 수행한다.

```c
bool changed = atomic_compare_exchange_weak_explicit(
    &state,
    &expected,
    desired,
    memory_order_acq_rel,
    memory_order_acquire
);
```

### 성공 경로

`state`의 현재 값이 `expected`와 같으면 비교에 성공한다.

```text
state == expected
    -> state의 기존 값을 읽음
    -> state에 desired를 저장
    -> true 반환
```

성공 경로는 값을 읽고 새로운 값을 쓰므로 read-modify-write다.

```text
load state
compare
store desired

위 과정 전체가 하나의 atomic operation
```

다른 thread는 비교와 store 사이에 끼어들어 `state`를 변경할 수 없다.

성공할 때는 `expected`의 값이 변경되지 않는다.

### 실패 경로

`state`의 현재 값이 `expected`와 다르면 비교에 실패한다.

```text
state != expected
    -> state의 현재 값만 읽음
    -> state에는 아무 값도 쓰지 않음
    -> expected를 state의 현재 값으로 변경
    -> false 반환
```

예를 들어 다음 상태에서 CAS를 실행한다고 해보자.

```text
state    = 3
expected = 2
desired  = 4
```

비교에 실패한 뒤의 값은 다음과 같다.

```text
state    = 3    // 변경되지 않음
expected = 3    // 실제 state 값으로 갱신
반환 값  = false
```

실패 경로는 atomic 객체에 값을 쓰지 않고 load만 수행한다.

이 차이 때문에 compare-exchange는 success order와 failure order를 따로 받는다.

```text
success order
    -> 성공한 read-modify-write 전체에 적용

failure order
    -> 실패했을 때 수행한 load에만 적용
```

### Success order의 역할

성공한 compare-exchange에는 read와 write가 모두 있으므로 목적에 따라 여러 order를 사용할 수 있다.

```text
memory_order_relaxed
    -> state 전환의 atomicity만 필요

memory_order_acquire
    -> 기존 state를 만든 thread의 작업을 획득

memory_order_release
    -> 현재 thread의 앞선 작업을 새 state와 함께 공개

memory_order_acq_rel
    -> 이전 작업을 획득하고 현재 작업을 다시 공개

memory_order_seq_cst
    -> acquire/release 의미와 seq_cst 전체 순서가 필요
```

`memory_order_acq_rel` 성공 경로는 다음 두 방향을 동시에 연결할 수 있다.

```text
이전 thread의 일반 write
    sequenced-before
이전 thread의 release operation
    synchronizes-with
현재 thread의 성공한 CAS acquire
    sequenced-before
현재 thread의 이후 read
```

그리고 CAS의 release 쪽은 현재 thread의 앞선 작업을 다음 thread로 넘길 수 있다.

```text
현재 thread의 일반 write
    sequenced-before
현재 thread의 성공한 CAS release
    synchronizes-with
다음 thread의 acquire operation
    sequenced-before
다음 thread의 이후 read
```

이를 하나의 상태 handoff로 줄이면 다음과 같다.

```text
이전 thread의 release
        |
        | synchronizes-with
        v
성공한 CAS의 acquire + release
        |
        | synchronizes-with
        v
다음 thread의 acquire
```

CAS는 acquire 쪽으로 이전 상태에 딸린 작업을 받아들이고, release 쪽으로 현재 thread가 준비한 작업을 새로운 상태와 함께 공개한다.

단순히 숫자 상태 하나만 바꾸고 주변 데이터를 동기화하지 않는다면 acq_rel까지 필요하지 않을 수 있다.

### Failure order의 역할

실패 경로에는 atomic 객체에 대한 write가 없다.

따라서 다른 thread에 현재 thread의 작업을 공개하는 release 의미를 적용할 대상도 없다.

```text
실패 경로
    state load
    expected 갱신
    return false

state store는 없음
```

이 때문에 failure order에는 `memory_order_release`와 `memory_order_acq_rel`을 사용할 수 없다.

실패 후 무엇을 하느냐에 따라 relaxed와 acquire를 선택할 수 있다.

```text
실패 후 expected 값만 비교하고 다시 시도
    -> memory_order_relaxed를 검토

실패하면서 읽은 state에 연결된 일반 데이터를 사용
    -> memory_order_acquire를 검토
```

예를 들어 실패한 현재 상태만 확인하고 함수를 종료한다면 failure acquire가 필요하지 않을 수 있다.

```c
bool try_change_state(int desired) {
    int expected = STATE_READY;

    return atomic_compare_exchange_strong_explicit(
        &state,
        &expected,
        desired,
        memory_order_acq_rel,
        memory_order_relaxed
    );
}
```

반면 실패하면서 읽은 `expected`가 다른 thread가 release로 공개한 상태이고, 실패 경로에서 그 상태에 딸린 데이터를 읽어야 한다면 acquire가 필요할 수 있다.

```c
if (!atomic_compare_exchange_strong_explicit(
        &state,
        &expected,
        desired,
        memory_order_acq_rel,
        memory_order_acquire)) {
    inspect_data_for(expected);
}
```

이 코드는 `inspect_data_for`가 실제로 어떤 데이터를 읽고 그 데이터를 누가 어떻게 공개했는지까지 함께 확인해야 한다.

failure order를 acquire로 지정했다는 사실만으로 임의의 데이터가 자동으로 보호되지는 않는다.

### Weak CAS의 재시도

`atomic_compare_exchange_weak`는 `state == expected`여도 spurious failure를 허용한다.

따라서 일반적으로 loop 안에서 사용한다.

```c
bool change_ready_to_running(void) {
    int expected = STATE_READY;

    while (!atomic_compare_exchange_weak_explicit(
        &state,
        &expected,
        STATE_RUNNING,
        memory_order_acq_rel,
        memory_order_relaxed
    )) {
        if (expected != STATE_READY) {
            return false;
        }
    }

    return true;
}
```

실패할 때마다 `expected`에는 CAS가 읽은 현재 `state` 값이 들어간다.

```text
첫 시도
    expected = STATE_READY

다른 값 때문에 실패
    expected = 현재 state

spurious failure
    state가 여전히 STATE_READY라면 expected도 STATE_READY
    loop에서 다시 시도
```

위 loop는 실패 후 상태 값만 검사하므로 failure order로 relaxed를 사용한다.

성공 이후 이전 thread가 공개한 데이터가 필요하고, 성공 이전의 작업도 다음 thread에 공개해야 한다는 전제에서 success order로 acq_rel을 사용했다.

실제 코드에서는 이 두 요구가 모두 있는지 확인해야 한다.

### 유효한 order 조합

failure order는 success order보다 더 강할 수 없다.

`memory_order_consume`을 제외하고 자주 사용하는 유효 조합을 정리하면 다음과 같다.

| Success order | 사용 가능한 failure order |
|---|---|
| `memory_order_relaxed` | `memory_order_relaxed` |
| `memory_order_acquire` | `memory_order_relaxed`, `memory_order_acquire` |
| `memory_order_release` | `memory_order_relaxed` |
| `memory_order_acq_rel` | `memory_order_relaxed`, `memory_order_acquire` |
| `memory_order_seq_cst` | `memory_order_relaxed`, `memory_order_acquire`, `memory_order_seq_cst` |

실패 경로는 load일 뿐이므로 release 의미를 가질 수 없다는 원칙으로 보면 표를 이해하기 쉽다.

```text
성공
    read + write
    -> acquire와 release 모두 가능

실패
    read only
    -> acquire는 가능
    -> release는 불가능
```

compare-exchange에서 memory order를 선택할 때는 성공과 실패를 별도 실행 경로로 그려보는 것이 좋다.

```text
성공 후 어떤 데이터를 읽는가?
성공 전에 준비한 무엇을 공개하는가?
실패 후 expected만 검사하는가?
실패하며 관찰한 상태에 딸린 데이터를 읽는가?
```

이 질문에 대한 답이 success order와 failure order를 결정한다.

# 6. Sequential consistency

## 6.1 기본 memory order

`memory_order_seq_cst`는 sequentially consistent ordering을 의미한다.

`_explicit`이 없는 기본 atomic API는 seq_cst를 사용한다.

```c
atomic_store(&ready, true);
atomic_load(&ready);
atomic_fetch_add(&counter, 1);
```

위 코드는 각각 다음과 같은 의미다.

```c
atomic_store_explicit(&ready, true, memory_order_seq_cst);
atomic_load_explicit(&ready, memory_order_seq_cst);
atomic_fetch_add_explicit(&counter, 1, memory_order_seq_cst);
```

seq_cst load에는 acquire 의미가 있고, seq_cst store에는 release 의미가 있다.

read-modify-write에는 양쪽 의미가 포함된다.

여기에 더해 모든 seq_cst 연산이 thread들이 동의할 수 있는 하나의 total order에 놓인다는 강한 제약을 제공한다.

## 6.2 단일 전체 순서

두 atomic 변수를 사용하는 예제를 보자.

```c
atomic_int x = 0;
atomic_int y = 0;

// Thread A
atomic_store_explicit(&x, 1, memory_order_seq_cst);
int r1 = atomic_load_explicit(&y, memory_order_seq_cst);

// Thread B
atomic_store_explicit(&y, 1, memory_order_seq_cst);
int r2 = atomic_load_explicit(&x, memory_order_seq_cst);
```

seq_cst에서는 두 thread가 모두 `0`을 읽는 결과를 허용하지 않는다.

```text
r1 == 0 && r2 == 0  -> 허용되지 않음
```

모든 seq_cst 연산을 하나의 순서로 놓으면서 각 thread의 코드 순서도 지키려 하면, 두 load가 모두 상대편 store보다 먼저 와야 하는 모순이 생기기 때문이다.

```text
x = 1  <  y 읽기     Thread A의 순서
y = 1  <  x 읽기     Thread B의 순서

y 읽기가 0을 보려면  y 읽기 < y = 1
x 읽기가 0을 보려면  x 읽기 < x = 1

모두 연결하면 순환이 생겨 하나의 total order를 만들 수 없다.
```

같은 코드를 relaxed로 작성하면 두 load가 모두 `0`을 읽는 결과가 허용될 수 있다.

seq_cst의 강점은 여러 atomic 객체가 섞인 코드에서도 가능한 실행 결과를 더 직관적으로 제한한다는 점이다.

## 6.3 강한 보장과 비용

seq_cst가 항상 별도의 무거운 명령으로 바뀌는 것은 아니다.

실제 비용은 CPU architecture, 연산 종류, compiler가 선택한 명령에 따라 다르다.

어떤 target에서는 acquire/release와 같은 명령이 나올 수 있고, 다른 target이나 연산에서는 추가 fence 또는 더 강한 명령이 필요할 수 있다.

따라서 다음처럼 판단하면 안 된다.

```text
relaxed는 항상 빠르다.
seq_cst는 항상 느리다.
```

다만 seq_cst는 compiler와 CPU에 더 강한 순서 제약을 요구하므로 최적화와 구현의 자유를 더 제한할 수 있다.

정확성이 우선이라면 기본 seq_cst로 시작하고, 병목을 측정한 뒤 더 약한 order로 바꾸는 접근이 안전하다.

# 7. Consume과 fence

## 7.1 Consume ordering

C11에는 `memory_order_consume`도 정의되어 있다.

consume은 acquire보다 약한 형태로, atomic load에서 얻은 값에 대한 dependency를 따라 ordering을 제공하려는 목적을 가진다.

```text
acquire
    -> load 이후의 일반적인 memory 접근에 ordering 제공

consume
    -> 읽은 값에 의존하는 memory 접근을 중심으로 ordering 제공
```

하지만 dependency의 의미를 language와 compiler 최적화에 걸쳐 정확하게 다루기 어렵다.

실제 compiler는 consume을 더 강한 acquire처럼 구현하는 경우가 일반적이다.

따라서 실무 C 코드에서는 특별한 이유와 target 검증이 없다면 `memory_order_acquire`를 사용하는 편이 이해하고 유지하기 쉽다.

## 7.2 Atomic fence

`atomic_thread_fence`는 특정 atomic 객체를 직접 읽거나 쓰지 않고 ordering 제약만 만든다.

```c
atomic_thread_fence(memory_order_acquire);
atomic_thread_fence(memory_order_release);
atomic_thread_fence(memory_order_seq_cst);
```

fence는 단독으로 임의의 thread를 동기화하지 않는다.

어떤 atomic 객체의 load와 store가 fence를 연결하는지까지 함께 증명해야 한다.

잘못 배치한 fence는 코드만 복잡하게 만들고 필요한 happens-before 관계를 만들지 못할 수 있다.

처음에는 ordering을 atomic load, store, read-modify-write에 직접 지정하는 방식이 더 명확하다.

# 8. 연산별 memory order

## 8.1 Load와 store

모든 atomic 연산에 모든 memory order를 사용할 수 있는 것은 아니다.

atomic load는 값을 쓰지 않으므로 release 의미를 가질 수 없다.

```text
atomic load에서 사용 가능
    memory_order_relaxed
    memory_order_consume
    memory_order_acquire
    memory_order_seq_cst

atomic load에서 사용 불가
    memory_order_release
    memory_order_acq_rel
```

atomic store는 값을 읽어 획득하지 않으므로 acquire 의미를 가질 수 없다.

```text
atomic store에서 사용 가능
    memory_order_relaxed
    memory_order_release
    memory_order_seq_cst

atomic store에서 사용 불가
    memory_order_consume
    memory_order_acquire
    memory_order_acq_rel
```

## 8.2 Read-modify-write

`exchange`, `fetch_add`, 성공한 `compare_exchange` 같은 read-modify-write는 값을 읽고 쓰기 때문에 다양한 order를 사용할 수 있다.

```text
read-modify-write에서 사용 가능
    memory_order_relaxed
    memory_order_consume
    memory_order_acquire
    memory_order_release
    memory_order_acq_rel
    memory_order_seq_cst
```

필요한 의미에 따라 선택한다.

```text
atomicity만 필요                     -> relaxed
이전 thread의 작업을 받아야 함        -> acquire
현재 thread의 작업을 공개해야 함       -> release
받기와 공개를 모두 해야 함             -> acq_rel
seq_cst 전체 순서까지 필요             -> seq_cst
```

## 8.3 Compare-exchange의 failure order

compare-exchange 실패 경로는 store를 수행하지 않는다.

따라서 failure order에는 release 의미를 지정할 수 없다.

```text
failure order에서 사용 가능
    memory_order_relaxed
    memory_order_consume
    memory_order_acquire
    memory_order_seq_cst

failure order에서 사용 불가
    memory_order_release
    memory_order_acq_rel
```

failure order는 success order보다 강할 수도 없다.

명시적 compare-exchange를 사용할 때 이 규칙을 놓치기 쉬우므로 compiler warning을 확인해야 한다.

# 9. Memory order 선택 기준

## 9.1 독립적인 atomic 값

다른 일반 데이터를 보호하거나 공개하지 않고 atomic 값 자체의 정확성만 필요하다면 relaxed를 검토할 수 있다.

```text
통계 counter
이벤트 발생 횟수
독립적인 reference count의 증감
```

reference count도 객체 파괴와 lifetime 동기화 단계에서는 더 강한 order가 필요할 수 있으므로 전체 알고리즘을 따로 검토해야 한다.

## 9.2 데이터 공개와 전달

한 thread가 작성한 일반 데이터를 flag나 pointer를 통해 다른 thread에 전달한다면 release/acquire가 대표적인 선택이다.

```text
Producer
    data 작성
    release store로 ready 공개

Consumer
    acquire load로 ready 확인
    data 사용
```

이때 acquire가 release가 저장한 값 또는 그에 연결된 release sequence의 값을 읽는지 확인해야 한다.

## 9.3 상태 전환

현재 상태와 이전 thread의 결과를 획득하면서 새로운 상태를 다음 thread에 공개하는 read-modify-write라면 acq_rel을 검토할 수 있다.

```text
CAS 기반 상태 전환
소유권 handoff
lock-free 자료구조의 node 연결
```

이 영역에서는 객체 lifetime, ABA 문제, memory reclamation까지 함께 고려해야 하므로 memory order만 맞춘다고 알고리즘 전체가 안전해지는 것은 아니다.

## 9.4 기본값과 최적화

memory order를 확신하기 어렵다면 seq_cst로 시작하는 것이 좋다.

```text
1. 먼저 seq_cst로 올바른 알고리즘을 작성한다.
2. test와 sanitizer로 기본적인 오류를 확인한다.
3. profiler와 benchmark로 실제 병목을 측정한다.
4. happens-before 관계를 설명할 수 있는 연산만 약하게 바꾼다.
5. target architecture에서 assembly와 성능을 다시 확인한다.
```

약한 memory order는 단순한 성능 옵션이 아니다.

프로그램이 허용하는 실행 결과 자체를 바꾸는 correctness 조건이다.

# 정리

이번 글에서는 C atomic operation의 memory order를 살펴봤다.

핵심은 다음과 같다.

```text
atomicity는 atomic 객체 자체의 연산을 나뉘지 않게 한다.
memory ordering은 주변 memory 접근의 관찰 순서를 정한다.
sequenced-before는 C abstract machine의 thread 내부 관계다.
sequenced-before가 실제 CPU 명령이나 다른 core의 관찰 순서를 뜻하지는 않는다.
relaxed도 atomicity는 유지하지만 thread 사이의 동기화를 만들지 않는다.
relaxed는 서로 다른 atomic 객체의 관찰 순서를 연결하지 않는다.
release store와 그 값을 읽은 acquire load는 synchronizes-with 관계를 만든다.
이 관계를 통해 앞선 write와 뒤의 read 사이에 happens-before가 형성된다.
acq_rel은 read-modify-write에서 acquire와 release 역할을 함께 수행한다.
seq_cst는 acquire/release 의미와 seq_cst 연산 사이의 단일 total order를 제공한다.
consume은 실무에서 acquire로 대체되는 경우가 많다.
fence는 atomic 연산과의 연결까지 함께 이해해야 한다.
약한 memory order는 측정 전에 적용하는 단순한 성능 최적화가 아니다.
```

memory order를 선택할 때는 "이 연산이 얼마나 강해야 하는가"보다 먼저 "어떤 write가 어떤 read보다 happens-before여야 하는가"를 그려보는 것이 좋다.

다음 글에서는 atomic과 mutex를 실제 코드 관점에서 비교하고, spinlock 구현을 통해 atomic으로 lock을 만들 때 생기는 장점과 한계를 정리한다.

# 참고 자료

- [C11 draft N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [C Memory Model Rationale N1479](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1479.htm)
- [GCC - Built-in Functions for Memory Model Aware Atomic Operations](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)
- [LLVM Atomic Instructions and Concurrency Guide](https://llvm.org/docs/Atomics.html)
