---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: Spring Boot 애플리케이션에 Scouter Agent 연동과 모니터링 시작"
date: 2026-07-26
series: paygerduty
description: 
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Server`를 구성했다고 해서 애플리케이션의 상태가 자동으로 수집되지는 않습니다.

CPU 사용률, Heap 메모리, Active Service, XLog와 같은 운영 정보를 수집하려면 모니터링 대상 애플리케이션에 `Scouter Agent`를 연동해야 합니다.

`Scouter Agent`가 연동되면 `Scouter Client`를 통해 해당 애플리케이션의 다양한 운영 정보를 실시간으로 확인할 수 있습니다.

---

## 1. Spring Boot 애플리케이션에 Scouter Agent 연동하기

먼저 `Scouter Agent`를 다운로드한 뒤, 설정 파일과 JVM 옵션을 추가해 애플리케이션과 함께 실행되도록 구성했습니다.

---

### 1.1. Scouter Agent 다운로드

`Scouter` 프로젝트의 공식 GitHub 저장소의 주소는 다음과 같습니다.

* [https://github.com/scouter-project/scouter](https://github.com/scouter-project/scouter)

공식 GitHub 저장소의 `release` 브랜치에서 `2.17.1` 태그를 선택했습니다.

<img class="main-image" src="/assets/images/scouter-release-tags.jpg" alt="Scouter 프로젝트 Release 브랜치의 Tag 목록">

그리고 페이지 하단에 위치한 **Assets**에서 `scouter-all-2.17.1.tar.gz` 파일을 다운로드했습니다. 

<img class="main-image" src="/assets/images/scouter-assets.jpg" alt="Scouter 2.17.1 배포 버전의 Assets 목록">

압축을 해제하면 `scouter/agent.java` 디렉터리에서 `Scouter Agent` 연동에 필요한 `scouter.agent.jar` 파일을 확인할 수 있습니다.

**참고**

> `scouter-all-2.17.1.tar.gz`에는 Server, Client, Agent 등 Scouter를 구성하는 모든 구성 요소가 포함되어 있습니다. 이번 실습에서는 Agent 연동에 필요한 `scouter.agent.jar`만 사용합니다.

---

### 1.2. scouter.agent.jar 파일 이동

작성중




