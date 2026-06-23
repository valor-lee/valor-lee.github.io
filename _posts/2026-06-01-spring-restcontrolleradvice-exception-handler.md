---
title: '[Spring] @RestControllerAdvice는 여러 ExceptionHandler 중 무엇을 선택할까'
date: 2026-06-01 00:58:00 +09:00
categories: [Framework, Spring]
tags:
  [
    backend,
    Java,
    Spring,
    Spring Boot,
    exception handling,
    RestControllerAdvice,
    ControllerAdvice
  ]
---

# 개요

`multi-asset-oms-simulation-platform` 프로젝트에서 `audit-replay` 모듈에 API 예외 처리기를 추가했다.

```java
@RestControllerAdvice
public class AuditReplayExceptionHandler {

    @ExceptionHandler({
            OrderReplayException.class,
            IllegalArgumentException.class
    })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handleDomainException(RuntimeException exception) {
        return new AuditReplayErrorResponse(exception.getMessage());
    }
}
```

이 코드를 보다가 자연스럽게 이런 의문이 생겼다.

> 다른 모듈에도 `@RestControllerAdvice`가 있고, 거기서 같은 예외를 잡고 있으면 Spring은 무엇을 선택할까?

이번 글에서는 Spring MVC에서 Controller 예외가 발생했을 때 `@RestControllerAdvice`와 `@ExceptionHandler`가 어떻게 선택되는지 정리한다.

# @RestControllerAdvice는 공통 try-catch 역할을 한다

Controller 안에서 매번 아래처럼 try-catch를 작성하면 코드가 금방 지저분해진다.

```java
@GetMapping("/{orderId}")
public OrderReplayConsistencyResult check(@PathVariable UUID orderId) {
    try {
        return consistencyService.check(orderId);
    } catch (OrderReplayException exception) {
        throw exception;
    }
}
```

Spring은 이런 반복을 줄이기 위해 `@ControllerAdvice`, `@RestControllerAdvice`를 제공한다.

`@RestControllerAdvice`는 쉽게 말하면 여러 Controller 주변에 붙는 공통 예외 처리기다.

```java
@RestControllerAdvice
public class AuditReplayExceptionHandler {

    @ExceptionHandler(OrderReplayException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handle(OrderReplayException exception) {
        return new AuditReplayErrorResponse(exception.getMessage());
    }
}
```

Controller나 request binding 과정에서 `OrderReplayException`이 발생하면 Spring MVC가 이 handler를 찾아 실행한다.

응답은 JSON으로 내려간다.

```json
{
  "message": "order not found"
}
```

# 예외 처리 흐름

예를 들어 아래 API가 있다고 하자.

```java
@RestController
@RequestMapping("/api/audit-replay/order-replay/consistency")
public class OrderReplayConsistencyController {

    private final OrderReplayConsistencyService consistencyService;

    public OrderReplayConsistencyController(OrderReplayConsistencyService consistencyService) {
        this.consistencyService = consistencyService;
    }

    @GetMapping("/{orderId}")
    public OrderReplayConsistencyResult check(@PathVariable("orderId") UUID orderId) {
        return consistencyService.check(orderId);
    }
}
```

요청이 들어오면 흐름은 대략 아래와 같다.

```text
1. HTTP 요청이 들어온다.
2. Spring MVC가 URL에 맞는 Controller method를 찾는다.
3. path variable, query parameter 등을 method argument로 변환한다.
4. Controller method를 실행한다.
5. Controller 또는 service에서 예외가 발생한다.
6. Spring MVC가 등록된 @RestControllerAdvice 중 처리 가능한 handler를 찾는다.
7. 선택된 @ExceptionHandler method를 실행한다.
8. handler가 반환한 객체를 JSON으로 변환해 응답한다.
```

핵심은 Controller에 직접 try-catch가 없어도 Spring MVC가 예외 처리기를 찾아준다는 점이다.

# 여러 ExceptionHandler가 있으면 어떤 것을 고를까

