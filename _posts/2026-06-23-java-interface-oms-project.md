---
title: '[Java] OMS 프로젝트에서 interface를 활용한 방법'
date: 2026-06-23 23:40:00 +09:00
categories: [Language, Java]
tags:
  [
    Java,
    interface,
    Spring,
    backend,
    project,
    OMS,
    architecture
  ]
---

# 개요

`multi-asset-oms-simulation-platform` 프로젝트를 진행하면서 Java의 `interface`를 여러 곳에 사용했다.

처음에는 interface를 단순히 "구현 클래스가 메서드를 강제로 구현하게 하는 문법" 정도로 이해했다.

하지만 프로젝트를 진행하다 보니 interface는 단순 문법보다 더 실용적인 역할을 했다.

이번 프로젝트에서 interface를 사용한 목적은 크게 두 가지였다.

1. application service가 구체 저장소 구현을 모르도록 하기
2. 여러 risk rule을 같은 방식으로 실행할 수 있게 하기

즉, interface는 "무엇을 할 수 있어야 하는가"라는 계약을 먼저 세우고, 실제 구현은 뒤로 미루는 도구였다.

# 글의 구성

현재 프로젝트에서 interface는 주로 repository port와 risk rule 계약에 사용하고 있다.

대표적인 예시는 다음과 같다.

```text
Repository Port
    application service가 저장소 구현을 직접 알지 않도록 하는 계약

Risk Rule
    여러 리스크 규칙을 같은 방식으로 실행하기 위한 계약
```

repository 계열 interface는 application service가 in-memory 저장소인지, JPA 저장소인지, 외부 저장소인지 알지 못하게 하기 위해 사용했다.

`PreTradeRiskRule`은 여러 리스크 규칙을 하나의 리스트로 실행하기 위해 사용했다.

# 활용 1. 저장소 구현을 숨기는 Repository Port

첫 번째 활용 방식은 repository port다.

핵심 의도는 application service가 구체 저장소 구현을 모르도록 만드는 것이다.

## 저장소 기능을 계약으로 정의하기

예를 들어 execution 모듈에는 `OrderRepository` interface가 있다.

```java
public interface OrderRepository {

    Order save(Order order);

    Optional<Order> findByOrderId(UUID orderId);

    Optional<Order> findByIntentId(UUID intentId);

    List<Order> findAll();
}
```

이 interface는 "Order 저장소라면 최소한 이런 기능을 제공해야 한다"는 계약이다.

중요한 점은 여기에는 저장 방식이 없다.

`Map`에 저장하는지, PostgreSQL에 저장하는지, Redis를 쓰는지 같은 정보가 없다.

현재 MVP 단계에서는 `InMemoryOrderRepository`가 이 interface를 구현한다.(2026-06-23)

```text
OrderRepository
    -> InMemoryOrderRepository
```

하지만 service는 `InMemoryOrderRepository`를 직접 의존하지 않는다.

예를 들어 `OrderFillService`는 아래처럼 interface에 의존한다.

```java
private final OrderRepository orderRepository;
private final OrderFillExecutionRepository fillExecutionRepository;

...
```

이 구조 덕분에 `OrderFillService`는 저장소가 어떻게 구현되어 있는지 몰라도 된다.

service가 관심을 가져야 하는 것은 "order를 조회하고 저장할 수 있는가"이지, "어떤 자료구조나 DB로 저장하는가"가 아니다.

## application과 infrastructure 분리하기

프로젝트에서는 패키지도 interface의 의도를 드러내도록 나누었다.

```text
execution
    application
        port
            OrderRepository
            OrderFillExecutionRepository
    infrastructure
        InMemoryOrderRepository
        InMemoryOrderFillExecutionRepository
```

`application.port`는 application service가 바깥세상에 요구하는 기능이다.

`infrastructure`는 그 요구사항을 실제로 구현하는 영역이다.

이 구조를 쓰면 의존 방향이 깔끔해진다.

```text
Application Service
    -> Repository Interface
        <- InMemory Repository
```

application service가 infrastructure 구현체를 직접 알지 않아도 되기 때문에, 나중에 JPA 기반 repository로 바꾸더라도 service의 핵심 로직은 그대로 둘 수 있다.

각 service는 자신에게 필요한 repository interface만 바라본다.


## 테스트하기 쉬운 구조 만들기

interface를 두면 테스트도 편해진다.

현재는 대부분 in-memory repository를 사용해 service 테스트를 작성하고 있다.

예를 들어 `AverageCostService` 테스트에서는 실제 DB를 띄우지 않아도 된다.

