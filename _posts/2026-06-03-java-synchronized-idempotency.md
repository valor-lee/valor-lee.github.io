---
title: '[Java] synchronized로 idempotency 중복 생성을 막는 이유'
date: 2026-06-03 15:40:00 +09:00
categories: [Language, Java]
tags:
  [
    Java,
    concurrency,
    synchronized,
    idempotency,
    Spring,
    backend
  ]
---

# 개요

`multi-asset-oms-simulation-platform` 프로젝트에서 주문 의도 생성 API에 `idempotencyKey` 중복 방어를 추가했다.

같은 `idempotencyKey`와 같은 요청 내용이 다시 들어오면 새 `OrderIntent`를 만들지 않고 기존 결과를 반환한다.

```java
public synchronized OrderIntent create(CreateOrderIntentCommand command) {
    Optional<OrderIntent> existingIntent = findExistingIntent(command.idempotencyKey());
    if (existingIntent.isPresent()) {
        return resolveExistingIntent(existingIntent.get(), command);
    }
    return orderIntentRepository.save(orderIntentFactory.create(command));
}
```

여기서 궁금했던 지점은 `synchronized`다.

> 왜 `synchronized`가 필요할까?
> 그냥 repository에서 먼저 조회하고 없으면 저장하면 안 될까?

이번 글에서는 `synchronized`의 역할과, 왜 MVP 단계에서는 유용하지만 실무 DB 환경에서는 다른 방식으로 바꿔야 하는지 정리한다.

# idempotencyKey가 필요한 이유

주문 생성 API는 중복 요청에 취약하다.

예를 들어 사용자가 버튼을 두 번 누를 수 있고, 네트워크 timeout 때문에 클라이언트가 같은 요청을 재시도할 수도 있다.

```http
POST /api/order-intents/manual
```

```json
{
  "portfolioId": "portfolio-1",
  "instrumentId": "005930",
  "side": "BUY",
  "orderType": "LIMIT",
  "requestedQty": 10,
  "limitPrice": 55000,
  "timeInForce": "DAY",
  "idempotencyKey": "manual-key-1",
  "createdBy": "operator"
}
```

같은 요청이 두 번 들어왔을 때 주문 의도가 두 개 생기면 위험하다.

```text
첫 번째 요청 -> OrderIntent A 생성
두 번째 요청 -> OrderIntent B 생성
```

사용자는 한 번 주문했다고 생각하지만, 시스템에는 두 개의 주문 의도가 생긴다.

그래서 `idempotencyKey`를 사용한다.

```text
같은 idempotencyKey + 같은 요청 내용
    -> 기존 OrderIntent 반환

같은 idempotencyKey + 다른 요청 내용
    -> 409 Conflict
```

# 조회 후 저장 사이의 틈

처음에는 아래 흐름만 생각하기 쉽다.

```text
1. idempotencyKey로 기존 OrderIntent 조회
2. 없으면 새 OrderIntent 생성
3. repository에 저장
```

코드로 보면 이런 모양이다.

```java
public OrderIntent create(CreateOrderIntentCommand command) {
    Optional<OrderIntent> existingIntent =
            orderIntentRepository.findByIdempotencyKey(command.idempotencyKey());

    if (existingIntent.isPresent()) {
        return existingIntent.get();
    }

    OrderIntent intent = orderIntentFactory.create(command);
    return orderIntentRepository.save(intent);
}
```

혼자 실행하면 문제가 없어 보인다.

하지만 요청이 동시에 두 개 들어오면 이야기가 달라진다.

```text
Thread A: manual-key-1 조회 -> 없음
Thread B: manual-key-1 조회 -> 없음
Thread A: 새 OrderIntent 생성 후 저장
Thread B: 새 OrderIntent 생성 후 저장
```

둘 다 조회 시점에는 "아직 없음"이라고 판단했기 때문에, 같은 key로 두 개의 intent를 만들 수 있다.

이 문제는 조회와 저장 사이에 생기는 race condition이다.

# synchronized의 역할

`synchronized`는 Java에서 한 객체의 특정 메서드나 블록에 동시에 여러 thread가 들어오지 못하게 막는다.

```java
public synchronized OrderIntent create(CreateOrderIntentCommand command) {
    ...
}
```

위 코드는 같은 `OrderIntentCreator` 객체 기준으로 `create()`를 한 번에 한 thread만 실행하게 만든다.

그러면 흐름이 이렇게 바뀐다.

```text
Thread A: create() 진입
Thread B: create() 대기

Thread A: manual-key-1 조회 -> 없음
Thread A: 새 OrderIntent 생성 후 저장
Thread A: create() 종료

Thread B: create() 진입
Thread B: manual-key-1 조회 -> 있음
Thread B: 기존 OrderIntent 반환
```

즉 MVP의 in-memory repository 환경에서는 `synchronized`가 간단한 동시성 방어 역할을 한다.

# 왜 MVP에서는 괜찮은가

현재 프로젝트는 아직 DB 영속화 전 단계이고, `InMemoryOrderIntentRepository`를 사용한다.

이 단계에서는 아래 두 조건이 맞는다.

```text
1. 하나의 JVM 안에서 테스트한다.
2. repository도 메모리 객체다.
```

이런 환경에서는 `synchronized`가 꽤 실용적이다.

- 구현이 단순하다.
- 테스트하기 쉽다.
- 같은 JVM 안의 동시 요청을 막을 수 있다.
- DB transaction이나 unique constraint가 없어도 최소한의 중복 방어를 할 수 있다.

