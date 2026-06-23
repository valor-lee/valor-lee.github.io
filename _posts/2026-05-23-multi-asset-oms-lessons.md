---
title: '[프로젝트] 멀티자산 OMS 시뮬레이션 플랫폼을 만들며 배운 것'
date: 2026-05-23 00:30:00 +09:00
categories: [Project, OMS]
tags:
  [
    project,
    backend,
    Java,
    Spring,
    OMS,
    trading,
    domain modeling
  ]
---

# 개요

최근 `multi-asset-oms-simulation-platform` 프로젝트를 진행하면서 멀티자산 주문관리시스템(OMS)의 기본 흐름을 직접 구현하고 있다.

처음에는 "주문을 생성하고 체결시키는 시스템" 정도로 생각했지만, 구현을 시작하니 실제 핵심은 단순 주문 API가 아니었다.

이 프로젝트에서 계속 마주친 핵심 문제는 다음과 같았다.

- 주문은 바로 시장으로 나가는 것이 아니라, 먼저 "주문 의도"로 표현된다.
- 주문 의도는 사전 리스크 검사를 통과해야 실제 주문이 된다.
- 주문은 외부 시장 응답에 따라 여러 상태를 가진다.
- ACK, FILL, CANCEL 같은 이벤트는 중복으로 들어올 수 있다.
- 체결 이후에도 trade capture, settlement, position ledger, cash ledger가 이어진다.
- 상태 전이와 원장 반영은 정합성이 가장 중요하다.

즉, OMS는 "주문 넣기" 기능이 아니라 주문의 전체 생명주기를 안정적으로 관리하는 시스템에 가깝다.

# 전체 흐름

현재 프로젝트는 대략 아래 흐름을 목표로 구현하고 있다.

```text
Manual / Strategy / Rebalancing
    -> OrderIntent
    -> Pre-Trade Risk
    -> Order
    -> Submission
    -> ACK / REJECT
    -> Fill / Cancel
    -> Trade Capture
    -> Settlement
    -> Position Ledger / Cash Ledger
```

여기서 중요한 점은 각 단계의 책임이 다르다는 것이다.

`OrderIntent`는 아직 실제 시장 주문이 아니다.

수동 입력, 리밸런싱, 전략 신호처럼 서로 다른 주문 소스를 하나의 공통 형태로 모은 것이다. 이 단계에서는 "무엇을 얼마나 사고팔고 싶은가"를 표현한다.

반면 `Order`는 execution 계층에서 실제 주문 생명주기를 관리하는 대상이다. 시장으로 전송되고, 접수되고, 체결되고, 취소되는 상태는 `Order`가 가진다.

그리고 `Trade`는 체결 결과를 post-trade 영역에서 관리하기 위한 모델이다. `Order`가 주문의 상태를 말한다면, `Trade`는 실제로 발생한 거래 사실을 말한다.

# 1. 주문 의도와 주문은 분리해야 한다

처음 배운 중요한 개념은 `OrderIntent`와 `Order`를 분리하는 것이었다.

수동 주문, 전략 기반 주문, 리밸런싱 주문은 모두 출발점이 다르다. 하지만 이후 리스크 검사와 실행 흐름에서는 같은 방식으로 처리되어야 한다.

그래서 프로젝트에서는 앞단의 다양한 입력을 `OrderIntent`로 통합했다.

```text
ManualOrderIntentRequest
StrategyOrderIntentRequest
RebalancingOrderIntentRequest
        -> OrderIntent
```

이렇게 하면 주문 소스가 늘어나도 리스크 계층이나 execution 계층은 입력 출처를 몰라도 된다.

내가 이해한 핵심은 다음과 같다.

- `OrderIntent`는 주문을 내고 싶다는 의사 표현이다.
- `Order`는 OMS가 실제 실행 대상으로 관리하는 주문이다.
- 리스크 승인 전까지는 `Order`를 만들지 않는다.
- 주문 소스가 달라도 공통 파이프라인에 태울 수 있다.

이 분리를 해두니 이후 `CREATED -> RISK_APPROVED -> CONVERTED_TO_ORDER` 흐름을 만들기가 훨씬 명확해졌다.

# 2. 상태 전이는 불변 스냅샷으로 다룰 수 있다

프로젝트에서는 `OrderIntent`를 record로 만들고, 상태 전이 시 기존 객체를 직접 수정하지 않고 새 인스턴스를 반환하는 방향으로 구현했다.