Spring은 예외가 발생하면 처리 가능한 `@ExceptionHandler` 후보를 찾는다.

선택 기준은 크게 세 가지로 볼 수 있다.

1. 예외 타입이 더 구체적인 handler가 우선이다.
2. 여러 advice가 같은 수준으로 처리할 수 있으면 `@Order` 우선순위를 본다.
3. 그래도 애매하면 등록 순서에 영향을 받을 수 있으므로 위험하다.

예를 들어 아래 두 handler가 있다고 하자.

```java
@ExceptionHandler(RuntimeException.class)
public ErrorResponse handleRuntime(RuntimeException exception) {
    return new ErrorResponse("runtime error");
}
```

```java
@ExceptionHandler(OrderReplayException.class)
public AuditReplayErrorResponse handleOrderReplay(OrderReplayException exception) {
    return new AuditReplayErrorResponse(exception.getMessage());
}
```

`OrderReplayException`이 `RuntimeException`을 상속한다면 두 handler 모두 처리할 수 있다.

하지만 이 경우에는 `OrderReplayException`을 직접 지정한 handler가 더 구체적이므로 우선 선택된다.

```text
RuntimeException handler
    넓은 범위

OrderReplayException handler
    더 좁고 구체적인 범위
```

Spring은 더 구체적인 예외 타입을 선호한다.

# 문제가 되는 경우

문제는 서로 다른 모듈에서 같은 예외를 같은 수준으로 잡는 경우다.

예를 들어 audit-replay 모듈에 이런 handler가 있다.

```java
@RestControllerAdvice
public class AuditReplayExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handle(IllegalArgumentException exception) {
        return new AuditReplayErrorResponse(exception.getMessage());
    }
}
```

그런데 intent-generation 모듈에도 이런 handler가 있다고 하자.

```java
@RestControllerAdvice
public class IntentGenerationExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public IntentErrorResponse handle(IllegalArgumentException exception) {
        return new IntentErrorResponse(exception.getMessage());
    }
}
```

이제 어떤 Controller에서 `IllegalArgumentException`이 발생하면 둘 다 처리 후보가 될 수 있다.

```text
AuditReplayExceptionHandler
    IllegalArgumentException 처리 가능

IntentGenerationExceptionHandler
    IllegalArgumentException 처리 가능
```

이런 구조는 유지보수에 좋지 않다.

왜냐하면 모듈 A의 예외가 모듈 B의 handler에서 처리될 가능성이 생기기 때문이다.

특히 `IllegalArgumentException`, `RuntimeException`, `Exception`처럼 넓은 예외 타입을 전역 advice에서 잡으면 충돌 가능성이 커진다.

# 해결 방법 1. 더 구체적인 예외를 사용한다

가장 좋은 방법 중 하나는 도메인 전용 예외를 사용하는 것이다.

예를 들어 audit-replay에서는 이미 `OrderReplayException`이 있다.

```java
public class OrderReplayException extends RuntimeException {

    public OrderReplayException(String message) {
        super(message);
    }
}
```

그러면 예외 처리기도 더 명확해진다.

```java
@ExceptionHandler(OrderReplayException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public AuditReplayErrorResponse handleOrderReplayException(OrderReplayException exception) {
    return new AuditReplayErrorResponse(exception.getMessage());
}
```

이렇게 하면 다른 모듈의 일반적인 `IllegalArgumentException`과 충돌할 가능성이 줄어든다.

도메인 전용 예외는 에러의 의미를 코드에 남기는 효과도 있다.

```text
IllegalArgumentException
    어떤 입력이 잘못됐다는 일반적인 의미

OrderReplayException
    order replay 흐름에서 발생한 도메인 오류
```

# 해결 방법 2. Advice 적용 범위를 제한한다

모듈별 handler를 유지하고 싶다면 `@RestControllerAdvice`의 적용 범위를 제한하는 것이 좋다.

기본 형태는 모든 Controller에 적용될 수 있다.

```java
@RestControllerAdvice
public class AuditReplayExceptionHandler {
}
```