그래서 MVP에서는 "동작 의미를 먼저 고정하는 장치"로 쓸 수 있다.

# synchronized의 한계

하지만 `synchronized`가 실무 최종 해법은 아니다.

가장 큰 한계는 JVM 내부 lock이라는 점이다.

서버가 두 대라면 어떻게 될까?

```text
Server A의 OrderIntentCreator lock
Server B의 OrderIntentCreator lock
```

두 lock은 서로 모른다.

즉 Server A와 Server B가 동시에 같은 `idempotencyKey`를 처리하면, 각 서버는 자기 JVM 안에서만 lock을 잡는다.

```text
Server A: manual-key-1 조회 -> 없음
Server B: manual-key-1 조회 -> 없음
Server A: 저장
Server B: 저장
```

여전히 중복 생성 가능성이 있다.

또 하나의 문제는 성능이다.

```java
public synchronized OrderIntent create(...)
```

이렇게 메서드 전체에 lock을 걸면, 서로 다른 `idempotencyKey` 요청도 한 줄로 서게 된다.

```text
manual-key-1 요청
manual-key-2 요청
manual-key-3 요청
```

사실 이 세 요청은 서로 독립적이다. 하지만 메서드 전체 lock을 잡으면 동시에 처리하지 못한다.

MVP에서는 괜찮지만 트래픽이 커지면 병목이 될 수 있다.

# 실무에서는 어떻게 바꾸는가

DB를 쓰는 단계로 가면 보통 `synchronized` 대신 DB 제약과 transaction으로 막는다.

기본 아이디어는 이렇다.

```text
1. idempotency_key에 unique constraint를 건다.
2. 요청 payload를 canonical form으로 만든다.
3. canonical payload로 request_hash를 만든다.
4. idempotency_key와 request_hash를 함께 저장한다.
5. 같은 key가 다시 들어오면 hash를 비교한다.
```

예를 들면 테이블에 이런 컬럼을 둘 수 있다.

```text
order_intent
    intent_id
    idempotency_key
    request_hash
    portfolio_id
    instrument_id
    side
    order_type
    requested_qty
    limit_price
    ...
```

그리고 DB에는 unique constraint를 둔다.

```sql
ALTER TABLE order_intent
ADD CONSTRAINT uk_order_intent_idempotency_key UNIQUE (idempotency_key);
```

처리 흐름은 이렇게 된다.

```text
새 요청
    -> request_hash 계산
    -> insert 시도
    -> 성공하면 새 OrderIntent 반환
    -> unique constraint 위반이면 기존 row 조회
        -> hash 같음: 기존 OrderIntent 반환
        -> hash 다름: 409 Conflict
```

이 방식은 서버가 여러 대여도 DB가 최종 중복 방어선을 제공한다.

# requestHash를 쓰는 이유

현재 MVP 코드에서는 기존 intent와 새 command의 필드를 직접 비교한다.

```java
private boolean matches(OrderIntent existingIntent, CreateOrderIntentCommand command) {
    return Objects.equals(existingIntent.portfolioId(), normalize(command.portfolioId()))
            && Objects.equals(existingIntent.instrumentId(), normalize(command.instrumentId()))
            && existingIntent.sourceType() == command.sourceType()
            && Objects.equals(existingIntent.sourceRefId(), normalize(command.sourceRefId()))
            && existingIntent.side() == command.side()
            && existingIntent.orderType() == command.orderType()
            && sameNumber(existingIntent.requestedQty(), command.requestedQty())
            && sameNumber(existingIntent.limitPrice(), command.limitPrice())
            && existingIntent.timeInForce() == command.timeInForce()
            && Objects.equals(existingIntent.reason(), normalize(command.reason()))
            && Objects.equals(existingIntent.createdBy(), normalize(command.createdBy()));
}
```

이 정도 필드 수에서는 큰 문제가 아니다.

하지만 필드가 늘어나면 비교 로직도 같이 커진다.

```text
필드 추가
    -> matches() 수정 필요
    -> 테스트 수정 필요
    -> 누락 가능성 증가
```

그래서 영속화 단계에서는 normalized payload를 만들고 hash로 비교하는 방식이 더 단순해진다.

```text
normalized payload -> SHA-256 -> request_hash
```

이러면 비교는 단순해진다.

```text
기존 request_hash == 새 request_hash
```

물론 hash를 만들 때 어떤 필드를 포함할지도 정책이다.

예를 들어 `reason`을 포함할지 말지는 선택해야 한다.

```text
보수적 정책:
    reason도 다르면 다른 요청으로 보고 409

실무 편의 정책:
    수량, 가격, 종목, source, side가 같으면 같은 주문으로 보고 reason 차이는 무시
```

현재 프로젝트는 보수적 정책에 가깝다.

# 정리

`synchronized`는 현재 MVP 단계에서 in-memory repository의 race condition을 줄이기 위한 간단한 방어다.

하지만 최종 해법은 아니다.

정리하면 다음과 같다.

| 단계 | 적합한 방식 |
| --- | --- |
| MVP / in-memory | `synchronized`로 조회-생성-저장 구간 보호 |
| 단일 서버 + DB | transaction + unique constraint |
| 다중 서버 + DB | unique constraint + request_hash + 충돌 처리 |

현재 코드의 `synchronized`는 "지금 단계에서 의미를 고정하기 위한 임시 안전장치"로 이해하면 된다.

DB 영속화 단계에서는 `idempotency_key` unique constraint와 `request_hash` 기반 비교로 바꾸는 것이 더 실무적인 방향이다.