예를 들어 risk 평가 전 intent는 `CREATED` 상태이고, risk 승인 이후에는 `RISK_APPROVED` 상태의 새 intent를 저장한다.

이 방식의 장점은 상태 전후가 명확하게 남는다는 것이다.

처음에는 객체를 그냥 변경하면 더 간단하지 않을까 생각했지만, 주문 시스템에서는 "언제 어떤 상태였는가"가 중요하다. 기존 객체를 직접 변경하면 이전 상태를 바라보던 흐름도 같이 영향을 받을 수 있다.

불변 스냅샷 방식으로 보면 다음과 같은 이점이 있다.

- risk 평가 전 `CREATED` intent와 평가 후 `RISK_APPROVED` intent를 구분할 수 있다.
- 테스트에서 상태 전이 전후를 비교하기 쉽다.
- 감사 로그나 이벤트 모델로 확장하기 좋다.
- 여러 흐름이 같은 객체를 참조할 때 예상치 못한 변경을 줄일 수 있다.

주문 시스템에서 상태는 단순 필드가 아니라 업무 이벤트의 결과라는 점을 배웠다.

# 3. 리스크 검사는 rule 단위로 분리하는 것이 좋다

초기 pre-trade risk 구현에서는 서비스 안에 규칙 판단 로직이 모여 있었다.

처음에는 규칙이 적어서 문제가 없었지만, 아래 규칙들이 추가되면서 서비스가 빠르게 커졌다.

- 양수 수량 검사
- 지정가 주문 가격 필수 검사
- 양수 지정가 검사
- 최대 주문 수량 검사
- 최대 주문 금액 검사
- 최대 포지션 수량 검사
- 중복 open order 검사
- price band 검사
- kill switch 검사

그래서 `PreTradeRiskRule` 인터페이스를 만들고, 각 규칙을 별도 클래스로 분리했다.

```text
PreTradeRiskRule
    -> PositiveQuantityRule
    -> LimitPriceRequiredRule
    -> MaxOrderNotionalRule
    -> PriceBandRule
    -> KillSwitchRule
```

그리고 `PreTradeRiskCheckService`는 각 rule을 실행하고 결과를 모아 최종 decision을 만드는 역할만 담당하게 했다.

이 구조에서 배운 점은 interface와 abstract class의 역할 차이다.

`PreTradeRiskRule` 인터페이스는 모든 rule이 지켜야 하는 실행 계약이다.

```text
입력: PreTradeRiskCheckCommand, PreTradeRiskCheckContext
출력: PreTradeRiskRuleCheckResult
```

반면 `AbstractPreTradeRiskRule`은 `passed`, `failed`, `skipped` 같은 결과 생성 helper를 제공한다.

즉, interface는 "무엇을 할 수 있어야 하는가"를 정하고, abstract class는 "반복되는 구현을 어떻게 공유할 것인가"를 담당한다.

이 분리를 하고 나니 새 규칙을 추가할 때 기존 서비스 전체를 건드리지 않아도 되었다.

# 4. 리스크 입력 컨텍스트는 한곳에 섞지 않는 것이 좋다

리스크 검사를 하다 보면 다양한 외부 정보가 필요해진다.

- 한도 정보
- 현재 포지션
- open order 존재 여부
- 시장 가격 밴드
- kill switch 설정

처음에는 이런 값을 하나의 context에 계속 추가하면 될 것처럼 보였다. 하지만 그렇게 하면 한도값, 현재 노출, 시장 데이터, 운영 제어값이 한 객체에 섞인다.

그래서 프로젝트에서는 context를 성격별로 나누었다.

```text
PreTradeRiskLimitContext
PreTradeRiskExposureContext
PreTradeRiskOpenOrderContext
PreTradeRiskMarketContext
PreTradeRiskControlContext
```

그리고 `PreTradeRiskCheckContext`가 이들을 묶어 전달한다.

이 설계에서 배운 점은 입력값도 도메인 의미에 따라 나눠야 한다는 것이다.

단순히 "검사에 필요한 값 모음"으로 보면 하나의 DTO에 다 넣고 싶어진다. 하지만 한도, 현재 상태, 시장 정보, 운영 제어값은 변경 주체와 조회 경로가 다르다.

미리 분리해두면 이후 DB, 외부 API, 캐시, 운영자 설정 등으로 연결할 때 확장하기 좋다.

