---
layout: post
title: "[토이프로젝트] Spring Boot Logback 로그 마스킹 구현: (3) 동작 검증과 한계"
date: 2026-02-20
description: Spring Boot Logback 로그 마스킹 기능을 API 요청으로 테스트하고, 정규식과 Trigger 기반 문자열 마스킹 방식의 한계와 운영 시 고려사항을 정리합니다.
image: /assets/images/security.png
categories: 토이프로젝트
tags:
  - 보안
  - 자바
---

지난 포스팅에서 코드를 중심으로 어떻게 로그 마스킹을 구현했는지 설명했습니다.

이번에는 테스트 API를 이용해 전화번호와 이메일 주소가 실제 로그에서 어떻게 마스킹되는지 확인해보겠습니다. 또한 정규식과 Trigger를 이용한 문자열 기반 마스킹 방식의 한계와 운영 시 고려사항도 살펴보겠습니다.

---

## 1. 테스트 API

로그 마스킹 동작을 확인하기 위해 요청으로 받은 문자열을 로그에 출력하고 그대로 응답하는 테스트 API를 구현했습니다. API 호출 과정에서 응답 데이터는 원본으로 유지되고, 로그에 기록되는 문자열만 마스킹됩니다.

**API 요청 형식**

```text
Method: POST
URL: /api/payload
Content-Type: application/json

Request Body
- message: 로그에 출력할 테스트 문자열
```

**PayloadController.java**
```java
@PostMapping("/api/payload")
public APIResponse<PayloadResponse> publish(@Valid @RequestBody PayloadRequest request){
    PayloadResponse payloadResponse = payloadService.publish(request);

    log.info("received : {}",request.getMessage());

    return APIResponse.ok(payloadResponse);
}
```

---

## 2. 테스트 결과

**요청**
```bash
curl -X POST "http://localhost:8080/api/payload" \
  -H "Content-Type: application/json" \
  -d '{"message":"phone=010-1234-5678, email=abcdef@gmail.com"}'
```

**응답**
```text
{
  "code": 200,
  "status": "OK",
  "message": "OK",
  "data": {
    "message": "phone=010-1234-5678, email=abcdef@gmail.com",
    "created": "09:12:34.567"
  }
}
```

**로그 출력**
```text
received : phone=010-****-5678, email=ab****@gmail.com
```

마스킹은 응답 데이터나 애플리케이션 내부 값을 변경하는 기능이 아닙니다. `MaskingPatternLayout`이 최종 로그 문자열만 가공하므로 API 응답에는 원본 값이 유지되고, 로그에 기록되는 값만 마스킹됩니다.

다양한 입력 데이터를 사용해 추가 테스트를 진행했고, 결과는 아래 표로 정리했습니다.

| 테스트 입력                   | 로그 출력 결과                   |
| ------------------------ | -------------------------- |
| `phone=010-1234-5678`    | `phone=010-****-5678`      |
| `email=abcdef@gmail.com` | `email=ab****@gmail.com`   |
| `message=hello`          | 원본 그대로 출력                  |
| `phone=01012345678`      | Trigger가 `010-`이면 마스킹되지 않음 |
| `phone1=010-1234-5678`, `phone2=010-9876-5432`               | 두 값 모두 마스킹                 |



---

## 3. 로그 마스킹 구현 방식의 한계

`PatternLayout` 기반 문자열 처리 방식은 구조가 단순해 적용 가능성은 높습니다. 하지만 실제 운영 환경에 사용하려면 성능과 탐지 정확성, 장애 처리 방식을 추가로 검증할 필요가 있습니다.

---

### 3.1. 정규식 처리 비용

정규식으로 문자열에서 특정 패턴이 존재하는지 검사하는 코드는 성능에 부담을 줄 수 있습니다. 

그래서 Trigger 기반 사전 필터링을 적용하여 특정 키워드가 포함된 로그에만 정규식 검사를 수행하도록 했습니다.

하지만, 매우 긴 로그 문자열이나 복잡한 패턴이 포함되는 경우 마스킹 처리 속도가 느려질 수 있습니다. 

운영 환경에서는 정규식의 복잡도와 Rule 개수를 최소화하고, 로그 길이와 처리량을 기준으로 성능 영향을 측정할 필요가 있습니다.

---

### 3.2. Trigger로 인한 탐지 누락

Trigger에 포함되지 않은 형식의 민감정보는 정규식 검사 자체가 수행되지 않습니다. 

예를 들어 Trigger가 `010-`으로 설정된 경우 하이픈이 없는 `01012345678`은 정규식과 일치하더라도 사전 필터를 통과하지 못할 수 있습니다.

---

### 3.3. 정규식 기반 탐지의 한계

문자열 기반 방식의 본질적인 한계입니다.

시스템은 개인정보의 의미를 이해하는 것이 아니라 등록된 패턴과 일치하는 문자열만 처리합니다. 새로운 형식의 계좌번호나 토큰, 주민등록번호 등이 로그에 기록되더라도 Rule이 정의되지 않았다면 마스킹되지 않습니다.

---

### 3.4. 과도한 마스킹 가능성

개인정보가 아닌 일반 문자열도 정규식 패턴과 일치하면 민감정보로 판단되어 마스킹될 수 있습니다. 따라서 Rule을 지나치게 넓게 정의하면 디버깅에 필요한 정보까지 사라질 수 있습니다.

---

### 3.5. 설정 변경 시 재기동 필요

애플리케이션 실행 중에 Rule에 정의된 마스킹 규칙을 동적으로 변경할 수 없습니다. 

현재 구현된 방식에서는 `logback.xml` 파일의 내용을 수정하고, 애플리케이션을 재기동해야 합니다.

---

## 4. 마무리

이번 시리즈에서는 여러 로그 마스킹 방식을 비교하고, `PatternLayout`을 확장하여 로그 출력 직전에 민감정보를 마스킹하는 기능을 구현했습니다. 또한 Trigger 기반 사전 필터링을 적용해 불필요한 정규식 수행을 줄이고, 테스트 API를 통해 실제 출력 결과를 확인했습니다.

문자열 기반 방식은 애플리케이션 코드를 변경하지 않고 여러 로그에 공통으로 적용할 수 있다는 장점이 있습니다. 반면 Rule로 정의하지 않은 개인정보는 탐지할 수 없고, Trigger 설정에 따라 마스킹이 누락되거나 정규식 수행 횟수가 증가할 수 있습니다.

따라서 로그 마스킹만으로 개인정보 유출 문제를 모두 해결하려 하기보다, 애초에 민감정보를 로그에 남기지 않는 원칙을 우선하고 마스킹은 민감정보가 로그에 기록되는 상황을 줄이기 위한 보조적인 안전장치로 활용하는 것이 바람직합니다.

---

## 5. 소스코드

전체 소스 코드는 아래 저장소에서 확인할 수 있습니다.

* [logging-pattern-mask](https://github.com/sanghoon-lee/logging-pattern-mask)