```text
AverageCostService
    -> InMemoryTradeRepository
    -> InMemoryAverageCostRepository
```

이 구조에서는 테스트가 빠르고 단순하다.

나중에 DB를 붙이면 다음처럼 테스트 층을 나눌 수 있다.

```text
Service 단위 테스트
    -> InMemory Repository

Repository 통합 테스트
    -> Jpa Repository / PostgreSQL
```

즉, interface를 둔 덕분에 "도메인 규칙 테스트"와 "저장소 구현 테스트"를 분리할 수 있다.

서비스 테스트에서는 평균단가 계산, realized PnL 계산, idempotency 같은 비즈니스 규칙에 집중할 수 있다.

# 활용 2. 여러 Risk Rule을 같은 방식으로 실행하기

두 번째 활용 방식은 pre-trade risk rule이다.

핵심 의도는 서로 다른 리스크 규칙을 하나의 실행 계약으로 묶는 것이다.

## 규칙이 늘어나면서 생긴 문제

프로젝트 초반에는 `PreTradeRiskCheckService` 안에 모든 규칙 판단 로직이 들어 있었다.

처음에는 괜찮았지만, 규칙이 늘어나면서 service가 점점 커졌다.

현재 리스크 규칙은 대략 아래와 같다.

```text
PositiveQuantityRule
LimitPriceRequiredRule
PositiveLimitPriceRule
MaxOrderQuantityRule
MaxOrderNotionalRule
MaxPositionQuantityRule
DuplicateOpenOrderRule
PriceBandRule
KillSwitchRule
```

이 규칙들은 세부 로직은 다르지만 공통점이 있다.

모두 같은 입력을 받고, 같은 결과 타입을 반환한다.

## 실행 계약으로 PreTradeRiskRule 만들기

그래서 `PreTradeRiskRule` interface를 만들었다.

```java
public interface PreTradeRiskRule {

    PreTradeRiskRuleCheckResult evaluate(
            PreTradeRiskCheckCommand command,
            PreTradeRiskCheckContext checkContext
    );
}
```

이 interface의 의미는 단순하다.

> 리스크 규칙이라면 command와 context를 받아 rule check result를 반환할 수 있어야 한다.

이제 `PreTradeRiskCheckService`는 구체 rule class를 일일이 다르게 처리하지 않는다.

```java
List<PreTradeRiskRuleCheckResult> ruleResults = rules.stream()
        .map(rule -> rule.evaluate(command, resolvedContext))
        .toList();
```

`PositiveQuantityRule`이든 `KillSwitchRule`이든 service 입장에서는 모두 `PreTradeRiskRule`이다.

이게 interface를 사용하는 가장 전형적인 효과였다.

서로 다른 구현체를 같은 타입으로 다룰 수 있다.

## interface와 abstract class를 함께 사용한 이유

pre-trade risk rule에서는 interface와 abstract class를 함께 사용했다.

```text
PreTradeRiskRule
    -> AbstractPreTradeRiskRule
        -> MaxOrderNotionalRule
        -> PriceBandRule
        -> KillSwitchRule
```

`PreTradeRiskRule`은 실행 계약이다.

반면 `AbstractPreTradeRiskRule`은 공통 helper를 제공한다.

```java
protected PreTradeRiskRuleCheckResult passed(...)
protected PreTradeRiskRuleCheckResult failed(...)
protected PreTradeRiskRuleCheckResult skipped(...)
protected String valueOf(Object value)
```

각 rule은 결과를 만들 때 `PASSED`, `FAILED`, `SKIPPED`를 반복해서 생성해야 한다.

이 반복 코드를 모든 rule에 복사하면 지저분해진다.

그래서 interface는 계약을 담당하고, abstract class는 구현 중복을 줄이는 역할을 맡겼다.

내가 정리한 차이는 이렇다.

```text
interface
    무엇을 할 수 있어야 하는가를 정의한다.

abstract class
    공통 구현을 제공하면서 일부 판단은 하위 클래스에 맡긴다.
```

이번 프로젝트에서는 이 둘의 역할을 섞지 않으려고 했다.

`PreTradeRiskRule`은 최대한 얇게 유지하고, 반복 구현은 `AbstractPreTradeRiskRule`로 보냈다.

## 새 규칙을 추가할 때 변경 범위 줄이기

리스크 규칙을 추가하는 경우를 생각해보면 interface의 장점이 더 잘 보인다.

예를 들어 일일 손실 한도 rule을 추가한다고 가정한다.

interface가 없다면 `PreTradeRiskCheckService` 안에 새로운 if 문과 결과 생성 로직을 추가해야 할 가능성이 높다.

