---
layout: post
title: "[토이 프로젝트] Retry + Circuit Breaker를 직접 구현해보며 이해하기"
date: 2026-04-05
categories: 토이프로젝트
---

직전 포스팅에서 Circuit Breaker의 개념을 정리했습니다.

개념적으로는 충분히 이해했지만, 코드로 직접 구현해보며 동작을 확인해보고 싶었습니다.  
팀원들에게 Circuit Breaker 도입의 필요성을 설명하면서, 실제로 어떤 방식으로 구현할 수 있는지도 함께 보여주면 좋겠다는 생각이 들었기 때문입니다.

그래서 간단한 토이 프로젝트를 구성해보았습니다.

---

## 1. 소스코드 구조

소스코드 구조는 다음과 같습니다.

```text
src/main/java/sanghoon/study/circuitbreaker/demo
 │
 ├─ DemoApplication.java
 ├─ config
 │   └─ RestClientConfig.java
 ├─ order
 │   ├─ OrderController.java
 │   └─ OrderService.java
 └─ payment
     ├─ PaymentController.java
     └─ PaymentToggleService.java
```

내부 서비스 간 호출(`OrderService` → `PaymentService`)만으로 동작을 확인할 수 있도록 설계했기 때문에,
서버를 분리하지 않고 하나의 애플리케이션 내에서 테스트가 가능합니다.

또한 `toggle API`를 통해 `PaymentService`의 상태를 의도적으로 장애 또는 정상 상태로 전환할 수 있도록 구성했습니다.

---

## 2. Retry와 Circuit Breaker

서비스 호출 과정에서 발생하는 장애는 항상 동일한 성격을 가지지는 않습니다.

일시적인 네트워크 문제나 순간적인 부하로 인해 실패하는 경우도 있고, 서비스 자체에 문제가 발생하여 지속적으로 실패하는 경우도 있습니다.

이 두 가지 상황은 서로 다른 방식으로 대응해야 합니다.

### 2.1. Retry의 역할

Retry는 일시적인 장애를 해결하기 위한 전략입니다. 짧은 시간 내에 동일한 요청을 다시 시도함으로써 일시적인 오류를 극복할 수 있습니다.

예를 들어, 네트워크 지연이나 순간적인 트래픽 증가로 인해 실패한 요청은 재시도를 통해 정상적으로 처리될 가능성이 있습니다. 하지만 Retry만으로는 모든 문제를 해결할 수 없습니다.

지속적으로 실패하는 상황에서 계속 재시도를 수행하게 되면, 불필요한 요청이 반복되면서 시스템에 추가적인 부하를 발생시킬 수 있습니다.

---

### 2.2. Circuit Breaker의 역할

Circuit Breaker는 반복되는 실패 상황에서 더 이상 호출을 시도하지 않도록 차단하는 역할을 합니다.

일정 비율 이상의 실패가 발생하면 Circuit을 OPEN 상태로 전환하고, 이 상태에서는 실제 서비스 호출을 수행하지 않습니다.

이를 통해 장애가 발생한 서비스에 대한 호출을 줄이고, 전체 시스템으로 장애가 확산되는 것을 방지할 수 있습니다.

---

### 2.3. Retry와 Circuit Breaker를 함께 사용하는 이유

Retry와 Circuit Breaker는 서로 보완적인 역할을 합니다.

Retry는 “잠깐의 문제일 수 있으니 다시 시도해보자”는 접근이고, Circuit Breaker는 “이미 장애 상태이므로 더 이상 호출하지 말자”는 접근입니다.

즉,

- Retry → 일시적인 장애 대응
- Circuit Breaker → 지속적인 장애 차단

두 가지를 함께 적용하면 불필요한 재시도를 줄이면서도, 일시적인 오류는 효과적으로 처리할 수 있습니다.

이번 토이 프로젝트에서는 이 두 가지 패턴을 함께 적용하여 각각의 역할과 동작 흐름을 직접 확인해보았습니다.

---

## 3. payment 패키지

먼저 호출 대상이 되는 `PaymentService` 관련 코드를 살펴보겠습니다.

`payment` 패키지는 실제 외부 서비스를 단순화한 형태로, 정상 응답을 반환하거나 의도적으로 장애를 발생시키는 역할을 합니다.

### 3.1. PaymentToggleService

```java
@Service
public class PaymentToggleService {

    private final AtomicBoolean enabled = new AtomicBoolean(true);

    public boolean isEnabled() {
        return enabled.get();
    }

    public boolean change(boolean enabled) {
        this.enabled.set(enabled);
        return this.enabled.get();
    }
}
```

