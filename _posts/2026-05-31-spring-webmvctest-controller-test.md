---
title: '[Spring] @WebMvcTest와 MockMvc로 Controller 테스트 이해하기'
date: 2026-05-31 12:05:00 +09:00
categories: [Framework, Spring]
tags:
  [
    backend,
    Java,
    Spring,
    Spring Boot,
    test,
    WebMvcTest,
    MockMvc
  ]
---

# 개요

`multi-asset-oms-simulation-platform` 프로젝트에서 replay consistency report를 HTTP API로 노출하면서 Controller 테스트를 작성했다.

테스트 파일에는 아래와 같은 Spring 테스트 어노테이션과 문법이 등장했다.

- `@WebMvcTest`
- `@ContextConfiguration`
- `@SpringBootConfiguration`
- `@Autowired`
- `@MockBean`
- `MockMvc`
- `when(...).thenReturn(...)`
- `jsonPath(...)`

처음 보면 이 코드가 실제 서버를 띄우는 것인지, service 로직까지 실행하는 것인지, JSON 검증은 어떻게 하는 것인지 헷갈릴 수 있다.

이번 글에서는 `OrderReplayConsistencyReportControllerTest`를 기준으로 Controller 테스트의 원리와 문법을 정리한다.

# 테스트 대상 Controller

테스트 대상은 replay consistency report를 조회하는 Controller다.

```java
@RestController
@RequestMapping("/api/audit-replay/order-replay/consistency-report")
public class OrderReplayConsistencyReportController {

    private final OrderReplayConsistencyReportService reportService;

    public OrderReplayConsistencyReportController(OrderReplayConsistencyReportService reportService) {
        this.reportService = reportService;
    }

    @GetMapping
    public OrderReplayConsistencyReport report(
            @RequestParam(name = "inconsistentOnly", defaultValue = "false") boolean inconsistentOnly
    ) {
        if (inconsistentOnly) {
            return reportService.checkInconsistentOnly();
        }
        return reportService.checkAll();
    }
}
```

이 Controller가 하는 일은 단순하다.

- `GET /api/audit-replay/order-replay/consistency-report` 요청을 받는다.
- query parameter `inconsistentOnly`가 `true`면 불일치 주문만 담은 report를 반환한다.
- query parameter가 없거나 `false`면 전체 report를 반환한다.

여기서 중요한 점은 Controller가 직접 replay 계산을 하지 않는다는 것이다.

실제 계산은 `OrderReplayConsistencyReportService`가 담당한다. Controller는 HTTP 요청을 service 호출로 연결하고, service 결과를 JSON 응답으로 반환하는 역할만 맡는다.

# Controller 테스트의 목적

Controller 테스트는 비즈니스 로직 전체를 검증하는 테스트가 아니다.

이 테스트가 확인해야 하는 것은 다음과 같다.

- URL이 올바르게 매핑되는가?
- query parameter가 올바르게 처리되는가?
- Controller가 적절한 service 메서드를 호출하는가?
- 반환 객체가 JSON 응답으로 잘 변환되는가?
- HTTP status가 기대한 값인가?
- JSON 응답 필드 이름과 값이 API 계약에 맞는가?

반대로 아래 내용은 Controller 테스트의 주 관심사가 아니다.

- repository에서 주문을 잘 조회하는가?
- audit trail을 올바르게 replay하는가?
- consistency mismatch reason을 정확히 계산하는가?
- 전체 주문 집계가 정확히 계산되는가?

그런 로직은 service test에서 검증하는 편이 좋다.

즉 Controller 테스트는 아래 흐름만 가볍게 확인한다.

```text
HTTP request
    -> Controller
    -> Mock service
    -> JSON response
```

# 테스트 코드 전체 모양

테스트는 대략 아래 구조다.

```java
@WebMvcTest(OrderReplayConsistencyReportController.class)
@ContextConfiguration(classes = {
        OrderReplayConsistencyReportControllerTest.TestApplication.class,
        OrderReplayConsistencyReportController.class
})
class OrderReplayConsistencyReportControllerTest {

    @SpringBootConfiguration
    static class TestApplication {
    }

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderReplayConsistencyReportService reportService;

    @Test
    void returnsAllConsistencyReportByDefault() throws Exception {
        when(reportService.checkAll()).thenReturn(...);

        mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.totalCount").value(2));
    }
}
```

이제 하나씩 뜯어보자.

# @WebMvcTest

```java
@WebMvcTest(OrderReplayConsistencyReportController.class)
```

`@WebMvcTest`는 Spring MVC 계층만 테스트할 때 사용하는 어노테이션이다.

전체 Spring Boot 애플리케이션을 다 띄우는 것이 아니라, Controller 테스트에 필요한 최소한의 MVC 구성만 로딩한다.

