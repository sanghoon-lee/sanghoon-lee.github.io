---
layout: post
title: "[토이프로젝트] Spring Boot Logback 로그 마스킹 구현: (1) PatternLayout 방식을 선택한 이유"
date: 2026-02-16
description: Spring Boot Logback PatternLayout을 이용한 로그 마스킹 프로젝트의 첫 번째 글입니다. 코드 직접 처리, JSON 직렬화, PatternLayout 방식을 비교하고 문자열 기반 방식을 선택한 이유를 설명합니다.
image: /assets/images/security.png
categories: 토이프로젝트
tags:
  - 보안
  - 자바
---

실제 서비스를 운영하다 보면 전화번호나 이메일 주소 같은 민감정보가 로그에 그대로 남는 경우를 종종 볼 수 있습니다.

민감정보를 반드시 마스킹해서 로그에 출력하도록 규칙을 만들어도 개발자가 빠뜨리는 경우가 생기기 때문입니다.

<img class="main-image" src="/assets/images/security.png" alt="민감정보">

그렇다면 개발자가 신경 쓰지 않아도 로그 출력 시점에 자동으로 마스킹할 수는 없을까요?

이런 질문에 대해 해답을 찾아보기 위해 토이프로젝트를 시작하게 되었습니다.

---

## 1. 로그 마스킹 구현 방식 비교

선택지는 몇 가지로 나눌 수 있었습니다.

| 방식       | 장점       | 단점        |
| -------- | -------- | --------- |
| 코드에서 직접 처리 | 단순       | 개발자 실수 가능 |
| JSON 직렬화 | 구조적      | JSON만 가능  |
| Logback  | 코드 수정 없음 | 문자열 패턴 필요 |


---

### 1.1. 코드에서 직접 처리

로그를 남기기 전에 개발자가 직접 값을 가공하는 방식으로 가장 직관적이기는 합니다.

```java
log.info("phone={}", mask(phoneNumber));
```

단순해 보이지만 실질적으로는 문제가 있습니다. 모든 개발자가 일관되게 적용해야 하기 때문입니다. 빠뜨리는 순간 그대로 민감정보가 노출됩니다. 

기존 코드 전체를 점검해야 하며 개발자의 주의에도 의존해야 합니다.

---

### 1.2. JSON 직렬화

`Jackson` 모듈을 확장하는 방식입니다. 민감정보에 해당되는 특정 필드에 어노테이션을 붙이면 `JSON` 형식의 데이터를 직렬화하는 시점에 `ObjectMapper`를 커스터마이징해서 자동으로 민감정보가 마스킹되도록 구현할 수 있습니다.

하지만 실제로 구현해보니 몇 가지 문제점이 보였습니다.

* `JSON` 형식이 아닌 일반 문자열은 어떻게 처리할 것인가?
* `JSON`과 일반 문자열이 조합되면 어떻게 처리할 것인가?
* 직렬화/역직렬화 비용은 무시해도 되는가?

무엇보다 구조가 복잡했습니다. 그래서 이번 프로젝트에서는 `JSON` 직렬화 과정에 개입하기보다, 로그가 최종적으로 출력되는 시점에 문자열을 처리하는 방식이 더 적합하다고 판단했습니다.

---

### 1.3. Logback

`Logback`의 `PatternLayout`을 확장하여 출력 문자열을 가공하는 방식입니다.

이 방식의 장점은 명확했습니다.

* 애플리케이션 코드 수정 불필요
* 모든 로그에 일괄 적용
* 출력 직전 1회 처리
* JSON 파싱 불필요 

로그는 최종적으로 문자열로 출력됩니다. `JSON` 객체든 일반 문자열이든 로그에 기록되는 순간에는 모두 문자열이 됩니다.

그렇다면 **"출력 직전에 한 번만 가공하면 충분하지 않을까요?"**

문자열에서 특정 패턴만 찾아 바꿔도 대부분의 로그 마스킹 요구사항을 해결할 수 있으며, 가장 단순하면서도 다양한 로그에 공통으로 적용할 수 있는 방법이라고 판단했습니다.

따라서 이번 프로젝트는 `Logback`의 `PatternLayout`을 확장해 출력 직전 문자열을 마스킹하는 방식으로 구현하게 되었습니다.

---

## 2. 프로젝트 구성

```nohighlight
sanghoon.study.logging.mask
 │
 ├─LoggingPatternMaskApplication
 │
 ├─api
 │ ├─ controller
 │ │  └─ PayloadController
 │ ├─ service
 │ │  ├─ request
 │ │  │  └─ PayloadRequest 
 │ │  ├─ response
 │ │  │  └─ PayloadResponse
 │ │  └─ PayloadService
 │ └─ APIResponse
 │
 ├─core
 │ ├─ SensitiveStringSanitizer
 │ └─ DefaultSensitiveStringSanitizer
 │    └─ Rule
 │
 └─logback
   ├─ RuleSpecParser
   └─ MaskingPatternLayout
```

* `api` → 로그 마스킹 동작을 확인하기 위한 테스트 API
* `core` → 문자열 기반 마스킹을 수행하는 핵심 로직
* `logback` → `Logback`의 `PatternLayout`을 확장하여 출력 직전에 로그를 마스킹

---

## 3. 다음 포스팅

지금까지 여러 로그 마스킹 방식을 비교하고, `PatternLayout` 기반의 문자열 처리 방식을 선택한 이유를 살펴봤습니다.

다음 포스팅에서는 `PatternLayout`을 직접 확장하고 정규식을 이용해 전화번호와 이메일 주소를 자동으로 마스킹하는 기능을 단계별로 구현해보겠습니다.