`PaymentToggleService`는 서비스의 상태를 관리하는 역할을 합니다. 내부적으로 AtomicBoolean을 사용하여 스레드 안전하게 상태를 유지하고 있으며, API를 통해 이 값을 변경할 수 있도록 했습니다.

### 3.2. PaymentController

```java
@RestController
@RequestMapping("/payments")
public class PaymentController {

    private final PaymentToggleService paymentToggleService;

    public PaymentController(PaymentToggleService paymentToggleService) {
        this.paymentToggleService = paymentToggleService;
    }

    @PostMapping("/toggle")
    public Map<String, Object> toggle(@RequestParam boolean enabled) {
        boolean current = paymentToggleService.change(enabled);
        return Map.of(
                "paymentEnabled", current,
                "changedAt", LocalDateTime.now().toString()
        );
    }

    @GetMapping("/status")
    public Map<String, Object> status() {
        return Map.of(
                "paymentEnabled", paymentToggleService.isEnabled()
        );
    }

    @PostMapping("/process/{orderId}")
    public Map<String, Object> process(@PathVariable String orderId) {
        if (!paymentToggleService.isEnabled()) {
            throw new ResponseStatusException(
                    HttpStatus.INTERNAL_SERVER_ERROR,
                    "Payment service is unavailable"
            );
        }

        return Map.of(
                "success", true,
                "orderId", orderId,
                "message", "Payment completed",
                "processedAt", LocalDateTime.now().toString()
        );
    }
}
```
`PaymentController`의 주요 역할은 다음과 같습니다.

* toggle API를 통해 서비스 상태를 변경
* status API를 통한 서비스 상태를 조회
* process API를 통해 결제 요청 처리

특히 process API에서는 서비스가 비활성화된 상태일 경우 예외를 발생시켜 의도적으로 실패 상황을 만들 수 있도록 했습니다. 의도적으로 만든 실패를 통해 Retry와 Circuit Breaker의 동작을 명확하게 확인할 수 있습니다.

이를 통해 `OrderService`에서 Retry와 Circuit Breaker가 어떻게 동작하는지를 쉽게 확인할 수 있습니다.

---

## 4. order 패키지

다음으로 `PaymentService`를 호출하는 order 패키지를 살펴보겠습니다.

이 패키지에서는 `Retry`와 `Circuit Breaker`를 실제로 적용하여 호출 실패 시 어떤 방식으로 동작하는지를 확인할 수 있도록 구성했습니다.

---

### 4.1. OrderService

```java
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final RestClient restClient;

    public OrderService(RestClient restClient) {
        this.restClient = restClient;
    }

    @Retry(name = "paymentService")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public Map<String, Object> placeOrder(String orderId) {
        log.info("OrderService.placeOrder - calling payment service. orderId={}", orderId);

        Map<String, Object> response = restClient.post()
                .uri("/payments/process/{orderId}", orderId)
                .retrieve()
                .body(new org.springframework.core.ParameterizedTypeReference<Map<String, Object>>() {});

        log.info("OrderService.placeOrder - payment success. orderId={}", orderId);
        return response;
    }

    public Map<String, Object> paymentFallback(String orderId, Throwable t) {
        log.warn("OrderService.paymentFallback - fallback executed. orderId={}, reason={}",
                orderId, t.toString());

        return Map.of(
                "success", false,
                "orderId", orderId,
                "message", "fallback: payment service temporarily unavailable",
                "reason", t.getClass().getSimpleName()
        );
    }
}
```
`OrderService`는 `PaymentService`를 호출하는 역할을 담당합니다.

여기서 핵심은 다음 두 가지입니다.

```java
@Retry(name = "paymentService")
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
```

* Retry → 호출 실패 시 재시도 수행
* Circuit Breaker → 실패가 누적되면 호출 차단

fallback은 Circuit Breaker에서 처리되며, Retry는 예외 발생 시 재시도를 수행합니다.

하나의 요청은 다음과 같은 순서로 처리됩니다.

```text
OrderService 호출
  ↓
Retry (최대 N회 재시도)
  ↓
실패 누적
  ↓
Circuit Breaker OPEN
  ↓
이후 요청은 즉시 fallback 처리
```

즉, Retry는 일시적인 오류를 보완하고, Circuit Breaker는 지속적인 장애 상황에서 호출을 차단하는 역할을 합니다.

Circuit Breaker가 OPEN 상태가 되면, `PaymentService` 호출 없이 바로 fallback이 실행됩니다.

