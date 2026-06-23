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

# Controller 테스트 전에 알아야 할 Spring MVC

`@WebMvcTest`와 `MockMvc`를 이해하려면 먼저 Spring MVC가 어떤 역할을 하는지 알아야 한다.

웹 애플리케이션은 결국 클라이언트의 HTTP 요청을 받아서 응답을 돌려주는 프로그램이다. 예를 들어 브라우저나 프론트엔드가 아래처럼 요청을 보낸다고 하자.

```http
GET /api/audit-replay/order-replay/consistency-report
```

서버는 이 요청을 받아서 알맞은 Java 메서드를 실행하고, 그 결과를 HTTP 응답으로 변환해야 한다.

Spring MVC는 이 과정을 편하게 만들어주는 Spring의 웹 계층 프레임워크다.

내가 이해한 Spring MVC의 핵심은 다음과 같다.

```text
HTTP 요청을 Java 메서드 호출로 연결하고,
Java 객체를 HTTP 응답으로 다시 바꿔주는 구조
```

우리가 Controller에 `@GetMapping`, `@RequestParam`, `@RequestBody`, `@ResponseBody`, `@RestController` 같은 어노테이션을 붙이면 Spring MVC가 그 사이의 복잡한 연결 작업을 대신해준다.

# Servlet이란?

Spring MVC는 내부적으로 Servlet API 위에서 동작한다.

Servlet은 Java로 웹 요청을 처리하기 위한 표준 규약이다. 쉽게 말하면 "HTTP 요청이 들어왔을 때 실행될 Java 클래스는 이런 방식으로 만들어야 한다"는 약속이다.

하지만 실제 개발자가 매번 Servlet 클래스를 직접 만들고, 요청 URL을 직접 분기하고, parameter를 직접 파싱하고, 응답을 직접 작성하면 코드가 금방 복잡해진다.

그래서 Spring MVC는 Servlet 기반 구조 위에 더 편한 추상화를 제공한다.

대표적인 Servlet Container로는 Apache Tomcat이 있다. Tomcat은 Servlet을 실행해주는 런타임 환경이다.

Spring Boot 웹 애플리케이션을 만들면 기본적으로 내장 Tomcat이 함께 실행된다. 그래서 별도로 Tomcat을 설치하지 않아도 `SpringApplication.run(...)`만으로 웹 서버가 뜨는 것이다.

흐름을 단순화하면 다음과 같다.

```text
Client
    -> Tomcat
    -> Servlet
    -> Spring MVC
    -> Controller
```

# MVC란?

MVC는 Model, View, Controller의 약자다.

웹 애플리케이션을 한 덩어리로 만들지 않고, 역할별로 나누어 관리하기 위한 구조다.

내가 이해한 MVC의 목적은 "화면, 요청 처리, 비즈니스 데이터를 뒤섞지 않기"다.

```text
Controller
    요청을 받는다.
    어떤 일을 해야 하는지 결정한다.

Model
    처리 결과로 만들어진 데이터다.
    View나 응답으로 전달된다.

View
    Model을 사용해 사용자에게 보여줄 결과를 만든다.
```

전통적인 웹 애플리케이션에서는 Controller가 service를 호출해 Model을 만들고, View template이 그 Model을 HTML로 렌더링했다.

```text
Controller
    -> Service
    -> Model
    -> View
    -> HTML response
```

반면 REST API에서는 View template을 사용하지 않는 경우가 많다. Controller가 반환한 Java 객체를 JSON으로 변환해서 응답한다.

```text
Controller
    -> Service
    -> Java object
    -> JSON response
```

이번 글에서 다루는 `@RestController` 테스트도 이 REST API 방식에 가깝다.

# Model, View, Controller를 내 말로 정리하면

Model은 요청을 처리한 결과 데이터다.

예를 들어 consistency report API에서는 아래 객체가 Model에 해당한다고 볼 수 있다.

```java
OrderReplayConsistencyReport
```

이 객체 안에는 전체 주문 수, 일치한 주문 수, 불일치한 주문 수, 주문별 검증 결과가 들어 있다.

View는 응답을 표현하는 방식이다.

HTML 페이지를 응답할 수도 있고, PDF나 Excel 파일을 응답할 수도 있고, JSON을 응답할 수도 있다.

Spring MVC에서는 ViewResolver가 논리적인 view 이름을 실제 view로 바꿔주는 역할을 한다. 예를 들어 `"orders/list"`라는 이름을 `orders/list.html` 같은 템플릿으로 연결할 수 있다.

다만 `@RestController`에서는 보통 ViewResolver를 거쳐 HTML을 렌더링하지 않는다. 대신 `HttpMessageConverter`가 Java 객체를 JSON으로 변환한다.

Controller는 HTTP 요청을 처음 받는 진입점이다.

Controller는 비즈니스 로직을 직접 많이 들고 있으면 안 된다. 요청 parameter를 해석하고, service를 호출하고, service 결과를 응답으로 돌려주는 연결 역할에 집중하는 편이 좋다.