대표적으로 아래 기능들이 테스트 대상이 된다.

- `@Controller`, `@RestController`
- URL mapping
- `@RequestParam`
- `@RequestBody`
- JSON serialization / deserialization
- validation
- Spring MVC exception handling
- `MockMvc`

반면 service, repository, DB 연결 같은 계층은 기본적으로 로딩하지 않는다.

그래서 `@WebMvcTest`는 일반적인 통합 테스트보다 빠르다. 대신 Controller가 의존하는 service는 직접 mock으로 등록해야 한다.

# @ContextConfiguration

```java
@ContextConfiguration(classes = {
        OrderReplayConsistencyReportControllerTest.TestApplication.class,
        OrderReplayConsistencyReportController.class
})
```

`@ContextConfiguration`은 테스트용 Spring Context에 어떤 클래스를 등록할지 지정한다.

이번 프로젝트의 `audit-replay` 모듈은 독립적인 라이브러리 모듈이다. 그래서 테스트에서 Spring Boot 설정 클래스를 자동으로 찾지 못할 수 있다.

그럴 때 테스트 안에 작은 설정 클래스를 두고, Controller와 함께 명시적으로 등록한다.

```java
@SpringBootConfiguration
static class TestApplication {
}
```

이 클래스는 비어 있지만 의미가 있다.

Spring Boot 테스트가 "이 클래스를 기준 설정으로 사용하면 되겠다"고 인식하게 해준다.

# @Autowired MockMvc

```java
@Autowired
private MockMvc mockMvc;
```

`MockMvc`는 실제 서버를 띄우지 않고 HTTP 요청을 흉내 내는 테스트 도구다.

예를 들어 아래 코드는 실제 네트워크 요청이 아니다.

```java
mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report"))
```

하지만 Spring MVC 내부에서는 실제 HTTP 요청처럼 처리된다.

흐름은 다음과 같다.

```text
MockMvc GET request
    -> DispatcherServlet
    -> Controller method
    -> return object
    -> JSON response
    -> test assertion
```

즉 port를 열지 않고도 Controller의 HTTP 동작을 검증할 수 있다.

# @MockBean

```java
@MockBean
private OrderReplayConsistencyReportService reportService;
```

`@MockBean`은 Spring Context에 Mockito mock 객체를 bean으로 등록한다.

Controller는 생성자에서 `OrderReplayConsistencyReportService`를 주입받는다.

```java
public OrderReplayConsistencyReportController(OrderReplayConsistencyReportService reportService) {
    this.reportService = reportService;
}
```

테스트에서는 실제 service가 아니라 mock service가 주입된다.

이렇게 하는 이유는 Controller 테스트의 범위를 좁히기 위해서다.

Controller 테스트에서 실제 service를 사용하면 아래 로직까지 모두 따라 들어간다.

- repository 조회
- audit event 조회
- replay 실행
- consistency 계산
- mismatch reason 계산

그러면 Controller 테스트가 무거워지고, 실패 원인도 흐려진다.

여기서는 service가 특정 값을 반환한다고 가정하고, Controller가 그 값을 HTTP 응답으로 잘 내려주는지만 확인한다.

# Mockito의 when(...).thenReturn(...)

```java
when(reportService.checkAll()).thenReturn(new OrderReplayConsistencyReport(
        2,
        1,
        1,
        List.of(...),
        Instant.parse("2026-05-31T02:00:00Z")
));
```

이 문법은 Mockito에서 사용하는 stubbing이다.

읽는 방법은 간단하다.

```text
reportService.checkAll()이 호출되면
thenReturn(...) 안의 객체를 반환해라.
```

즉 service 로직을 실제로 실행하지 않고, 테스트가 원하는 결과를 미리 정해두는 것이다.

예를 들어 전체 report API 테스트에서는 `checkAll()`이 호출되면 아래 의미의 데이터를 반환하게 만든다.

```text
totalCount = 2
consistentCount = 1
inconsistentCount = 1
results = 주문별 검증 결과 목록
checkedAt = 2026-05-31T02:00:00Z
```

# record 생성자 이해하기

테스트에서는 `OrderReplayConsistencyReport` record를 직접 생성한다.

```java
new OrderReplayConsistencyReport(
        2,
        1,
        1,
        List.of(...),
        Instant.parse("2026-05-31T02:00:00Z")
)
```

record는 선언된 필드 순서대로 생성자 인자를 받는다.

```java
public record OrderReplayConsistencyReport(
        int totalCount,
        int consistentCount,
        int inconsistentCount,
        List<OrderReplayConsistencyResult> results,
        Instant checkedAt
) {
}
```

따라서 위 생성자는 아래와 같은 의미다.

```text
totalCount = 2
consistentCount = 1
inconsistentCount = 1
results = List.of(...)
checkedAt = Instant.parse(...)
```

