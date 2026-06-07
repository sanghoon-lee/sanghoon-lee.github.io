---
layout: post
title: "[토이 프로젝트] Retry + Circuit Breaker를 직접 구현해보며 이해하기"
date: 2026-04-05
description: Retry + Circuit Breaker를 직접 구현해 본 토이 프로젝트 수행에 대한 포스팅입니다.
image: /assets/images/repair.jpg
categories: 토이프로젝트
---

Circuit Breaker 패턴에 대해 공부하면서 개념과 동작 원리는 어느 정도 이해할 수 있었습니다.

일시적인 장애는 Retry로 극복하고, 지속적인 장애는 Circuit Breaker로 차단하여 시스템의 안정성과 회복력을 높일 수 있다는 점도 알게 되었습니다.

하지만 실제 장애 상황에서 두 패턴이 어떻게 동작하는지 직접 살펴보고 싶었습니다.

그래서 간단한 토이 프로젝트를 만들어 장애 상황을 재현해보면서 Retry와 Circuit Breaker의 동작을 단계별로 확인했습니다.

---

## 1. 프로젝트 구조

프로젝트 구조는 다음과 같습니다.

```nohighlight
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

* `order` 패키지: 주문 요청을 처리하며 내부적으로 `payment` 패키지를 호출
* `payment` 패키지: 결제 요청을 처리하고 장애 상황을 제어하는 기능 제공

서버를 분리하지 않더라도 하나의 애플리케이션 내에서 테스트가 가능하도록 `order` 패키지에서 `payment` 패키지를 호출하는 구조로 구성했습니다.

---

### 1.1. payment 패키지

`payment` 패키지는 주문 서비스로부터 호출되는 가상의 결제 서비스를 구현한 영역으로 다음의 API를 제공합니다.

* `/payments/toggle`: 서비스 상태(정상/장애)를 변경
* `/payments/status`: 현재 서비스 상태(정상/장애)를 조회
* `/payments/process/{orderId}`: 주문에 대한 결제 처리

특히 `/payments/process/{orderId}` API는 서비스 상태가 장애인 경우 의도적으로 예외를 발생시킵니다. 이를 통해 장애 상황을 손쉽게 재현하며 Retry와 Circuit Breaker의 동작을 확인할 수 있습니다.

`payment` 패키지의 `PaymentController` 클래스는 서비스 상태 변경과 결제 처리 API를 제공하며, `PaymentToggleService` 클래스는 현재 서비스 상태를 메모리에 보관하는 역할을 담당합니다.

서비스 상태는 `AtomicBoolean`을 사용하여 메모리에 보관했습니다.

**PaymentController.java**

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

**PaymentToggleService.java**

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

---

### 1.2. order 패키지

`order` 패키지는 가상의 주문 서비스를 구현한 영역으로 다음의 API를 제공합니다.

* `/orders/{orderId}/pay`: 주문에 대한 결제 요청

`order` 패키지의 `OrderController` 클래스는 주문에 대한 결제 요청 API를 제공하며, `OrderService` 클래스는 요청을 처리하면서 내부적으로 `payment` 패키지에 구현된 결제 서비스를 호출합니다. 또한 장애 상황에 대응할 수 있도록 Retry와 Circuit Breaker가 적용되어 있습니다.

**OrderController.java**

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

**OrderService.java**

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


`OrderService` 클래스에서 핵심적인 부분은 아래의 코드입니다.

```java
@Retry(name = "paymentService")
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
```

* Retry → 호출 실패 시 재시도 수행
* Circuit Breaker → 실패 이력을 기반으로 호출 차단 여부를 결정

Retry는 일시적인 장애를 극복하기 위해 재시도를 수행하며, Circuit Breaker는 누적된 실패율을 기준으로 상태를 변경합니다.

```text
OrderService 호출
  ↓
PaymentService 호출 실패
  ↓
Retry 수행
  ↓
Retry 실패
  ↓
Circuit Breaker에 실패 이력 기록
  ↓
실패가 누적되면 OPEN 상태 전환
  ↓