그래서 Controller는 두꺼운 계산기라기보다 얇은 어댑터에 가깝다.

```text
HTTP 세계
    -> Controller
    -> Java/service 세계
```

# Spring MVC의 중심, DispatcherServlet

Spring MVC에서 가장 중요한 구성 요소는 `DispatcherServlet`이다.

`DispatcherServlet`은 이름 그대로 요청을 알맞은 곳으로 dispatch, 즉 배분하는 Servlet이다.

Spring Boot 애플리케이션에서 클라이언트 요청이 들어오면 대부분의 요청은 먼저 `DispatcherServlet`으로 들어간다. 그리고 `DispatcherServlet`이 "이 URL은 어떤 Controller 메서드가 처리해야 하지?"를 찾아낸다.

단순화하면 다음 흐름이다.

```text
Client
    -> Tomcat
    -> DispatcherServlet
    -> HandlerMapping
    -> HandlerAdapter
    -> Controller
    -> return value
    -> HTTP response
```

`DispatcherServlet`은 `HttpServlet` 계열을 상속받아 Servlet으로 동작한다.

```text
DispatcherServlet
    -> FrameworkServlet
    -> HttpServletBean
    -> HttpServlet
```

요청이 들어오면 Servlet의 `service()` 흐름을 타고 내려가다가, 최종적으로 Spring MVC의 핵심 처리 메서드인 `DispatcherServlet.doDispatch()`가 호출된다.

개발자가 보통 이 메서드를 직접 호출할 일은 없다. 하지만 `MockMvc` 테스트를 이해할 때는 중요하다.

`MockMvc`는 실제 서버 포트를 열지 않지만, 내부적으로는 이 Spring MVC 요청 처리 흐름을 태운다. 그래서 Controller mapping, parameter binding, JSON 변환 같은 동작을 실제와 가깝게 검증할 수 있다.

# Spring MVC 요청 처리 순서

Spring MVC의 요청 처리 과정을 조금 더 풀면 다음과 같다.

```text
1. 요청이 DispatcherServlet에 들어온다.
2. HandlerMapping이 URL에 맞는 handler를 찾는다.
3. HandlerAdapter가 해당 handler를 실행할 방법을 찾는다.
4. HandlerAdapter가 Controller 메서드를 호출한다.
5. Controller가 ModelAndView 또는 응답 객체를 반환한다.
6. ViewResolver가 view를 찾거나, HttpMessageConverter가 응답 body를 만든다.
7. 최종 HTTP 응답이 클라이언트에게 내려간다.
```

여기서 handler는 보통 Controller 메서드라고 생각하면 된다.

예를 들어 아래 메서드는 하나의 handler가 된다.

```java
@GetMapping
public OrderReplayConsistencyReport report(
        @RequestParam(name = "inconsistentOnly", defaultValue = "false") boolean inconsistentOnly
) {
    ...
}
```

`HandlerMapping`은 요청 URL과 HTTP method를 보고 이 메서드를 찾아낸다.

`HandlerAdapter`는 이 메서드를 실제로 호출할 수 있도록 parameter를 준비한다. query parameter인 `inconsistentOnly`를 boolean 값으로 바꿔 넣어주는 일도 이 과정에 포함된다.

그 다음 반환된 `OrderReplayConsistencyReport` 객체는 JSON 응답으로 변환된다.

# Spring MVC가 확장 가능한 이유

Spring MVC는 내부 동작의 많은 부분을 인터페이스로 분리해두었다.

대표적인 인터페이스는 다음과 같다.

- `HandlerMapping`: 요청을 처리할 handler를 찾는다.
- `HandlerAdapter`: 찾은 handler를 실행한다.
- `ViewResolver`: view 이름을 실제 view로 바꾼다.
- `View`: 최종 화면을 렌더링한다.

이 구조 덕분에 Spring MVC는 `DispatcherServlet` 자체를 고치지 않고도 동작을 바꿀 수 있다.

개발자는 보통 이 인터페이스를 직접 구현하지 않아도 된다. Spring Boot가 기본 설정을 대부분 자동으로 등록해주기 때문이다.

하지만 큰 그림을 알고 있으면 테스트 코드가 덜 낯설어진다.

특히 `@WebMvcTest`는 바로 이 MVC 계층만 얇게 띄워서 테스트하는 도구다.

즉 이번 글에서 다루는 Controller 테스트는 아래 전체 흐름 중 Controller 주변만 잘라서 보는 테스트다.

```text
HTTP request
    -> DispatcherServlet
    -> HandlerMapping
    -> HandlerAdapter
    -> Controller
    -> JSON response
```

service, repository, database까지 모두 확인하는 테스트가 아니라, "Spring MVC가 요청을 Controller까지 잘 연결하고, Controller 결과를 응답으로 잘 바꾸는가"를 확인하는 테스트다.

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
