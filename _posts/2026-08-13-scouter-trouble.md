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

Spring Boot로 개발된 애플리케이션에 JMeter로 부하를 발생시켜 Scouter에서 운영 정보가 어떻게 변화하는지 확인하기 위한 토이프로젝트를 진행하던 도중에 이상 현상을 발견했습니다.

분명 애플리케이션은 정상적으로 실행되고 있었고, Scouter Agent도 정상적으로 연동된 상태였습니다.

하지만 아무리 애플리케이션에 요청을 전송해도 XLog 창에는 아무 표시도 나타나지 않았습니다.

---

## 1. 문제 상황

모니터링 대상 애플리케이션의 실행 환경은 다음과 같습니다.

* Spring Boot 3.5.14
* Scouter 2.17.1
* Java 21
* Embedded Tomcat

아래와 같은 조건으로 JMeter를 구성하고, 단일 API를 반복 호출하는 단순한 시나리오로 애플리케이션에 요청을 집중시켰습니다.

- Thread Group
  - Number of Threads : 50
  - Ramp-up Period : 10s
  - Loop Count : Infinite
- HTTP Request
  - Protocol : http
  - Server IP : 192.168.56.11
  - Port : 8081
  - GET /api/server

부하를 발생시키자 Heap Used 창을 통해 메모리 사용량의 변화가 관찰되었습니다.

<img class="main-image" src="/assets/images/loadtest.png" alt="부하 발생시 Heap Used 변화">

하지만 XLog, Active Service 등은 아무런 정보도 출력되지 않았습니다.

---

## 2. 애플리케이션과 Scouter Agent 동작 확인

직접 애플리케이션의 API를 호출해 봤지만 정상적으로 응답이 반환되었습니다. 

```bash
$ curl http://192.168.56.11:8081/api/server                                                                                                             er
```

```json
{
    "time":"2026-08-12T12:17:54.806022143",
    "serverName":"api-01"
}
```

더 이상했던 점은 JVM 관련 정보는 정상적으로 수집되고 있었다는 것입니다. Heap Used 창뿐만 아니라 Object Tree에도 정상적으로 애플리케이션이 등록되어 있었습니다.

이를 통해 Scouter Agent 자체는 정상적으로 동작하고 있다는 것을 확인할 수 있었습니다.

그렇다면 왜 XLog와 Active Service는 생성되지 않는 것일까요?

---

## 3. Scouter Agent가 운영 정보를 수집하는 방식

Scouter Agent가 수집하는 정보 중 Heap Memory, GC, CPU, Thread와 같은 JVM 관련 정보는 JVM 내부의 상태를 기반으로 수집됩니다.

반면 Active Service, XLog, Profile과 같은 요청 추적 정보는 애플리케이션으로 들어오는 요청을 추적해야 생성됩니다.

JVM 정보가 정상적으로 보인다고 해서 HTTP 요청까지 정상적으로 추적되고 있다는 의미는 아닙니다.

즉, JVM 관련 정보는 정상적으로 수집되고 있었지만 애플리케이션으로 들어오는 HTTP 요청은 정상적으로 추적되지 않고 있었던 것입니다.

---

## 4. 원인

Spring Boot 3부터는 Java EE API가 Jakarta EE로 변경되었습니다.

예를 들어 Servlet API도 `javax.servlet.*`에서 `jakarta.servlet.*` 로 변경되었습니다.

그런데 제가 사용한 Scouter 버전은 2.17.1이었습니다.

Scouter 릴리스 노트를 살펴보니 v2.20.0에서 Java Agent에 Jakarta Servlet 지원이 추가된 것을 확인할 수 있었습니다.

| 구분                 | 사용 환경                  |
| ------------------ | ---------------------- |
| Spring Boot 3.5.14 | `jakarta.servlet.*` 사용 |
| Scouter 2.17.1     | Jakarta Servlet 미지원    |
| Scouter 2.20.0     | Jakarta Servlet 지원     |

결국 Spring Boot 3.5.14와 Scouter 2.17.1의 호환성 문제가 원인이었습니다. Scouter Agent 자체는 정상적으로 동작하고 있었기 때문에 JVM 관련 정보는 수집할 수 있었지만, Jakarta Servlet 기반의 HTTP 요청을 정상적으로 추적하지 못해 Active Service와 XLog가 생성되지 않았던 것입니다.

**참고**

> [Scouter 2.20.0 버전 릴리스 노트](https://github.com/scouter-project/scouter/releases/tag/v2.20.0)

---

## 5. 해결 방법

Jakarta Servlet을 지원하는 Scouter 2.20.0으로 버전을 변경해서 문제를 해결할 수 있었습니다.

하지만 현재 Docker Hub에 공개된 최신 이미지가 2.17.1 버전이었기 때문에 Scouter 2.20.0 배포 파일을 다운로드하고 Docker 이미지를 직접 생성해서 사용했습니다.

이후 진행하고 있던 토이프로젝트를 Scouter 2.20.0 버전으로 변경했고, 관련 포스팅도 새로운 환경에 맞게 수정했습니다.

---

## 6. 마무리

이번 문제를 통해 Heap 정보가 정상적으로 수집된다고 해서 Scouter의 모든 모니터링 기능이 정상적으로 동작하는 것은 아니라는 점을 알게 되었습니다.

JVM 상태를 수집하는 것과 HTTP 요청을 추적하는 것은 서로 다른 영역이기 때문에, 일부 정보만 정상적으로 보인다면 Agent의 연결 상태뿐만 아니라 애플리케이션 환경과 Scouter 버전의 호환성도 함께 확인할 필요가 있습니다.

특히 Spring Boot 3 환경에서 Heap 정보는 정상적으로 보이지만 Active Service나 XLog가 생성되지 않는다면 Jakarta Servlet을 지원하는 Scouter 버전인지 확인해보는 것이 좋습니다.