# List.of(...)

```java
List.of(
        result(...),
        result(...)
)
```

`List.of(...)`는 Java에서 불변 리스트를 만드는 문법이다.

테스트 데이터처럼 "이 목록은 이 값들로 고정된다"는 의미를 표현할 때 자주 쓴다.

주의할 점은 `List.of(...)`로 만든 리스트는 수정할 수 없다는 것이다.

```java
List<String> values = List.of("A", "B");
values.add("C"); // UnsupportedOperationException
```

테스트 데이터에는 오히려 이 특성이 좋다. 테스트 중간에 실수로 값이 바뀌는 일을 줄일 수 있다.

# mockMvc.perform(...)

```java
mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report"))
```

`perform(...)`은 MockMvc로 요청을 실행하는 메서드다.

`get(...)`은 GET 요청을 만든다.

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
```

이 static import 덕분에 원래는 아래처럼 써야 하는 코드를 짧게 쓸 수 있다.

```java
MockMvcRequestBuilders.get("/api/audit-replay/order-replay/consistency-report")
```

요청에 query parameter를 붙이고 싶으면 `.param(...)`을 사용한다.

```java
mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report")
        .param("inconsistentOnly", "true"))
```

이 요청은 실제 HTTP로 보면 아래와 같다.

```http
GET /api/audit-replay/order-replay/consistency-report?inconsistentOnly=true
```

# andExpect(...)

```java
.andExpect(status().isOk())
.andExpect(jsonPath("$.totalCount").value(2))
```

`andExpect(...)`는 응답에 대한 기대값을 검증한다.

`status().isOk()`는 HTTP status가 200인지 확인한다.

```java
.andExpect(status().isOk())
```

만약 URL mapping이 틀렸거나 Controller에서 예외가 발생하면 이 검증은 실패한다.

# jsonPath(...)

```java
.andExpect(jsonPath("$.totalCount").value(2))
```

`jsonPath`는 JSON 응답에서 특정 위치의 값을 꺼내 검증하는 문법이다.

예를 들어 응답 JSON이 아래와 같다고 하자.

```json
{
  "totalCount": 2,
  "consistentCount": 1,
  "inconsistentCount": 1,
  "results": [
    {
      "orderId": "00000000-0000-0000-0000-000000018001",
      "consistent": true,
      "mismatchReasons": []
    },
    {
      "orderId": "00000000-0000-0000-0000-000000018002",
      "consistent": false,
      "mismatchReasons": ["FILLED_QUANTITY_MISMATCH"]
    }
  ],
  "checkedAt": "2026-05-31T02:00:00Z"
}
```

그러면 각 jsonPath는 아래 의미다.

```java
jsonPath("$.totalCount").value(2)
```

최상위 JSON 객체의 `totalCount` 값이 2인지 확인한다.

```java
jsonPath("$.results[0].orderId")
```

`results` 배열의 첫 번째 객체의 `orderId`를 가리킨다.

```java
jsonPath("$.results[1].mismatchReasons[0]")
```

`results` 배열의 두 번째 객체 안에 있는 `mismatchReasons` 배열의 첫 번째 값을 가리킨다.

```java
jsonPath("$.results.length()").value(1)
```

`results` 배열의 길이가 1인지 확인한다.

# @RequestParam name을 명시해야 하는 이유

Controller에는 아래 코드가 있다.

```java
@RequestParam(name = "inconsistentOnly", defaultValue = "false") boolean inconsistentOnly
```

처음에는 이렇게 쓸 수도 있다고 생각할 수 있다.

```java
@RequestParam(defaultValue = "false") boolean inconsistentOnly
```

하지만 이 경우 Spring은 Java reflection으로 메서드 파라미터 이름을 알아내려고 한다.

현재 Gradle 컴파일 설정에서 parameter name 정보가 보장되지 않으면 아래와 같은 오류가 날 수 있다.

```text
Name for argument of type [boolean] not specified,
and parameter name information not available via reflection.
```

그래서 query parameter 이름은 명시적으로 쓰는 편이 안전하다.

```java
@RequestParam(name = "inconsistentOnly", defaultValue = "false")
```

이렇게 하면 컴파일 옵션과 무관하게 Spring이 어떤 query parameter를 읽어야 하는지 알 수 있다.

# helper method를 둔 이유

테스트 아래쪽에는 `result(...)` helper method가 있다.

```java
private OrderReplayConsistencyResult result(
        String orderId,
        boolean consistent,
        List<OrderReplayMismatchReason> mismatchReasons,
        OrderStatus actualStatus,
        OrderStatus replayedStatus,
        String actualFilledQuantity,
        String replayedFilledQuantity
) {
    return new OrderReplayConsistencyResult(
            UUID.fromString(orderId),
            consistent,
            mismatchReasons,
            actualStatus,
            replayedStatus,
            new BigDecimal(actualFilledQuantity),
            new BigDecimal(replayedFilledQuantity),
            2,
            Instant.parse("2026-05-31T02:00:00Z")
    );
}
```

이 helper method는 테스트 데이터 생성을 짧게 만들기 위한 것이다.

만약 helper가 없다면 테스트 본문마다 `new OrderReplayConsistencyResult(...)`를 길게 작성해야 한다.

그렇게 되면 테스트의 핵심인 HTTP 요청과 JSON 검증보다 테스트 데이터 생성 코드가 더 눈에 띄게 된다.

helper를 사용하면 테스트 본문에서 중요한 값만 드러낼 수 있다.

```java
result(
        "00000000-0000-0000-0000-000000018002",
        false,
        List.of(OrderReplayMismatchReason.FILLED_QUANTITY_MISMATCH),
        OrderStatus.PARTIALLY_FILLED,
        OrderStatus.PARTIALLY_FILLED,
        "3",
        "4"
)
```

이 코드는 아래 의미를 바로 보여준다.

```text
이 주문은 inconsistent 상태다.
mismatch reason은 FILLED_QUANTITY_MISMATCH다.
실제 filled quantity는 3이고 replay 결과는 4다.
```

# 첫 번째 테스트 흐름

첫 번째 테스트는 query parameter 없이 요청했을 때 전체 report를 반환하는지 확인한다.

```java
@Test
void returnsAllConsistencyReportByDefault() throws Exception {
    when(reportService.checkAll()).thenReturn(...);

    mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.totalCount").value(2))
            .andExpect(jsonPath("$.consistentCount").value(1))
            .andExpect(jsonPath("$.inconsistentCount").value(1));
}
```

테스트 흐름은 다음과 같다.

```text
1. reportService.checkAll()의 반환값을 미리 정한다.
2. query parameter 없이 GET 요청을 보낸다.
3. Controller는 inconsistentOnly 기본값 false를 사용한다.
4. Controller는 reportService.checkAll()을 호출한다.
5. mock service는 미리 정한 report를 반환한다.
6. Spring MVC가 report를 JSON으로 변환한다.
7. 테스트가 status와 JSON 필드를 검증한다.
```

# 두 번째 테스트 흐름

두 번째 테스트는 `inconsistentOnly=true`로 요청했을 때 불일치 주문만 들어 있는 report를 반환하는지 확인한다.

```java
@Test
void returnsOnlyInconsistentReportWhenRequested() throws Exception {
    when(reportService.checkInconsistentOnly()).thenReturn(...);

    mockMvc.perform(get("/api/audit-replay/order-replay/consistency-report")
                    .param("inconsistentOnly", "true"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.results.length()").value(1))
            .andExpect(jsonPath("$.results[0].mismatchReasons[0]").value("STATUS_MISMATCH"));
}
```

테스트 흐름은 다음과 같다.

```text
1. reportService.checkInconsistentOnly()의 반환값을 미리 정한다.
2. inconsistentOnly=true query parameter와 함께 GET 요청을 보낸다.
3. Controller는 inconsistentOnly 값을 true로 받는다.
4. Controller는 reportService.checkInconsistentOnly()를 호출한다.
5. mock service는 미리 정한 report를 반환한다.
6. 테스트는 결과 목록이 1개인지, mismatch reason이 맞는지 확인한다.
```

# 이 테스트를 한 문장으로 요약하면

이 테스트는 "HTTP 요청이 들어왔을 때 Controller가 service 결과를 올바른 JSON 응답으로 변환하는가"를 검증한다.

service 로직 자체를 믿고 넘어가는 것이 아니라, service 로직은 별도 service test에서 검증한다.

Controller test와 service test의 책임을 나누면 테스트가 더 읽기 쉬워진다.

```text
Controller test
    HTTP 요청, parameter, status, JSON 응답 검증

Service test
    비즈니스 로직, 계산, 상태 판단, repository 연동 검증
```

# 마무리

`@WebMvcTest`와 `MockMvc`는 Controller 계층을 빠르게 테스트하기 위한 도구다.

처음에는 어노테이션이 많아서 복잡해 보이지만, 핵심 원리는 단순하다.

```text
Controller만 띄운다.
Service는 mock으로 바꾼다.
MockMvc로 HTTP 요청을 흉내 낸다.
응답 status와 JSON을 검증한다.
```

이 감각을 잡으면 Controller 테스트가 훨씬 덜 낯설어진다.

그리고 query parameter처럼 외부 요청과 연결되는 값은 가능하면 이름을 명시하는 편이 좋다.

```java
@RequestParam(name = "inconsistentOnly", defaultValue = "false")
```

작은 차이지만 테스트와 운영 환경 모두에서 더 안정적인 API 코드를 만들 수 있다.
