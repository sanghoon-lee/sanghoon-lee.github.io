---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (4) JMeter로 부하를 발생시키며 운영 정보 확인하기"
date: 2026-08-10
series: paygerduty
description: 웹 브라우저와 JMeter로 Spring Boot 애플리케이션에 요청과 부하를 발생시키고, Scouter의 XLog, TPS, Active Service 등 주요 운영 정보가 어떻게 변화하는지 확인합니다.
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

이전 포스팅에서는 모니터링 대상 애플리케이션으로 유입되는 요청이 없는 상태에서 `Scouter Client`가 제공하는 주요 운영 정보가 무엇인지 살펴봤습니다.

이번에는 실제로 애플리케이션에 요청을 발생시키고, `Active Service`, `XLog` 등의 운영 정보가 어떻게 변화하는지 확인해보겠습니다. 먼저 웹 브라우저를 이용해 소량의 요청을 발생시킨 뒤, JMeter를 이용해 본격적으로 부하를 발생시켜 보겠습니다.

---

## 1. API 호출 후 운영 정보 확인

먼저 웹 브라우저를 이용해 각 애플리케이션의 API를 호출하고, 해당 요청이 `Scouter Client`에 어떻게 표시되는지 확인했습니다.

* `api-01` : `GET http://192.168.56.11:8081/api/server` 
* `api-02` : `GET http://192.168.56.11:8082/api/server`

<div class="image-row">
  <figure>
    <img src="/assets/images/api-01-req.jpg" alt="api-01 애플리케이션을 대상으로 API 호출">
    <figcaption>api-01 애플리케이션을 대상으로 API 호출</figcaption>
  </figure>

  <figure>
    <img src="/assets/images/api-02.req.jpg" alt="api-02 애플리케이션을 대상으로 API 호출">
    <figcaption>api-02 애플리케이션을 대상으로 API 호출</figcaption>
  </figure>
</div>

API는 `api-01` → `api-02` → `api-01` 순서로 총 3회 호출했습니다. 그러자 `Scouter Client`의 XLog 창에 세 번의 요청이 각각 하나의 점으로 표시되었습니다. XLog뿐만 아니라 Elapsed Time, TPS, Recent User 등 다른 화면의 정보도 함께 변화하는 것을 확인할 수 있었습니다.

<img class="sub-image" src="/assets/images/after-req-01.png" alt="Scouter Client 운영정보 변화(XLog)">

XLog 창에서 확인하려는 영역을 마우스로 드래그하면 해당 영역에 포함된 요청 목록을 확인할 수 있습니다. 목록에는 각 요청의 서비스명, 응답 시간 등의 정보가 표시되며, 특정 요청을 선택하면 상세 프로파일을 확인할 수 있습니다.

<img class="sub-image" src="/assets/images/after-req-02.png" alt="Scouter Client 운영정보 변화(XLog List)">

---

## 2. JMeter로 부하 발생

이어서 `JMeter`로 일정 시간 동안 API를 반복 호출하도록 테스트 시나리오를 아래와 같이 구성했습니다.

| 구분 | 설정 | 값 |
| --- | --- | --- |
| Thread Group | Number of Threads | 50 |
| Thread Group | Ramp-up Period | 10초 |
| Thread Group | Loop Count | 1,000 |
| HTTP Request | Protocol | HTTP |
| HTTP Request | Server IP | 192.168.56.11 |
| HTTP Request | Port | 8081 |
| HTTP Request | Method | GET |
| HTTP Request | Path | /api/server |

이렇게 구성하면 50개의 Thread가 10초에 걸쳐 순차적으로 시작되고, 각 Thread가 `api-01` 애플리케이션의 `/api/server` API를 1,000회씩 반복 호출합니다. 따라서 전체 테스트에서는 총 50,000회의 요청이 발생합니다.

---

### 2.1. 부하 발생 후 운영 정보 확인

위에서 구성한 테스트를 실행해봤습니다. `JMeter`로 반복적인 요청을 발생시키자 운영 정보의 변화를 더욱 뚜렷하게 확인할 수 있었습니다.

XLog 창에 많은 점이 연속적으로 나타났고, 이와 함께 Elapsed Time, Recent User 등의 운영 정보도 지속적으로 변화했습니다. TPS(Transactions Per Second) 창에서도 부하가 발생하면서 애플리케이션이 초당 처리하는 요청 건수가 증가하는 모습을 확인할 수 있었습니다.

<img class="sub-image" src="/assets/images/after-req-03.png" alt="부하 발생 후, XLog 변화">

---

### 2.2. Active Service 변화 확인

Object Tree에서 모니터링할 애플리케이션을 마우스 오른쪽 버튼으로 클릭하고 `Performance Counter` → `Active Service`를 선택하면 Active Service 창이 추가됩니다.

Active Service에는 아직 처리가 완료되지 않은 요청과 현재까지의 처리 시간 등이 표시됩니다. 따라서 응답이 지연되고 있는 요청을 확인하거나, 특정 요청이 장시간 처리 중인 상태인지 확인할 때 활용할 수 있습니다.

XLog가 처리가 완료된 요청의 실행 이력을 보여준다면, Active Service는 현재 처리 중인 요청을 보여준다는 차이가 있습니다. 따라서 XLog를 통해 완료된 요청의 처리 결과를 확인할 수 있고, Active Service를 통해서는 현재 애플리케이션에서 어떤 요청이 처리 중인지 확인할 수 있습니다.

`api-01`의 Active Service 창을 추가한 상태에서 JMeter 테스트를 다시 실행했습니다. 요청이 반복적으로 유입되면서 Active Service의 개수가 실시간으로 변화하는 것을 확인할 수 있었습니다.

<img class="sub-image" src="/assets/images/after-req-04.png" alt="부하 발생 후, Active Service 변화">

부하가 증가하면서 Active Service의 개수는 4개까지 증가했다가 다시 감소했습니다. 이를 통해 요청이 유입되는 동안 Active Service가 증가하고, 요청 처리가 완료되면 다시 감소하는 모습을 확인할 수 있었습니다.

---

## 3. 다음 포스팅

이번 포스팅에서는 웹 브라우저와 JMeter를 이용해 애플리케이션에 요청을 발생시키고, XLog, TPS, Active Service 등의 운영 정보가 어떻게 변화하는지 확인했습니다.

지금까지는 Scouter를 이용해 애플리케이션의 상태를 직접 확인하는 방법을 살펴봤습니다. 하지만 실제 운영 환경에서는 담당자가 Scouter 화면을 계속 지켜보고 있을 수는 없습니다.

다음 포스팅에서는 특정 조건에서 장애나 이상 징후를 자동으로 감지할 수 있도록 Scouter Alert를 설정하고 동작을 확인해보겠습니다.