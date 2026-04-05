---
layout: post
title: "[토이 프로젝트] Circuit Breaker + Retry를 직접 구현해보며 이해하기"
date: 2026-01-31
categories: 토이프로젝트
---

직전 포스팅에서 서킷 브레이커 패턴(Circuit Breaker Pattern)의 개념을 정리했습니다.

개념적으로는 충분히 이해했지만, 코드로 직접 구현해보며 동작을 확인해보고 싶었습니다.  
팀원들에게 서킷 브레이커 도입의 필요성을 설명하면서, 실제로 어떤 방식으로 구현할 수 있는지도 함께 보여주면 좋겠다는 생각이 들었기 때문입니다.

그래서 간단한 토이 프로젝트를 구성해보았습니다.

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

내부 서비스 호출만으로도 동작을 확인할 수 있도록 설계했기 때문에,
하나의 애플리케이션 내에서 테스트가 가능합니다.

또한 toggle API를 통해 내부 서비스의 상태를 의도적으로 장애 또는 정상 상태로 전환할 수 있도록 구성했습니다.


#CircuitBreaker #Retry #Resilience4j #SpringBoot #MSA #장애대응 #토이프로젝트