이후 요청은 즉시 fallback 처리
```

---

## 2. 설정

앞에서 살펴본 코드에서 Retry와 Circuit Breaker는 어노테이션만으로 적용되어 있는 것처럼 보이지만, 실제 동작 방식은 설정을 통해 결정됩니다.

설정은 `application.yml`에 정의됩니다. 설정을 통해 Retry와 Circuit Breaker의 동작을 세밀하게 조정할 수 있으며, 실무에서는 서비스 특성에 맞게 이 값을 튜닝하는 것이 중요합니다.

이번 토이프로젝트에서는 장애 상황을 빠르게 재현하기 위해 비교적 작은 임계값을 사용했습니다. 

---

### 2.1. Retry

```yaml
resilience4j:
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
```

* maxAttempts: 최초 호출을 포함한 최대 시도 횟수
* waitDuration: 재시도 간 대기 시간

---

### 2.2. Circuit Breaker

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

> 현재 설정에서는 최소 4건 이상의 호출이 발생한 이후부터 실패율을 계산합니다. 따라서 최근 4건의 요청 중 2건 이상이 실패하면 실패율이 50%가 되어 Circuit Breaker가 OPEN 상태로 전환됩니다.

---

## 3. 실행

이제 실제로 애플리케이션을 실행하고, Retry와 Circuit Breaker가 어떻게 동작하는지 확인해보겠습니다.

```bash
./gradlew bootRun
```

---

### 3.1. 정상 상태에서 호출

가상의 결제 서비스(`payment` 패키지)가 상태를 조회하면 정상인 것을 확인할 수 있습니다.

**요청**
```bash
$ curl http://localhost:8080/payments/status
```

**응답**
```json
{
  "paymentEnabled": true
}
```

이 상태에서 가상의 주문 서비스(`order` 패키지)가 결제 서비스를 호출하면 정상적인 응답을 받을 수 있습니다.

**요청**
```bash
$ curl -X POST http://localhost:8080/orders/1001/pay
```

**응답**
```json
{
  "success": true,
  "orderId": "1001",
  "message": "Payment completed",
  "processedAt":"2026-04-05T19:19:46.857306400"
}
```

---

### 3.2. Retry 수행

이번에는 Retry 수행을 확인하기 위해서 결제 서비스의 상태를 장애로 전환합니다.

**요청**
```bash
$ curl -X POST http://localhost:8080/payments/toggle?enabled=false
```

**응답**
```json
{
  "paymentEnabled": false,
  "changedAt":"2026-04-05T19:24:05.503712700"
}
```

이 상태에서 주문 서비스가 결제 서비스를 호출하면 오류로 처리된 응답을 받게 됩니다. 

**요청**
```bash
$ curl -X POST http://localhost:8080/orders/2001/pay
```

**응답**
```json
{
    "success":false,
    "reason":"InternalServerError",
    "message":"fallback: payment service temporarily unavailable",
    "orderId":"2001"
}
```

애플리케이션 로그를 통해 동일한 요청이 총 3회 시도된 것을 확인할 수 있습니다.

**로그**
```text
calling payment service. orderId=2001
calling payment service. orderId=2001
calling payment service. orderId=2001
fallback executed. orderId=2001, reason=org.springframework.web.client.HttpServerErrorException$InternalServerError: 500 Internal Server Error: "{"timestamp":"2026-04-05T10:41:06.338+00:00","status":500,"error":"Internal Server Error","path":"/payments/process/2001"}"
```

---

### 3.3. Circuit Breaker 차단

이후 다음과 같이 실패 요청을 여러 차례 반복하게 되면 Circuit Breaker가 OPEN 상태로 전환됩니다.

**요청**
```bash
curl -X POST http://localhost:8080/orders/2002/pay
curl -X POST http://localhost:8080/orders/2003/pay
curl -X POST http://localhost:8080/orders/2004/pay
curl -X POST http://localhost:8080/orders/2005/pay
```

이후 애플리케이션 로그에서 다음과 같은 메시지를 확인할 수 있습니다.

**로그**
```text
fallback executed. orderId=2005, reason=io.github.resilience4j.circuitbreaker.CallNotPermittedException: CircuitBreaker 'paymentService' is OPEN and does not permit further calls
```

CallNotPermittedException이 발생한 것으로 Circuit Breaker가 OPEN 상태로 전환되어 실제 호출이 차단된 상태를 의미합니다.

이 상태에서는 더 이상 즉시 fallback 로직이 실행됩니다.

---

### 3.4. Circuit Breaker 복구

이번에는 Circuit Breaker 차단이 복구되는지 확인하기 위해서 결제 서비스의 상태를 정상으로 전환합니다. 

**요청**
```bash
curl -X POST http://localhost:8080/payments/toggle?enabled=true
```

설정된 시간(waitDurationInOpenState)이 지난 후 요청을 수행하면 Circuit Breaker는 HALF_OPEN 상태가 되어 서비스 정상 여부를 확인하기 위한 호출을 허용합니다.

**요청**
```bash
curl -X POST http://localhost:8080/orders/3001/pay
```

**응답**
```json
{
  "success": true,
  "orderId": "3001",
  "message": "Payment completed",
  "processedAt":"2026-04-05T19:55:54.306016500"
}
```

그리고 정상 응답의 반환이 확인되면 Circuit Breaker는 CLOSED 상태로 복구됩니다.

---

## 4. 트러블 슈팅: Retry와 Circuit Breaker가 동작하지 않았던 이유

Resilience4j의 **@Retry**, **@CircuitBreaker** 어노테이션은 Spring AOP 기반으로 동작합니다. 따라서 프로젝트에 다음 의존성이 포함되어 있어야 실제로 Retry와 Circuit Breaker가 적용됩니다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

AOP 의존성이 없으면 **@Retry**, **@CircuitBreaker** 어노테이션이 메서드에 선언되어 있어도 RestClient 호출에서 발생한 예외가 그대로 메서드 외부로 전달됩니다. 그래서 Retry와 Circuit Breaker가 동작하지 않습니다. 

---

## 5. 정리

이번 토이프로젝트를 통해 확인한 내용은 다음과 같습니다.

* Retry는 일시적인 오류를 보완하기 위해 재시도를 수행한다
* Circuit Breaker는 반복되는 실패 상황에서 호출을 차단한다
* OPEN 상태에서는 실제 서비스 호출이 발생하지 않는다
* 일정 시간이 지나면 HALF_OPEN 상태를 거쳐 정상 상태로 복구된다
* Retry와 Circuit Breaker를 함께 사용하면 일시적인 장애와 지속적인 장애를 각각 효과적으로 처리할 수 있다

---

## 6. 소스 코드 

* [Circuit Breaker Demo](https://github.com/sanghoon-lee/circuit-breaker)

---

## 7. 관련 포스팅

* [[학습기록] 서킷 브레이커 패턴(Circuit Breaker Pattern): 개념정리](https://sanghoon-lee.github.io/2026/04/01/circuitbreaker/)

---

#CircuitBreaker #Retry #Resilience4j #SpringBoot #MSA #장애대응 #토이프로젝트