이 상태에서는 실제 서비스 호출 자체가 차단되기 때문에, 불필요한 부하를 줄이고 장애 확산을 방지할 수 있습니다.

## 4.2. OrderController

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/orders/{orderId}/pay")
    public Map<String, Object> pay(@PathVariable String orderId) {
        return orderService.placeOrder(orderId);
    }
}
```

`OrderController`는 테스트를 위한 진입점 역할을 합니다.

클라이언트는 이 API를 호출함으로써 내부적으로 `PaymentService` 호출이 발생하고, 그 과정에서 Retry와 Circuit Breaker의 동작을 확인할 수 있습니다.

---

## 5. config 패키지

`RestClientConfig`는 RestClient Bean을 생성하는 설정 클래스입니다.

이 설정을 통해 `OrderService`에서 `PaymentService`로의 호출 경로가 구성됩니다.

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
                .baseUrl("http://localhost:8080")
                .build();
    }
}
```

현재 프로젝트에서는 별도의 서버를 분리하지 않고 하나의 애플리케이션 내에서 테스트하기 때문에, baseUrl을 동일한 애플리케이션 주소(localhost)로 설정하여 내부 호출이 가능하도록 구성했습니다.

실무에서는 외부 서비스 주소를 사용하며, 환경별 설정으로 분리하는 것이 일반적입니다.

---

## 6. 설정

앞에서 살펴본 코드에서 `Retry`와 `Circuit Breaker`는 어노테이션만으로 적용되어 있는 것처럼 보이지만, 실제 동작 방식은 설정을 통해 결정됩니다.

이 프로젝트에서는 `application.yml`을 통해 Retry와 Circuit Breaker의 동작 기준을 정의했습니다.

### 6.1. Retry 설정

```yaml
resilience4j:
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
```

* maxAttempts: 최대 재시도 횟수
* waitDuration: 재시도 간 대기 시간

즉, 하나의 요청에 대해 최대 3번까지 재시도를 수행합니다.

### 6.2. Circuit Breaker 설정

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 4
        minimumNumberOfCalls: 4
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

* slidingWindowSize: 실패율을 계산하는 기준 요청 수
* minimumNumberOfCalls: 최소 호출 수
* failureRateThreshold: OPEN 상태로 전환되는 실패율 기준
* waitDurationInOpenState: OPEN 상태 유지 시간

예를 들어, 최근 4건 중 2건 이상 실패하면 (50%) Circuit Breaker가 OPEN 상태로 전환됩니다.

### 6.3. 설정이 의미하는 동작

이 설정을 기반으로 전체 흐름은 다음과 같이 동작합니다.

```text
요청 발생
 → Retry (최대 3번 재시도)
 → 실패 누적
 → 실패율 초과 시 Circuit Breaker OPEN
 → 이후 요청은 fallback 처리
 → 일정 시간 후 HALF_OPEN
 → 성공 시 CLOSED 복구
```

설정을 통해 Retry와 Circuit Breaker의 동작을 세밀하게 조정할 수 있으며, 실무에서는 서비스 특성에 맞게 이 값을 튜닝하는 것이 중요합니다.

---

## 7. 실행 및 동작 확인

이제 실제로 애플리케이션을 실행하고, Retry와 Circuit Breaker가 어떻게 동작하는지 확인해보겠습니다.

### 7.1. 애플리케이션 실행

```bash
./gradlew bootRun
```

### 7.2. PaymentService 상태 확인

```bash
curl http://localhost:8080/payments/status
```

```json
{
  "paymentEnabled": true
}
```

### 7.3. 정상 상태에서 호출

```bash
curl -X POST http://localhost:8080/orders/1001/pay
```

```json
{
  "success": true,
  "orderId": "1001",
  "message": "Payment completed",
  "processedAt":"2026-04-05T19:19:46.857306400"
}
```
정상 상태에서는 `PaymentService`가 정상적으로 호출됩니다.

### 7.4. 장애 상태로 전환

```bash
curl -X POST http://localhost:8080/payments/toggle?enabled=false
```

```json
{
  "paymentEnabled": false,
  "changedAt":"2026-04-05T19:24:05.503712700"
}
```

이제 PaymentService는 항상 실패하도록 설정됩니다.

### 7.5. Retry 동작 확인

```bash
curl -X POST http://localhost:8080/orders/2001/pay
```

```json
{
    "success":false,
    "reason":"InternalServerError",
    "message":"fallback: payment service temporarily unavailable",
    "orderId":"2001"
}
```