이것을 audit-replay 패키지로 제한할 수 있다.

```java
@RestControllerAdvice(basePackages = "com.multiassetoms.auditreplay")
public class AuditReplayExceptionHandler {
}
```

이렇게 하면 이 handler는 `com.multiassetoms.auditreplay` 아래 Controller에서 발생한 예외만 처리한다.

즉 다른 모듈의 Controller에서 `IllegalArgumentException`이 발생해도 audit-replay handler가 가로챌 가능성이 줄어든다.

이번 프로젝트에서도 이 방식으로 수정했다.

```java
@RestControllerAdvice(basePackages = "com.multiassetoms.auditreplay")
public class AuditReplayExceptionHandler {

    @ExceptionHandler({
            OrderReplayException.class,
            IllegalArgumentException.class
    })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handleDomainException(RuntimeException exception) {
        return new AuditReplayErrorResponse(exception.getMessage());
    }
}
```

`IllegalArgumentException`처럼 넓은 예외를 잡더라도, audit-replay 모듈 범위 안에서만 처리하게 된다.

# 해결 방법 3. @Order로 우선순위를 명시한다

여러 advice가 공통 예외를 처리해야 하는 경우 `@Order`를 사용할 수 있다.

```java
@Order(1)
@RestControllerAdvice
public class HighPriorityExceptionHandler {
}
```

```java
@Order(2)
@RestControllerAdvice
public class LowPriorityExceptionHandler {
}
```

숫자가 낮을수록 우선순위가 높다.

하지만 `@Order`는 충돌을 해결하는 최후의 수단에 가깝다.

가능하면 먼저 아래 두 가지를 고려하는 편이 좋다.

- 예외 타입을 더 구체적으로 만든다.
- advice 적용 범위를 모듈이나 controller 패키지로 제한한다.

`@Order`만으로 해결하면 나중에 advice가 늘어났을 때 전체 우선순위를 계속 기억해야 한다.

# basePackages와 assignableTypes

`@RestControllerAdvice`는 여러 방식으로 적용 대상을 제한할 수 있다.

패키지 기준 제한:

```java
@RestControllerAdvice(basePackages = "com.multiassetoms.auditreplay")
public class AuditReplayExceptionHandler {
}
```

특정 Controller 타입 기준 제한:

```java
@RestControllerAdvice(assignableTypes = {
        OrderExecutionReplayController.class,
        OrderReplayConsistencyController.class,
        OrderAuditTrailController.class,
        OrderReplayConsistencyReportController.class
})
public class AuditReplayExceptionHandler {
}
```

패키지 기준은 모듈 단위로 관리하기 좋다.

특정 Controller 기준은 적용 대상을 아주 명확히 제한할 수 있지만, Controller가 추가될 때마다 목록을 수정해야 한다.

현재 프로젝트에서는 audit-replay 모듈 전체에 같은 오류 응답 정책을 적용하는 것이 자연스러우므로 `basePackages`가 더 적합하다.

# 테스트에서는 왜 직접 등록했을까

운영 실행에서는 `@SpringBootApplication`의 component scan이 `@RestControllerAdvice`를 자동으로 찾는다.

하지만 `@WebMvcTest`는 전체 Spring Context를 띄우지 않고 MVC slice만 띄운다.

그래서 테스트에서는 필요한 Controller와 advice를 명시적으로 넣었다.

```java
@WebMvcTest(OrderExecutionReplayController.class)
@ContextConfiguration(classes = {
        OrderExecutionReplayControllerTest.TestApplication.class,
        OrderExecutionReplayController.class,
        AuditReplayExceptionHandler.class
})
class OrderExecutionReplayControllerTest {
}
```

이렇게 해야 테스트에서도 예외 handler가 동작한다.

예를 들어 `orderQuantity` query parameter가 빠진 요청을 보내면:

```java
mockMvc.perform(get("/api/audit-replay/order-replay/{orderId}", orderId))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.message").value("orderQuantity is required"));
```

