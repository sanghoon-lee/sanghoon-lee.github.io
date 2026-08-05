---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (3) Scouter Client 설치와 운영 정보 모니터링"
date: 2026-08-03
series: paygerduty
description: 
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

모니터링 대상 애플리케이션에 `Scouter Agent` 연동이 완료되었습니다. 이제 `Scouter Server`에는 애플리케이션의 운영 정보가 실시간으로 수집되고 있습니다.

이번에는 `Scouter Client`를 통해 CPU 및 Heap 메모리 사용률, Active Service, XLog 등 다양한 운영 정보를 실시간으로 확인해보겠습니다.

---

## 1. Scouter Client 준비

`Scouter Client`를 사용하기 위해 필요한 파일을 다운로드해야 합니다.

---

### 1.1. Scouter Client 다운로드

이전 포스팅과 동일하게 `Scouter` 프로젝트의 공식 GitHub 저장소의 `release` 브랜치에서 `2.17.1` 태그를 선택했습니다.

* [https://github.com/scouter-project/scouter](https://github.com/scouter-project/scouter)

<img class="sub-image" src="/assets/images/scouter-release-tags.jpg" alt="Scouter 프로젝트 Release 브랜치의 Tag 목록">

이번에는 페이지 하단의 **Assets**에서 `scouter.client.product-win32.win32.x86_64.zip` 파일을 다운로드했습니다.

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

로그인이 완료되면 `Scouter Client`에는 서버 전체의 상태를 보여주는 정보와 특정 애플리케이션의 상태를 보여주는 정보가 함께 표시됩니다.

<img class="sub-image" src="/assets/images/scouter1.jpg" alt="Scouter Client 기본 모니터링 화면">

기본적으로 화면 하단에는 Linux 서버의 CPU, 메모리, 디스크, 네트워크 등 서버 전체의 자원 사용 현황이 표시됩니다. 이러한 정보는 특정 애플리케이션을 선택하지 않아도 항상 확인할 수 있습니다.

반면 Heap 메모리, Active Service, XLog와 같은 애플리케이션 수준의 운영 정보는 Object Tree에서 모니터링 대상을 선택해야 확인할 수 있습니다.

---

### 2.1. Object Tree

좌측의 Object Tree에서 `Scouter Server`에 등록되어 있는 모니터링 대상 애플리케이션의 목록을 확인하고 선택할 수 있습니다.

<img class="main-image" src="/assets/images/objecttree.jpg" alt="Scouter Object Tree">

위 화면을 보면 많은 애플리케이션이 표시되어 있습니다. 하지만 현재 실행되고 있는 애플리케이션은 다음의 두 개뿐입니다.  

* f7ddc5a582f1 → simple-api-1
* f134b16023c7 → simple-api-2

나머지 항목은 토이프로젝트를 진행하면서 생성했던 애플리케이션 컨테이너들입니다. 현재는 실행이 종료되었지만, Scouter는 이전에 수집했던 Object 정보를 유지하기 때문에 Object Tree에 계속 표시됩니다.

앞부분의 `f7ddc5a582f1`, `f134b16023c7`은 Docker가 컨테이너에 부여한 **hostname**입니다. 뒤에 표시되는 `simple-api-1`, `simple-api-2`는 Docker Compose의 `JAVA_TOOL_OPTIONS`를 통해 `-Dobj_name`으로 지정한 이름입니다. 동일한 호스트에서 여러 애플리케이션 인스턴스를 실행하더라도 `obj_name`을 통해 각각의 애플리케이션을 쉽게 구분할 수 있습니다.

---

작성중