하지만 현재 구조에서는 새 class를 만들면 된다.

```text
DailyLossLimitRule implements PreTradeRiskRule
```

그리고 rule list에 등록한다.

```java
private static List<PreTradeRiskRule> defaultRules() {
    return List.of(
            new PositiveQuantityRule(),
            new LimitPriceRequiredRule(),
            new DailyLossLimitRule()
    );
}
```

물론 현재는 rule list를 코드에서 직접 만들고 있으므로, 완전히 열려 있는 구조는 아니다.

나중에는 Spring bean으로 rule들을 주입받고 order를 정렬하는 방식으로 바꿀 수 있다.

그래도 중요한 변화는 이미 생겼다.

새 규칙의 판단 로직은 새 rule class에 고립된다.

기존 service는 rule 실행과 decision 집계라는 책임에 집중한다.

# interface를 쓰며 배운 기준

interface를 사용하면서 단순히 "많이 만들면 좋은 것"이 아니라, 언제 필요한지 판단하는 기준도 조금씩 생겼다.

## interface가 의미 있는 경우

이번 프로젝트를 하면서 interface를 무조건 많이 만드는 것이 좋은 설계는 아니라는 것도 배웠다.

interface가 의미 있으려면 최소한 아래 중 하나는 있어야 한다고 느꼈다.

- 구현체를 바꿀 가능성이 있다.
- application 계층이 infrastructure를 몰라야 한다.
- 여러 구현체를 같은 방식으로 실행해야 한다.
- 테스트에서 대체 구현을 쓰면 이점이 있다.
- 도메인 계약을 먼저 고정하는 것이 중요하다.

반대로 구현체가 하나뿐이고 바꿀 가능성도 없고, 다형성도 필요 없다면 interface가 오히려 파일만 늘릴 수 있다.

이번 프로젝트에서 repository interface는 DB 전환 가능성이 있어서 의미가 있었다.

`PreTradeRiskRule`은 여러 rule 구현체를 하나의 실행 계약으로 묶어야 해서 의미가 있었다.

즉, interface를 쓴 이유가 분명했다.

## 현재 구조의 한계

현재 구조에도 아직 개선할 부분이 있다.

첫째, repository interface는 잘 분리했지만 in-memory 구현은 실제 DB transaction을 보장하지 않는다.

예를 들어 post-settlement accounting workflow에서는 position ledger, cash ledger, average cost, realized PnL을 순서대로 posting한다.

현재는 각 repository의 idempotency로 중복 posting은 막고 있지만, 여러 저장소를 가로지르는 atomicity는 아직 없다.

실제 DB로 전환하면 이 흐름은 하나의 transaction으로 묶거나 outbox 기반 재처리를 고민해야 한다.

둘째, `PreTradeRiskCheckService`의 rule list가 아직 코드에 고정되어 있다.

향후 rule enable/disable, priority, 운영자 설정을 붙이려면 rule 등록 방식을 더 유연하게 바꿔야 한다.

예를 들면 다음처럼 Spring이 `PreTradeRiskRule` 구현체 목록을 주입하게 만들 수 있다.

```java
public PreTradeRiskCheckService(List<PreTradeRiskRule> rules, Clock clock) {
    this.rules = rules;
    this.clock = clock;
}
```

그 후 rule별 우선순위는 `@Order`나 별도 priority 값으로 관리할 수 있다.

# 정리

이번 프로젝트에서 interface는 단순 문법이 아니라 책임 경계를 만드는 도구였다.

repository interface는 application service가 저장소 구현을 모르게 해주었다.

`PreTradeRiskRule` interface는 서로 다른 리스크 규칙을 같은 방식으로 실행할 수 있게 해주었다.

내가 이번 프로젝트에서 정리한 interface 사용 기준은 다음과 같다.

1. 구현체보다 먼저 계약을 정의하고 싶을 때 사용한다.
2. service가 infrastructure 구현을 직접 알면 안 될 때 사용한다.
3. 여러 구현체를 같은 방식으로 다루고 싶을 때 사용한다.
4. 테스트에서 구현을 바꿔 끼우면 이점이 있을 때 사용한다.
5. 반복 구현은 interface가 아니라 abstract class나 helper로 분리한다.

예전에는 interface를 "객체지향에서 다형성을 위해 쓰는 것" 정도로만 생각했다.

지금은 조금 더 구체적으로 이해하고 있다.

프로젝트에서 interface는 "지금 필요한 기능의 계약을 먼저 세우고, 구현 세부사항이 application 로직을 흔들지 않도록 막는 경계"에 가깝다.
