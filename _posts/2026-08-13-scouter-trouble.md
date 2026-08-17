---
layout: post
title: "[트러블슈팅] Spring Boot 3 + Scouter 2.17.1에서 XLog가 표시되지 않는 문제 해결"
date: 2026-08-13
description: Spring Boot 3 환경에서 Scouter 2.17.1 사용 시 Heap 정보는 수집되지만 XLog와 Active Service가 표시되지 않는 원인을 분석하고 해결한 과정을 정리합니다.
image: /assets/images/monitoring.jpg
categories: 트러블슈팅
tags:
  - 인프라
  - 자바
---

Spring Boot로 개발된 애플리케이션에 `JMeter`로 부하를 발생시키고, `Scouter Server`에 수집되는 운영 정보의 변화를 확인하는 토이프로젝트를 진행하던 중 이상한 현상을 발견했습니다.

분명 애플리케이션은 정상적으로 실행되고 있었고, `Scouter Agent`도 애플리케이션에 정상적으로 적용된 것으로 보였습니다. 하지만 애플리케이션에 요청을 계속 전송해도 `Active Service`와 `XLog`에는 아무런 정보도 표시되지 않았습니다.

---

## 1. 문제 상황

애플리케이션의 실행 환경은 다음과 같았습니다.

* Spring Boot 3.5.14
* Scouter 2.17.1
* Java 21
* Embedded Tomcat

그리고 단일 API를 반복 호출하는 단순한 시나리오로 애플리케이션에 부하를 발생시킬 수 있도록 `JMeter`를 아래와 같은 조건으로 구성했습니다.  

- Thread Group
  - Number of Threads : 50
  - Ramp-up Period : 10s
  - Loop Count : Infinite
- HTTP Request
  - Protocol : http
  - Server IP : 192.168.56.11
  - Port : 8081
  - GET /api/server

부하 테스트를 시작하자 `Heap Used` 창에서는 메모리 사용량의 변화가 나타났습니다.

<img class="main-image" src="/assets/images/loadtest.png" alt="부하 발생시 Heap Used 변화">

하지만 `XLog`와 `Active Service` 창에는 아무런 정보도 표시되지 않았습니다.

<div class="image-row">
  <figure>
    <img class="main-image" src="/assets/images/active-service.jpg" alt="변화 없는 Active Service">
    <figcaption>변화 없는 Active Service 창</figcaption>
  </figure>

  <figure>
    <img class="main-image" src="/assets/images/xlog.jpg" alt="변화 없는 XLog">
    <figcaption>변화 없는 XLog 창</figcaption>
  </figure>
</div>

---

## 2. 애플리케이션과 Scouter Agent 동작 확인

`curl` 명령을 이용해 직접 애플리케이션의 API를 호출해보면 정상적으로 응답이 반환되었습니다. 

```bash
$ curl http://192.168.56.11:8081/api/server
```

```json
{
    "time":"2026-08-12T12:17:54.806022143",
    "serverName":"api-01"
}
```

부하 테스트를 진행하는 동안 `Heap Used` 창에서는 메모리 사용량의 변화가 나타났고, `Object Tree`에도 애플리케이션이 정상적으로 등록되어 있었습니다. 즉, JVM 관련 정보는 정상적으로 수집되고 있는 상태였습니다.

이를 통해 `Scouter Agent` 자체가 동작하지 않는 문제는 아니라는 것을 확인할 수 있었습니다.

---

## 3. 원인 

그렇다면 왜 `XLog`와 `Active Service` 창에는 아무런 정보도 표시되지 않았을까요? 

`Active Service`, `XLog`와 같은 요청 추적 정보는 애플리케이션으로 유입되는 HTTP 요청을 추적해야 생성됩니다.

Spring Boot 3부터는 기존 Java EE의 `javax.*` API 대신 Jakarta EE의 `jakarta.*` API를 사용합니다. Servlet API 역시 `javax.servlet.*`에서 `jakarta.servlet.*`로 변경되었습니다.

애플리케이션은 Spring Boot 3.5.14 버전을 사용하고 있었지만, 애플리케이션에 적용된 `Scouter Agent`는 2.17.1 버전이었습니다.

| 구분                 | 사용 환경                  |
| ------------------ | ---------------------- |
| Spring Boot 3.5.14 | `jakarta.servlet.*` 사용 |
| Scouter Agent 2.17.1     | Jakarta Servlet 미지원 |
| Scouter Agent 2.20.0     | Jakarta Servlet 지원 |

결국 Spring Boot 3.5.14와 `Scouter Agent` 2.17.1의 호환성 문제가 원인이었습니다. Scouter Agent 자체는 동작하고 있었기 때문에 JVM 관련 정보는 수집할 수 있었지만, Jakarta Servlet 기반의 HTTP 요청을 정상적으로 추적하지 못해 `Active Service`와 `XLog` 창에는 아무 정보도 표시되지 않았던 것입니다.

**참고**

> Scouter 릴리스 노트를 살펴보니 v2.20.0에서 Java Agent에 Jakarta Servlet 지원이 추가된 것을 확인할 수 있었습니다.
> <br>
> [Scouter 2.20.0 버전 릴리스 노트](https://github.com/scouter-project/scouter/releases/tag/v2.20.0)

---

## 4. 문제 해결

`Scouter Agent`를 2.20.0 버전으로 변경하고, 동일한 방법으로 애플리케이션에 요청을 전송했습니다.

이번에는 이전과 달리 `Active Service`에 처리 중인 요청이 표시되었고, `XLog`에도 처리된 요청이 정상적으로 나타나는 것을 확인할 수 있었습니다.

<div class="image-row">
  <figure>
    <img class="main-image" src="/assets/images/xxxx" alt="정상적으로 표시되는 Active Service">
    <figcaption>Active Service</figcaption>
  </figure>

  <figure>
    <img class="main-image" src="/assets/images/xxxx" alt="정상적으로 표시되는 XLog">
    <figcaption>XLog</figcaption>
  </figure>
</div>

---

## 5. 마무리

이번 문제를 통해 일부 정보만 정상적으로 수집된다고 해서 `Scouter`의 모든 모니터링 기능이 정상적으로 동작하는 것은 아니라는 점을 알게 되었습니다.

JVM 상태를 수집하는 것과 HTTP 요청을 추적하는 것은 서로 다른 영역이기 때문에, 일부 정보만 정상적으로 보인다면 Agent의 연결 상태뿐만 아니라 애플리케이션 환경과 `Scouter` 버전의 호환성도 함께 확인할 필요가 있습니다.

특히 Spring Boot 3 환경에서 JVM 관련 정보는 정상적으로 수집되지만 `Active Service`나 `XLog`가 표시되지 않는다면, 사용 중인 `Scouter Agent`가 Jakarta Servlet을 지원하는 버전인지 확인해보는 것이 좋습니다.