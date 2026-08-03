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

이번에는 `Scouter Client`를 통해 CPU 사용률, Heap 메모리, Active Service, XLog 등 다양한 운영 정보를 실시간으로 확인해보겠습니다.

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

하지만 경고 메시지 하단의 추가정보를 클릭하자 실행 버튼이 활성화되었습니다.

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

공식 GitHub 저장소에서 다운로드한 파일임을 확인한 뒤 실행 버튼을 클릭했습니다. 그러자 `Scouter Client`의 로그인 화면이 나타났습니다.

<img class="sub-image" src="/assets/images/scouter.login.png" alt="scouter 서버 정보 입력창">

`Server Address`에는 `Scouter Server`가 실행되고 있는 주소인 192.168.56.13:6100을 입력했습니다. 또한 ID와 Password에는 기본 계정인 `admin` / `admin`을 입력한 뒤 로그인했습니다.

로그인이 완료되자 `Scouter Client`의 기본 모니터링 화면이 표시되었습니다.

<img class="sub-image" src="/assets/images/scouter1.jpg" alt="Scouter Client 기본 모니터링 화면">

---

## 2. 모니터링

작성중

