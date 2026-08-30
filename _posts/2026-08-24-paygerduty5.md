---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (5) Scouter Alert 설정과 장애 감지 테스트"
date: 2026-08-24
series: paygerduty
description: Scouter Alert 기능을 설정하고 애플리케이션에 이상 상황을 발생시켜, 설정한 조건에 따라 Alert가 정상적으로 발생하는지 확인합니다.
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Client`를 통해 서비스 상태를 실시간으로 확인하고 원인을 분석하는 데 큰 도움을 받을 수 있었습니다. 하지만 담당자가 운영 정보를 직접 확인해야 이상 상황을 발견할 수 있다는 한계가 있습니다.

새벽이나 휴일처럼 담당자가 운영 정보를 직접 확인하기 어려운 시간에도 장애는 얼마든지 발생할 수 있습니다. 따라서 운영 정보를 모니터링하는 것뿐만 아니라, 이상 상황을 자동으로 감지하고 담당자에게 알려주는 체계가 필요합니다.

---

## 1. 이상 상태 감지(Alert)와 알림(Notification)

이상 상황을 자동으로 감지하고 담당자에게 전달하는 과정을 이해하려면 먼저 `Alert`와 `Notification`의 차이를 구분할 필요가 있습니다.

`Alert`는 시스템에서 미리 정의한 조건에 해당하는 이상 상황이 발생했음을 감지하는 것이고, `Notification`은 발생한 Alert를 문자 메시지, 메신저, 전화 등의 수단으로 담당자에게 전달하는 것입니다.

예를 들어 애플리케이션의 CPU 사용률이 일정 수준 이상으로 올라가는 상황을 생각해볼 수 있습니다. CPU 사용률이 설정한 기준을 초과한 것을 감지하는 것이 `Alert`이고, 이를 담당자에게 문자 메시지나 전화로 알려주는 것이 `Notification`입니다.

---

## 2. Scouter Alert 설정

`Scouter Agent`는 애플리케이션의 운영 정보를 수집하여 `Scouter Server`로 전달하고, `Scouter Server`는 전달받은 운영 정보를 저장합니다. Alert 역시 `Scouter Server`에 수집된 운영 정보를 기준으로 설정한 조건에 해당하는지를 판단하여 발생합니다.

`Scouter`에서는 사용자가 직접 Alert 조건을 정의할 수 있습니다. 이러한 기능을 `Customizable Alert`라고 하며, `Scouter Client`에서 Alert Script를 작성하거나 `Scouter Server`의 `plugin` 디렉터리에 Alert 설정 파일을 직접 생성하는 방식으로 구성할 수 있습니다.

---

### 2.1. Alert Script 저장 경로 구성

Alert 조건은 Script 형태로 작성하며, 기본적으로 `Scouter Server`가 실행되는 디렉터리의 `plugin` 경로에 관련 파일을 배치합니다.

현재 `Scouter Server`는 Docker 컨테이너로 실행하고 있습니다. Alert Script를 컨테이너 내부에서 직접 관리하면 컨테이너를 다시 생성할 때 파일이 사라질 수 있으므로, 호스트의 `plugin` 디렉터리를 Docker Volume으로 연결해 사용했습니다.

이를 위해 호스트에 `plugin` 디렉터리를 생성했습니다.

```bash
$ mkdir -p /home/sanghoon/scouter-server/plugin
```

그리고 `docker-compose.yml`에 `plugin` 디렉터리에 대한 Volume 설정을 추가했습니다.

```yaml
services:
  scouter-server:
    image: scouter-server:2.20.0
    container_name: scouter-server
    restart: unless-stopped
    volumes:
      - /home/sanghoon/scouter-server/database:/scouter/server/database
      - /home/sanghoon/scouter-server/logs:/scouter/server/logs
      - /home/sanghoon/scouter-server/plugin:/scouter/server/plugin
    ports:
      - "6100:6100/tcp"
      - "6100:6100/udp"
```

이렇게 하면 호스트의 `/home/sanghoon/scouter-server/plugin` 디렉터리와 컨테이너의 `/scouter/server/plugin` 디렉터리가 연결됩니다.

변경한 Volume 설정을 적용하기 위해 컨테이너를 다시 생성했습니다.

```bash
$ docker compose down
$ docker compose up -d
```

이제 호스트의 plugin 디렉터리에서 Alert Script를 작성하면, `Scouter Server` 컨테이너에서도 동일한 Script를 사용할 수 있습니다.

---

### 2.2. Alert Script 작성

```java
@GetMapping("/error")
public Map<String, String> error() {
    throw new RuntimeException("Test error: "+serverName);
}
```

```
GET /api/error
      ↓
RuntimeException 발생
      ↓
Spring Boot → HTTP 500 응답
      ↓
Scouter Agent가 Exception 감지
      ↓
해당 요청의 XLog에 Error 표시
      ↓
오류 관련 집계값 증가
```