# 5. 주문 상태는 외부 시장 응답을 반영해야 한다

execution 계층을 구현하면서 주문은 생각보다 많은 상태를 가진다는 것을 체감했다.

현재 프로젝트에서는 다음 상태들을 다루고 있다.

```text
CREATED
SENT
ACKED
REJECTED
PARTIALLY_FILLED
FILLED
CANCEL_REQUESTED
CANCELED
```

중요한 점은 주문을 보냈다고 바로 체결되는 것이 아니라는 것이다.

먼저 broker나 exchange가 주문을 받았는지 `ACK`가 와야 하고, 이후 일부만 체결될 수도 있고, 전량 체결될 수도 있고, 취소 요청 이후에도 체결 이벤트가 먼저 도착할 수 있다.

특히 `CANCEL_REQUESTED` 상태에서도 fill을 허용한 부분이 기억에 남는다.

운영 관점에서는 취소 요청을 보냈다고 해서 시장에서 즉시 취소되는 것이 아니다. 취소 요청과 체결 이벤트는 경쟁할 수 있다.

그래서 `CANCEL_REQUESTED` 이후에도 fill을 반영할 수 있어야 실제 시장 흐름과 조금 더 가까워진다.

# 6. 멱등성은 주문 시스템에서 필수다

이번 프로젝트에서 가장 반복해서 등장한 단어는 idempotency였다.

외부 broker나 exchange에서 오는 이벤트는 네트워크 재시도, consumer 재처리, 장애 복구 과정에서 중복으로 들어올 수 있다.

만약 같은 fill 이벤트를 두 번 더하면 체결 수량이 실제보다 커진다.

그래서 fill 이벤트에는 `fillExecutionId`를 두고, 이미 처리한 체결이면 수량을 다시 더하지 않도록 했다.

```text
OrderFillExecution
    - fillExecutionId
    - orderId
    - fillQuantity
    - fillPrice
```

ACK, REJECT, CANCEL confirmation 같은 상태 전이 이벤트도 마찬가지다.

이들은 `OrderExecutionEvent`로 따로 저장하고, 같은 event id가 다시 들어오면 현재 order를 반환한다.

여기서 배운 점은 이벤트의 성격에 따라 멱등성 저장 모델도 달라질 수 있다는 것이다.

- fill은 수량 누적 이벤트이므로 `OrderFillExecution`으로 관리한다.
- ACK, REJECT, CANCEL confirmation은 상태 전이 이벤트이므로 `OrderExecutionEvent`로 관리한다.

둘 다 중복 방지가 필요하지만, 비즈니스 의미가 다르기 때문에 같은 테이블이나 같은 모델로 뭉개지 않는 것이 좋다.

# 7. post-trade는 체결 이후의 별도 세계다

주문이 `FILLED`가 되면 끝이라고 생각하기 쉽다.

하지만 post-trade 관점에서는 이제 시작이다.

현재 프로젝트에서는 다음 흐름을 구현했다.

```text
Order
    -> Trade Capture
    -> Settlement
    -> Position Ledger
    -> Cash Ledger
```

`TradeCaptureService`는 execution에서 완료된 order를 post-trade의 `Trade`로 캡처한다.

여기서 중요한 처리는 다음과 같다.

- `FILLED` order는 전체 체결 수량을 trade로 캡처한다.
- `CANCELED` order라도 체결 수량이 있으면 부분 체결 trade로 캡처한다.
- 이미 캡처된 order는 기존 trade를 반환해 중복 capture를 막는다.
- fill price가 모두 있으면 수량 가중 평균 체결가와 총 체결금액을 계산한다.

평균 체결가는 아래처럼 계산한다.

```text
averageFillPrice = sum(fillQuantity * fillPrice) / sum(fillQuantity)
grossNotional = sum(fillQuantity * fillPrice)
```

이후 settlement에서는 trade를 결제 예정 상태로 만들고, 결제가 완료되면 trade를 `SETTLED`로 전이한다.

그리고 settled trade만 position ledger와 cash ledger에 반영한다.

```text
BUY  -> position +quantity, cash -grossNotional
SELL -> position -quantity, cash +grossNotional
```

여기서 배운 점은 주문 상태와 실제 자산/현금 반영은 분리해야 한다는 것이다.

체결이 되었다고 곧바로 원장에 반영하는 것이 아니라, settlement를 거친 뒤 position ledger와 cash ledger에 기록한다. 이 흐름을 나눠두면 나중에 수수료, 세금, 환전, 결제일, 통화별 잔고를 확장하기 쉬워진다.

