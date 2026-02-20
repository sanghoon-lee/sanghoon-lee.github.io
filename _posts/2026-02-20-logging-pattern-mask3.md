---
layout: post
title: "[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (3) 테스트 방법 및 로그 마스킹 구현 방식의 한계"
date: 2026-02-20
categories: 토이프로젝트
---

이제부터 로그 마스킹의 테스트 방법과 구현 방식에는 어떤 한계가 있는지 설명해보도록 하겠습니다.

## 테스트 방법

테스트를 위해서 간단하게 문자열을 입력받아서 그대로 응답해주는 Echo API를 다음과 같이 구현했습니다.

* URL: POST /api/payload
* Content-Type: application/json
* Parameters
    <br>- message: 애플리케이션에 보낼 문자열 메시지 

API 요청을 받으면 매개변수(Parameter)로 전달된 message의 값을 로그로 출력하도록 했습니다.

**PayloadController.java**
```java
@PostMapping("/api/payload")
public APIResponse<PayloadResponse> publish(@Valid @RequestBody PayloadRequest request){
    PayloadResponse payloadResponse = payloadService.publish(request);

    log.info("received : {}",request.getMessage());

    return APIResponse.ok(payloadResponse);
}
```

**예시: 테스트 API 요청**
```bash
curl -X POST "http://localhost:8080/api/payload" \
  -H "Content-Type: application/json" \
  -d '{"message":"phone=010-1234-5678, email=abcdef@gmail.com"}'
```

**예시: 테스트 API 응답 및 로그 출력**
```bash
{
  "code": 200,
  "status": "OK",
  "message": "OK",
  "data": {
    "message": "phone=010-1234-5678, email=abcdef@gmail.com",
    "created": "09:12:34.567"
  }
}

received : phone=010-****-5678, email=ab****@gmail.com
```

## 로그 마스킹 구현 방식의 한계

토이 프로젝트에서 로그 마스킹을 구현한 방식은 매우 단순하고 효과적입니다. 하지만, 한계는 분명 존재합니다.

### 1. 성능

정규식으로 문자열에서 특정 패턴이 존재하는지 검사하는 코드는 성능에 부담을 줄 수 있습니다. 그래서 Trigger 기반 필터링을 활용하여 실제로 민감 정보가 포함된 경우에만 정규식 검사가 수행되도록 했습니다. 

하지만, 매우 긴 로그 문자열이나 복잡한 패턴이 포함되는 경우 마스킹 처리 속도가 느려질 수 있습니다. 이때에는 추가적으로 패턴 최적화 및 캐싱을 통해 성능 향상을 고려해야 합니다.

### 2. 패턴 변경의 유연성 부족

애플리케이션 실행 중에 Rule에 정의된 마스킹 규칙을 동적으로 변경할 수 없습니다. 현재 구현된 방식에서는 `logback.xml` 파일의 내용을 수정하고, 애플리케이션을 재기동해야 합니다.

## 마무리

로그 마스킹은 보안적인 측면뿐만 아니라, 시스템의 안정성 측면에서도 매우 중요한 기능입니다. 단순히 민감 정보를 숨기는 것뿐만 아니라, 정확하고 일관된 방식으로 로그를 통제해야 합니다.

단순한 문자열 패턴 매칭 기반으로 로그 마스킹 기능을 구현하고, 그에 대한 성능 최적화와 확장성까지 고민했습니다. 이 과정에서 배운 지식과 경험을 실무에 적용해서 애플리케이션의 로그 관리에 어려움을 겪고 있는 현실의 문제를 조금이라도 개선할 수 있도록 활용해 볼 계획입니다.

## 포스팅 시리즈

* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (1) 순수 문자열 패턴 기반으로 설계](https://sanghoon-lee.github.io/2026/02/16/logging-pattern-mask/)
* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세](https://sanghoon-lee.github.io/2026/02/18/logging-pattern-mask2/)
* [토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (3) 테스트 방법 및 로그 마스킹 구현 방식의 한계

## 참고: 소스 코드 
* [logging-pattern-mask](https://github.com/sanghoon-lee/logging-pattern-mask)
