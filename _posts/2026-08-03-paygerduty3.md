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

모니터링 대상 애플리케이션에 `Scouter Agent` 연동이 완료되었습니다. 이제 `Scouter Server`에는 애플리케이션의 운영 정보가 실시간으로 수집됩니다.

이번에는 `Scouter Client`를 설치하고, 수집된 운영 정보를 확인하는 방법을 살펴보겠습니다.

---

## 1. Scouter Client 준비

`Scouter Client`를 통해 `Scouter Server`에 접속해보겠습니다.

먼저 `Scouter Client`를 다운로드하고 실행합니다.

---

### 1.1. Scouter Client 다운로드

이전 포스팅과 동일하게 `Scouter` 프로젝트의 공식 GitHub 저장소의 `release` 브랜치에서 `2.17.1` 태그를 선택했습니다.

* [https://github.com/scouter-project/scouter](https://github.com/scouter-project/scouter)

<img class="sub-image" src="/assets/images/scouter-release-tags.jpg" alt="Scouter 프로젝트 Release 브랜치의 Tag 목록">

그리고 페이지 하단의 **Assets**에서 `scouter.client.product-win32.win32.x86_64.zip` 파일을 다운로드했습니다.

<img class="sub-image" src="/assets/images/scouter-assets2.jpg" alt="Scouter 2.17.1 배포 버전의 Assets 목록">

운영체제에 따라 제공되는 Client 파일이 다르므로 Windows가 아니라면 자신의 운영체제에 맞는 패키지를 다운로드하면 됩니다.

---

### 1.2. 압축 해제

다운로드한 파일의 압축을 해제하면 `scouter.client` 디렉터리가 생성됩니다.

실행과 관리가 편리하도록 `scouter.client` 디렉터리를 `C:\` 아래로 이동했습니다.

<img class="sub-image" src="/assets/images/scouter.client.png" alt="scouter.client 디렉터리">

---

### 1.3. Scouter Client 실행

그리고 `scouter.client` 디렉터리에 있는 `scouter.exe` 파일을 실행했습니다. 하지만 기대와 다르게 Windows의 PC 보호 경고창이 나타났습니다. `Scouter Client`가 출처를 알 수 없는 애플리케이션으로 분류되어 실행이 차단되는 것 같았습니다. 

하지만 경고 메시지 하단의 추가정보를 클릭하면 실행 버튼을 활성화할 수 있습니다.

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

**주의**

> 실행 버튼을 클릭하기 전에 공식 GitHub 저장소에서 다운로드한 파일인지 반드시 확인하는 것이 안전합니다.

실행 버튼을 클릭하자 `Scouter Client`의 로그인 화면이 나타났습니다.

<img class="main-image" src="/assets/images/scouter.login.png" alt="scouter 서버 정보 입력창">

`Server Address`에는 `Scouter Server`가 실행되고 있는 주소인 192.168.56.13:6100을 입력했습니다. 또한 `ID`와 `Password`에는 기본 계정인 `admin` / `admin`을 입력했습니다. 그리고 OK 버튼을 클릭해서 정상적으로 로그인할 수 있었습니다.

---

## 2. 운영 정보 확인

로그인이 완료되면 Scouter Client에서 서버 전체의 상태와 선택된 애플리케이션의 상태를 함께 확인할 수 있습니다.

<img class="sub-image" src="/assets/images/scouter1.jpg" alt="Scouter Client 기본 모니터링 화면">

기본 화면에는 Linux 서버의 CPU, 메모리, 디스크, 네트워크 등 서버 전체의 자원 사용 현황이 표시됩니다.

또한 Heap Memory, Active Service, XLog와 같은 애플리케이션 수준의 정보도 함께 표시되며, 이러한 정보는 Object Tree에서 선택된 애플리케이션을 기준으로 제공됩니다.

---

### 2.1. Object Tree

좌측의 Object Tree에서 `Scouter Server`에 등록되어 있는 모니터링 대상 애플리케이션의 목록을 확인하고 선택할 수 있습니다.

<img class="main-image" src="/assets/images/objecttree.jpg" alt="Scouter Object Tree">

위 화면을 보면 많은 애플리케이션이 표시되어 있습니다. 하지만 현재 실행되고 있는 애플리케이션은 다음의 두 개뿐입니다.  

* f7ddc5a582f1 → simple-api-1
* f134b16023c7 → simple-api-2

나머지 항목은 토이프로젝트를 진행하면서 생성했던 애플리케이션 컨테이너들입니다. 현재는 실행이 종료되었지만, Scouter는 이전에 수집했던 Object 정보를 유지하기 때문에 Object Tree에 계속 남아 있습니다.

앞부분의 `f7ddc5a582f1`, `f134b16023c7`은 Docker가 컨테이너에 부여한 **hostname**입니다. 뒤에 표시되는 `simple-api-1`, `simple-api-2`는 Docker Compose의 `JAVA_TOOL_OPTIONS`를 통해 `-Dobj_name`으로 지정한 이름입니다. 동일한 호스트에서 여러 애플리케이션 인스턴스를 실행하더라도 `obj_name`을 통해 각 인스턴스를 쉽게 구분할 수 있습니다.

---

### 2.2. CPU - Linux(서버의 CPU 사용률)

CPU - Linux 창에서는 `Scouter Agent`가 실행 중인 Linux 서버의 CPU 사용률을 실시간으로 확인할 수 있습니다.

<img class="main-image" src="/assets/images/cpu.jpg" alt="CPU - Linux">

이 그래프는 특정 애플리케이션이 아니라 서버 전체의 CPU 사용률을 나타냅니다. 따라서 Object Tree에서 `simple-api-1`이나 `simple-api-2`를 선택하지 않아도 항상 표시됩니다.

CPU 사용률이 지속적으로 높게 유지된다면 서버 전체에 부하가 발생하고 있는 상황일 수 있습니다. 반대로 CPU 사용률은 낮은데 특정 애플리케이션의 응답 시간이 느리다면 애플리케이션 내부의 병목이나 외부 시스템 호출을 의심해볼 수 있습니다.

현재는 두 개의 애플리케이션이 동일한 Linux 서버에서 실행되고 있기 때문에 CPU - Linux 창은 하나만 표시됩니다.

---

### 2.3. Heap Memory(JVM Heap 사용량)

Heap Memory 창에서는 Object Tree에서 선택된 애플리케이션의 JVM Heap 사용량을 실시간으로 확인할 수 있습니다.

로그인 직후에는 `Scouter Client`가 기본적으로 하나의 애플리케이션을 선택하기 때문에 별도로 선택하지 않아도 Heap 사용량이 표시됩니다. 다른 애플리케이션을 선택하면 해당 애플리케이션의 Heap 사용량으로 화면이 변경됩니다.

<img class="main-image" src="/assets/images/heap-memory.jpg" alt="Heap Memory">

그래프는 시간이 지남에 따라 Heap 사용량이 어떻게 변화하는지를 보여줍니다. 일반적으로 요청을 처리하면서 점차 증가하다가, Garbage Collection(GC) 이 수행되면 사용하지 않는 객체가 정리되면서 다시 감소하는 모습을 확인할 수 있습니다.

Heap 사용량이 지속적으로 증가하고 GC 이후에도 거의 줄어들지 않는다면 메모리 누수(Memory Leak)를 의심해볼 수 있습니다. 또한 Heap 사용량이 최대치에 자주 도달한다면 JVM Heap 크기를 조정하거나 애플리케이션의 메모리 사용 패턴을 분석할 필요가 있습니다.

---

### 2.4. Active Service(현재 처리 중인 요청)

Active Service 창에서는 현재 처리 중인 요청(Active Service)의 개수를 실시간으로 확인할 수 있습니다.

<img class="main-image" src="/assets/images/active-service.jpg" alt="Active Service">

그래프에는 시간에 따른 Active Service의 변화가 표시됩니다. 위 화면은 애플리케이션으로 유입되는 요청이 없는 상태에서 캡처한 화면이므로 Active Service도 0으로 표시됩니다.

반대로 많은 요청이 동시에 유입되거나 일부 요청의 처리 시간이 길어지면 Active Service가 빠르게 증가하는 모습을 확인할 수 있습니다.

만약 Active Service가 지속적으로 높은 상태를 유지한다면 애플리케이션이 요청을 제때 처리하지 못하고 있을 가능성이 있습니다. 이 경우에는 데이터베이스 조회 지연, 외부 API 호출 지연, 스레드 부족, 락 경합 등 요청 처리가 지연되는 원인이 없는지 함께 확인해 보는 것이 좋습니다.

---

### 2.5. XLog(요청 처리 이력)

XLog는 애플리케이션에서 처리된 요청의 실행 이력을 확인할 수 있는 기능입니다.

<img class="main-image" src="/assets/images/xlog.jpg" alt="XLog">

앞에서 살펴본 Active Service가 **현재 처리 중인 요청**을 보여준다면, XLog는 **처리가 완료된 요청의 실행 결과**를 보여줍니다.

각 요청은 하나의 점(Point)으로 표시되며, 점의 위치를 통해 요청이 발생한 시각과 처리 시간을 직관적으로 확인할 수 있습니다. 점이 위쪽에 위치할수록 요청 처리 시간이 길다는 의미이므로, 응답이 느린 요청을 한눈에 찾을 수 있습니다. 특정 요청을 더블 클릭하면 해당 요청의 상세 실행 정보도 확인할 수 있습니다.

응답 시간이 비정상적으로 오래 걸린 요청이나 오류가 발생한 요청을 빠르게 찾아 분석할 수 있기 때문에, 장애 원인을 추적할 때 가장 많이 활용되는 화면 중 하나입니다.

---

## 3. 마무리

`Scouter Client`를 설치하고 기본 모니터링 화면을 통해 운영 정보를 확인하는 방법을 살펴봤습니다.

지금까지는 모니터링 화면의 구성과 각 지표의 의미를 이해하는 과정이었다면, 다음 포스팅에서는 실제 부하를 발생시키면서 운영 정보를 활용해 장애를 분석하는 과정을 살펴보겠습니다.