# 8. 저장소 포트를 먼저 세우면 구현을 천천히 바꿀 수 있다

현재 프로젝트는 아직 DB/JPA를 붙이지 않고 in-memory repository를 많이 사용한다.

하지만 서비스가 직접 Map을 들고 있는 것이 아니라 repository port를 바라보도록 만들었다.

예를 들면 다음과 같다.

```text
OrderIntentRepository
OrderRepository
OrderFillExecutionRepository
TradeRepository
SettlementRepository
PositionLedgerRepository
CashLedgerRepository
```

그리고 현재는 `InMemory...Repository`가 adapter 역할을 한다.

이렇게 해두면 지금은 빠르게 도메인 규칙과 상태 전이를 테스트할 수 있고, 이후 JPA나 PostgreSQL로 옮길 때 application service의 핵심 로직을 크게 바꾸지 않아도 된다.

이번 프로젝트에서 "인프라 구현보다 도메인 계약을 먼저 고정한다"는 감각을 많이 배웠다.

# 9. 시간은 직접 만들지 말고 주입받는 것이 좋다

초기에는 서비스 내부에서 `Instant.now()`나 `Clock.systemUTC()`를 직접 호출할 수 있다.

하지만 테스트에서 시간 검증이 들어가면 금방 불안정해진다.

그래서 프로젝트에서는 공통 `Clock` bean을 등록하고, 시간 의존 서비스가 생성자 주입으로 `Clock`을 받도록 정리했다.

이렇게 바꾸니 테스트에서 고정 clock을 사용할 수 있었다.

특히 order conversion에서는 하나의 변환 동작 안에서 생성된 `Order.createdAt`과 `OrderIntent.updatedAt`이 같은 기준 시각을 사용하도록 만들 수 있었다.

시간은 단순 유틸이 아니라 도메인 이벤트의 기준값이다.

그래서 시간 정책도 주입 가능한 의존성으로 다루는 것이 좋다는 점을 배웠다.

# 10. 실제 DB에서는 transaction 경계가 중요하다

현재는 in-memory 구현이지만, 구현하면서 계속 transaction 경계를 생각하게 됐다.

예를 들어 settlement를 등록할 때는 다음 두 작업이 함께 성공해야 한다.

- settlement row 생성
- trade 상태를 `SETTLEMENT_PENDING`으로 전이

둘 중 하나만 성공하면 상태가 꼬인다.

fill 이벤트 처리도 마찬가지다.

- fill execution 저장
- order filled quantity와 status 갱신

이 둘이 함께 커밋되지 않으면 중복 이벤트 방어나 주문 상태 정합성이 깨질 수 있다.

아직 DB를 붙이지 않았지만, in-memory 단계에서도 "어떤 작업들이 하나의 transaction이어야 하는가"를 미리 생각해두는 것이 중요했다.

# 정리

이번 프로젝트를 진행하면서 OMS는 단순 CRUD 프로젝트가 아니라 상태, 이벤트, 정합성, 멱등성, 감사 가능성을 계속 고민해야 하는 도메인이라는 것을 배웠다.

현재까지 가장 크게 배운 내용을 정리하면 다음과 같다.

1. 주문 의도와 실제 주문은 분리해야 한다.
2. 상태 전이는 도메인 이벤트의 결과로 봐야 한다.
3. 리스크 규칙은 독립 rule로 분리하면 확장하기 좋다.
4. 외부 이벤트는 중복으로 들어올 수 있으므로 멱등성이 필요하다.
5. 체결 이후에는 trade, settlement, ledger 흐름이 따로 존재한다.
6. 시간과 저장소는 주입 가능한 의존성으로 다루면 테스트와 확장이 쉬워진다.
7. in-memory 구현 단계에서도 transaction 경계를 계속 의식해야 한다.

아직 market data, replay, audit, PnL, 실제 DB 영속화, 운영자 화면 등 남은 부분이 많다.

그래도 지금까지의 slice를 통해 "금융 시스템에서 왜 상태 관리와 정합성이 중요한가"를 조금씩 체감하고 있다.

다음에는 execution event replay나 audit log를 붙이면서, 지금까지 쌓은 상태 전이와 이벤트 기록을 어떻게 재현 가능한 구조로 만들 수 있을지 정리해보고자 한다.
