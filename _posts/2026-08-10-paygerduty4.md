---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (4) JMeter로 부하를 발생시키며 운영 정보 분석하기"
date: 2026-08-10
series: paygerduty
description: 
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

지난 포스팅에서는 `Scouter Client`를 설치하고, CPU 사용률, Heap Memory, Active Service, XLog 등 기본 모니터링 화면의 구성과 각 지표의 의미를 살펴봤습니다.

애플리케이션에 요청이 집중되면 응답 지연이 발생할 수 있습니다. 이러한 상황에서 운영 정보가 어떻게 변화하는지 확인하고, Active Service와 XLog를 활용해 응답 지연의 원인을 분석해보겠습니다.

---

## 1. 부하 테스트 준비

이를 위해 `JMeter`를 다음과 같이 구성했습니다.

- Thread Group
  - Number of Threads : 50
  - Ramp-up Period : 10s
  - Loop Count : Infinite
- HTTP Request
  - GET /api/server

이 테스트의 목적은 애플리케이션에 요청을 집중시켜 응답 지연 현상을 재현하는 것입니다. 따라서 단일 API를 반복 호출하는 단순한 시나리오로 테스트를 진행했습니다.

---

## 2. CPU 사용률 변화

---

## 3. Heap Memory 변화

---

## 4. Active Service 변화

---

## 5. XLog로 요청 분석하기

---

## 6. 마무리

작성중