애플리케이션 로그를 통해 동일한 요청이 Retry 설정에 따라 3회 재시도되는 것을 확인할 수 있습니다.

```text
calling payment service. orderId=2001
calling payment service. orderId=2001
calling payment service. orderId=2001
fallback executed. orderId=2001, reason=org.springframework.web.client.HttpServerErrorException$InternalServerError: 500 Internal Server Error: "{"timestamp":"2026-04-05T10:41:06.338+00:00","status":500,"error":"Internal Server Error","path":"/payments/process/2001"}"
```

### 7.6. Circuit Breaker OPEN 상태 확인

다음과 같이 여러 번 실패 요청을 반복합니다.

```bash
curl -X POST http://localhost:8080/orders/2002/pay
curl -X POST http://localhost:8080/orders/2003/pay
curl -X POST http://localhost:8080/orders/2004/pay
curl -X POST http://localhost:8080/orders/2005/pay
```

일정 횟수 이상의 실패가 누적되면 Circuit Breaker가 OPEN 상태로 전환됩니다.

이후 로그는 다음과 같이 변경됩니다.

```text
fallback executed. orderId=2005, reason=io.github.resilience4j.circuitbreaker.CallNotPermittedException: CircuitBreaker 'paymentService' is OPEN and does not permit further calls
```

CallNotPermittedException이 발생하는 경우, Circuit Breaker가 OPEN 상태로 전환되어 실제 호출이 차단된 상태입니다.

> CallNotPermittedException 발생 → 호출 차단

이 상태에서는 더 이상 `PaymentService`를 호출하지 않고, 즉시 fallback 로직이 실행됩니다.

### 7.7. HALF_OPEN → CLOSED 복구 확인

다시 `PaymentService`를 정상 상태로 변경합니다.

```bash
curl -X POST http://localhost:8080/payments/toggle?enabled=true
```

설정된 시간(waitDurationInOpenState)이 지난 후 요청을 수행합니다.

```bash
curl -X POST http://localhost:8080/orders/3001/pay
```

이 시점에는 Circuit Breaker가 HALF_OPEN 상태가 되어 일부 요청만 허용됩니다.

요청이 성공하면 Circuit Breaker는 CLOSED 상태로 복구됩니다.

```json
{
  "success": true,
  "orderId": "3001",
  "message": "Payment completed",
  "processedAt":"2026-04-05T19:55:54.306016500"
}
```

### 7.8. 전체 동작 흐름 정리

```text
정상 상태
  ↓
장애 발생
  ↓
Retry 수행 (재시도)
  ↓
실패 누적
  ↓
Circuit Breaker OPEN
  ↓
호출 차단 (fallback)
  ↓
일정 시간 후 HALF_OPEN
  ↓
정상 응답 시 CLOSED 복구
```

### 7.9. 정리

테스트를 통해 다음과 같은 점을 확인할 수 있습니다.

* Retry는 일시적인 오류를 보완하기 위해 재시도를 수행한다
* Circuit Breaker는 반복되는 실패 상황에서 호출을 차단한다
* OPEN 상태에서는 실제 서비스 호출이 발생하지 않는다
* 일정 시간이 지나면 HALF_OPEN 상태를 거쳐 정상 상태로 복구된다

이처럼 Retry와 Circuit Breaker를 함께 적용하면 일시적인 오류는 유연하게 처리하면서도, 지속적인 장애로부터 시스템을 보호할 수 있습니다.

---

## 8. 트러블 슈팅: Retry가 동작하지 않았던 이유

Resilience4j의 `@Retry`, `@CircuitBreaker` 어노테이션은 Spring AOP 기반으로 동작합니다. 따라서 프로젝트에 다음 의존성이 포함되어 있어야 실제로 Retry와 Circuit Breaker가 적용됩니다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

AOP 의존성이 없으면 @Retry, @CircuitBreaker 어노테이션이 메서드에 선언되어 있어도 RestClient 호출에서 발생한 예외가 그대로 메서드 외부로 전달됩니다. 그래서 Retry와 Circuit Breaker가 동작하지 않습니다. 

---

## 참고: 소스 코드 

* [Circuit Breaker Demo](https://github.com/sanghoon-lee/circuit-breaker)

---

## 관련 포스팅

* [[학습기록] 서킷 브레이커 패턴(Circuit Breaker Pattern): 개념정리](https://sanghoon-lee.github.io/2026/04/01/circuitbreaker/)

---

#CircuitBreaker #Retry #Resilience4j #SpringBoot #MSA #장애대응 #토이프로젝트