`AuditReplayExceptionHandler`가 등록되어 있기 때문에 JSON 오류 응답을 검증할 수 있다.

# compileOnly servlet API는 왜 필요했나

이번 예외 처리기에서는 Spring MVC의 servlet 기반 예외 타입을 사용했다.

```java
@ExceptionHandler(MissingServletRequestParameterException.class)
public AuditReplayErrorResponse handleMissingRequestParameter(
        MissingServletRequestParameterException exception
) {
    return new AuditReplayErrorResponse(exception.getParameterName() + " is required");
}
```

`MissingServletRequestParameterException`은 내부적으로 servlet API 타입을 참조한다.

그래서 `audit-replay` 모듈을 컴파일할 때 servlet API를 알 수 있어야 한다.

build file에는 아래 의존성을 추가했다.

```kotlin
compileOnly("jakarta.servlet:jakarta.servlet-api") // MVC exception handler uses servlet-based request exceptions
```

`compileOnly`의 의미는 다음과 같다.

```text
컴파일할 때만 필요하다.
실행 jar에는 포함하지 않는다.
실제 실행 시에는 spring-boot-starter-web이 제공한다.
```

`audit-replay`는 라이브러리 모듈이고, 실제 실행은 `oms-core`가 담당한다.

`oms-core`에는 이미 web starter가 있다.

```kotlin
implementation("org.springframework.boot:spring-boot-starter-web")
```

따라서 `audit-replay`는 컴파일할 때만 servlet API를 알면 된다.

# 이번 프로젝트에서 정리된 형태

최종적으로 audit-replay 예외 처리기는 아래 성격을 가진다.

```text
적용 범위:
    com.multiassetoms.auditreplay 패키지

처리 대상:
    OrderReplayException
    IllegalArgumentException
    MissingServletRequestParameterException
    MethodArgumentTypeMismatchException
    MethodArgumentNotValidException

응답 형태:
    HTTP 400 Bad Request
    { "message": "..." }
```

코드로 보면 다음과 같다.

```java
@RestControllerAdvice(basePackages = "com.multiassetoms.auditreplay")
public class AuditReplayExceptionHandler {

    @ExceptionHandler({
            OrderReplayException.class,
            IllegalArgumentException.class
    })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handleDomainException(RuntimeException exception) {
        return new AuditReplayErrorResponse(exception.getMessage());
    }

    @ExceptionHandler(MissingServletRequestParameterException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handleMissingRequestParameter(
            MissingServletRequestParameterException exception
    ) {
        return new AuditReplayErrorResponse(exception.getParameterName() + " is required");
    }

    @ExceptionHandler({
            MethodArgumentTypeMismatchException.class,
            MethodArgumentNotValidException.class
    })
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public AuditReplayErrorResponse handleInvalidRequestArgument(Exception exception) {
        return new AuditReplayErrorResponse("invalid request argument");
    }
}
```

# 마무리

`@RestControllerAdvice`는 매우 편리하지만, 아무 범위 제한 없이 넓은 예외를 잡으면 다른 모듈의 예외 처리와 충돌할 수 있다.

특히 멀티모듈 프로젝트에서는 아래 원칙을 지키는 것이 좋다.

- 가능하면 도메인 전용 예외를 사용한다.
- `RuntimeException`, `IllegalArgumentException`, `Exception`처럼 넓은 예외는 조심해서 잡는다.
- 모듈별 advice는 `basePackages`로 적용 범위를 제한한다.
- 여러 advice가 같은 예외를 잡아야 한다면 `@Order`로 우선순위를 명시한다.
- MVC slice test에서는 필요한 advice를 테스트 context에 직접 등록한다.

이번 audit-replay 작업에서는 `basePackages`로 적용 범위를 제한해 다른 모듈과의 예외 처리 충돌 가능성을 줄였다.

작은 설정처럼 보이지만, 모듈이 늘어날수록 이런 경계 설정이 유지보수를 꽤 편하게 만든다.
