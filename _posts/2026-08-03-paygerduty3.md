---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (3) Scouter Client 설치와 운영 정보 모니터링"
date: 2026-08-03
series: paygerduty
description: Windows용 Scouter Client를 설치하고 Object Tree, CPU, Heap Memory, Active Service, XLog 등 주요 운영 정보를 확인하는 방법을 정리합니다.
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Agent`는 모니터링 대상 애플리케이션의 운영 정보를 수집하여 `Scouter Server`로 전송합니다. 사용자는 `Scouter Client`를 통해 `Scouter Server`에 접속하여 수집된 운영 정보를 실시간으로 확인할 수 있습니다.

---

## 1. Scouter Client 준비

앞선 포스팅에서 사용했던 Scouter 2.20.0 배포 파일인 `scouter-all-2.20.0.tar.gz`에는 Scouter Client가 포함되어 있지 않습니다.

따라서 Scouter 공식 GitHub의 [v2.20.0 Releases 페이지](https://github.com/scouter-project/scouter/releases/tag/v2.20.0)에서 운영체제에 맞는 `Scouter Client`를 별도로 다운로드했습니다. 사용하는 운영체제가 Windows인 경우에는 `scouter.client.product-win32.win32.x86_64.zip` 파일을 선택하면 됩니다.

---

### 1.1. Scouter Client 파일 배치

다운로드한 파일의 압축을 해제하면 `scouter.client` 디렉터리가 생성됩니다.

실행과 관리가 편리하도록 `scouter.client` 디렉터리를 `C:\` 아래로 이동했습니다.

<img class="sub-image" src="/assets/images/scouter.client.png" alt="scouter.client 디렉터리">

---

### 1.2. Scouter Client 실행

`scouter.client` 디렉터리에서 `scouter.exe` 파일을 실행하면 기대와 다르게 Windows의 PC 보호 경고창이 나타났습니다. `Scouter Client`가 출처를 알 수 없는 애플리케이션으로 분류되어 실행이 차단되기 때문입니다.

이런 경우에는 경고 메시지 하단의 추가 정보를 클릭하면 실행 버튼을 활성화할 수 있습니다. 하지만 실행 버튼을 클릭하기 전에 공식 GitHub 저장소에서 다운로드한 파일인지 반드시 확인하는 것이 안전합니다.

<div class="image-row">
  <figure>
    <img src="/assets/images/warning1.jpg" alt="Windows의 PC보호 화면1">
    <figcaption>추가정보 클릭 전</figcaption>
  </figure>

  <figure>
    <img src="/assets/images/warning2.jpg" alt="Windows의 PC보호 화면2">
    <figcaption>추가정보 클릭 후</figcaption>
  </figure>
</div>

실행 버튼을 클릭하자 `Scouter Client`의 로그인 화면이 나타났습니다.

<img class="main-image" src="/assets/images/scouter.login.png" alt="scouter 서버 정보 입력창">

`Server Address`에는 `Scouter Server`가 실행되고 있는 서버의 IP(192.168.56.13:6100)를 입력하고, `ID`와 `Password`에는 기본 계정인 `admin` / `admin`을 입력했습니다. OK 버튼을 클릭하자 정상적으로 로그인할 수 있었습니다.

**참고**

> 실제 운영 환경에서는 기본 비밀번호를 그대로 사용하지 않는 것이 좋습니다.

---

## 2. 운영 정보 확인

로그인이 완료되면 `Scouter Client`에서 서버 전체의 상태와 선택된 애플리케이션의 상태를 함께 확인하는 기본 화면이 제공됩니다.

<img class="sub-image" src="/assets/images/scouter1.jpg" alt="Scouter Client 기본 모니터링 화면">

---

### 2.1. Object Tree

Object Tree 창에서는 `Scouter Server`가 운영 정보를 수집하고 있는 모니터링 대상 애플리케이션의 목록을 확인하고 선택할 수 있습니다.

<img class="main-image" src="/assets/images/objecttree.jpg" alt="Object Tree">

현재 모니터링 대상 애플리케이션은 다음과 같습니다.  

* `78afd06c5c17` → `api-01`
* `ef99dd4224dd` → `api-02`

`78afd06c5c17`, `ef99dd4224dd`은 애플리케이션 컨테이너에 부여된 **hostname**입니다. `Scouter Agent` 연동 이후에 애플리케이션 컨테이너의 실행과 종료를 반복했다면, 더 많은 애플리케이션 정보가 Object Tree에 남아 있을 수 있습니다. 애플리케이션 컨테이너가 종료되더라도 `Scouter Server`에는 이전에 수집된 정보가 일정 기간 남아 있을 수 있기 때문입니다.

`api-01`, `api-02`는 애플리케이션 컨테이너를 구분하기 위해 Docker Compose의 `JAVA_TOOL_OPTIONS`를 통해 `-Dobj_name`으로 지정했던 이름입니다. 

---

### 2.2. Heap Used(JVM Heap 사용량)

Heap Used 창에서는 Object Tree에서 선택된 애플리케이션의 JVM Heap 사용량 변화를 실시간으로 확인할 수 있습니다.

<img class="main-image" src="/assets/images/heap-memory.jpg" alt="Heap Used">

일반적으로 요청을 처리하면서 점차 증가하다가, Garbage Collection(GC)이 수행되면 사용하지 않는 객체가 정리되면서 다시 감소하는 모습을 확인할 수 있습니다.

만약 Heap 사용량이 지속적으로 증가하고 GC 이후에도 거의 줄어들지 않는다면 메모리 누수(Memory Leak)를 의심해볼 수 있습니다. 또한 Heap 사용량이 최대치에 자주 도달한다면 애플리케이션의 메모리 사용 패턴과 GC 동작을 분석하고, 필요한 경우 JVM Heap 크기가 적절한지도 함께 검토할 필요가 있습니다.

**참고**

> Object Tree에서 애플리케이션을 클릭하면 모니터링할 애플리케이션을 선택하거나 선택을 해제할 수 있습니다.

---


### 2.3. XLog(요청 처리 이력)

XLog 창에서는 애플리케이션에서 **처리가 완료된 요청의 실행 결과**을 확인할 수 있습니다.

<img class="sub-image" src="/assets/images/xlog.jpg" alt="XLog">

각 요청은 하나의 점(Point)으로 표시되며, 점의 위치를 통해 요청이 발생한 시각과 처리 시간을 직관적으로 확인할 수 있습니다. 점이 위쪽에 위치할수록 요청 처리 시간이 길다는 의미이므로, 응답이 느린 요청을 한눈에 찾을 수 있습니다. 특정 요청을 더블 클릭하면 해당 요청의 상세 실행 정보도 확인할 수 있습니다.

응답 시간이 비정상적으로 오래 걸린 요청이나 오류가 발생한 요청을 빠르게 찾아 분석할 수 있기 때문에, 장애 원인을 추적할 때 가장 많이 활용되는 화면 중 하나입니다.

---

## 3. 다음 포스팅

이번 포스팅에서는 `Scouter Client`를 설치하고, 실행해보면서 주요 창의 기능에 대해서 알아봤습니다.

다음 포스팅에서는 모니터링 대상 애플리케이션에 실제로 부하를 발생시키고, 운영 정보의 변화가 어떻게 나타나는지 확인해보겠습니다.

