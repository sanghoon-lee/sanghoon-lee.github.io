---
layout: post
title: "JMeter를 이용한 Vault Transit Engine 성능 테스트"
date: 2026-05-15
description: JMeter를 이용해서 Vault Transit Engine의 성능을 테스트해 본 과정을 정리해서 작성한 글입니다.
image: /assets/images/hashicorp.jpg
categories: 학습기록
---

---

시크릿 관리 통제 강화와 애플리케이션 내부에서 수행하던 암호화 연산의 외부화를 핵심 방향으로 Vault 기반 보안체계 구축을 검토하게 되었습니다.

사내 담당자들과 논의하는 과정에서 과거 HSM을 도입해서 암호화 연산의 외부화를 시도했으나 성능 이슈와 운영 안정성 문제로 실패했던 사례가 있었다는 이야기를 들을 수 있었습니다.

암호화 연산을 외부화하여 애플리케이션과 분리하는 구조에서는 성능과 응답 안정성이 매우 중요한 요소입니다. 따라서 Vault Transit Engine의 성능 특성과 기능적 적합성을 사전에 검증할 필요가 있었습니다.

---

## 1. Apache JMeter

과거 정부R&D과제를 수행하면서 KOLAS(Korea Laboratory Accreditation Scheme) 공인시험기관을 통해 소프트웨어의 성능과 기능을 시험하고, 공인 성적서를 발급받아 제출했던 경험이 있습니다. 당시 성능 테스트 도구로 Apache JMeter를 주로 사용했습니다. 

<img class="main-image" src="/assets/images/jmeter.png" alt="Apache JMeter">

Apache JMeter는 다양한 시스템의 부하 및 성능 테스트에 널리 사용되는 대표적인 오픈소스 기반의 성능 테스트 도구이며, 공인시험 환경에서도 활용되고 있습니다.

> KOLAS: 힌국인정기구(Korea Laboratory Accreditation Scheme)의 약자로, 국제 표준에 따라 시험 교정 검사 시관의 기술 능력과 품질 시스템을 평가하여 공인하는 산업통상자원부 산하 국가기구입니다.

---

## 2. 테스트 환경 

테스트를 위해서 최신 버전인 Apache JMeter 5.6.3 바이너리 파일을 다운로드 받아서 설치했습니다.

<img class="sub-image" src="/assets/images/jmeter_app.png" alt="Apache JMeter 실행화면">

---

## 3. 테스트 결과

---

#JMeter #부하테스트 #성능테스트 #vault #TPS