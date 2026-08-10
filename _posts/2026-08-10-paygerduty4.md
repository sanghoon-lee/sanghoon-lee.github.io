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

지난 포스팅에서는 `Scouter Client`를 설치하고, 애플리케이션으로 유입되는 요청이 없는 상태에서 CPU 사용률, Heap Memory, Active Service, XLog 등 기본 모니터링 화면의 구성과 각 지표의 의미를 살펴봤습니다.

이번에는 `JMeter`를 이용해 애플리케이션에 부하를 발생시키면서 운영 정보가 어떻게 변화하는지 확인해보겠습니다. 또한 Active Service와 XLog를 통해 요청 처리 상황을 분석하는 방법도 함께 살펴보겠습니다.

---

## 1. 부하 테스트 준비

애플리케이션에 부하를 발생시키기 위해 `JMeter`를 다음과 같이 구성했습니다.

- Thread Group
  - Number of Threads : 50
  - Ramp-up Period : 10s
  - Loop Count : Infinite
- HTTP Request
  - GET /api/server

이번 테스트의 목적은 `JMeter` 사용법을 설명하는 것이 아니라 Scouter에서 수집되는 운영 정보의 변화를 확인하는 것입니다. 따라서 단일 API를 반복 호출하는 단순한 시나리오로 테스트를 진행했습니다.

---

## 2. CPU 사용률 변화

---

## 3. Heap Memory 변화

---

## 4. Active Service 변화

---

## 5. XLog 분석

---

## 6. 마무리

작성중