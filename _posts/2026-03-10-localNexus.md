---
layout: post
title: "[학습기록] Nexus로 사설 라이브러리 저장소 구축해보기"
date: 2026-03-10
categories: 학습기록
---

<style>
.main-image {
  width: 100%;
  max-width: 900px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

일반적으로 소프트웨어 개발은 다음과 같은 과정을 거쳐 진행됩니다.

* 문제의 정의
* 문제의 구조화와 세분화
* 해결책 도출
* 구현과 검증

소프트웨어 개발은 단순히 **코드를 작성하는 작업**을 넘어 **논리적 사고**를 바탕으로 비즈니스 문제를 해결하는 행위로 설명할 수 있습니다.

그렇다면, 소프트웨어를 개발에서 `기술력`은 어떤 의미일까요?

## 소프트웨어 개발 조직의 기술력

소프트웨어 개발 조직의 `기술력`은 개발자들이 문제를 해결하는 과정에서 자연스럽게 습득한 **지식**과 **경험**이 조직 전체에 공유되고, 지속적으로 축적될 때 비로소 만들어집니다.

`기술력`이 부족한 조직은 특정 개발자의 개인적인 역량에 의존할 수 밖에 없습니다. 하지만, 의존도가 높은 개발자가 떠나게 되면 어떻게 될까요? 그 동안 쉽게 해결할 수 있었던 문제들도 처음부터 다시 고민하게 될 것입니다. 문제 해결의 방식이 조직에 축적되지 못하고, 개인에게만 머물러 있었기 때문입니다. 

그렇지만, 높은 `기술력`을 보유한 조직은 의존도가 높은 개발자가 떠나더라도 크게 문제가 되지 않을 것입니다. 과거부터 축적된 **지식**과 **경험**을 기반으로 문제를 해결할 수 있기 때문입니다.

결국, 소프트웨어 개발 조직의 `기술력`은 축적된 **지식**과 **경험**의 크기와 깊이에서 결정된다고 볼 수 있습니다.

## 기술력을 활용하는 방식: 공통 라이브러리

그렇다면 `기술력`은 어떤 방식으로 활용할 수 있을까요?

대표적인 방법 중 하나가 바로 **공통 라이브러리**입니다. 

소프트웨어를 개발하다 보면 프로젝트마다 반복적으로 등장하는 문제들이 있습니다. 주로 다음과 같은 것들입니다.

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

이러한 문제를 해결하면서 조직에 축적된 **지식**과 **경험**을 코드 형태로 정리한 결과물이 바로 **공통 라이브러리**입니다. 

**공통 라이브러리**를 이용하면 과거에 해결했던 문제를 위해 새롭게 설계하고, 다시 코드를 작성할 필요가 없습니다. 새로운 프로젝트에서도 이미 검증되고, 일관된 방식으로 쉽고 빠르게 문제를 해결할 수 있습니다. 개발자는 핵심 비즈니스 로직의 구현에 더 많은 시간을 집중할 수 있게 됩니다. 또한 반복되는 코드가 **공통 라이브러리** 내부에 감춰지기 때문에, 전체적으로 코드 가독성이 좋아집니다. 

결과적으로 소프트웨어 개발 생산성은 높아지고, 유지보수는 또한 훨씬 용이해지게 됩니다. 

## 우리 조직의 현실

안타깝게도 현재 제가 근무하고 있는 개발 조직에서는 **공통 라이브러리**라고 할 만한 것이 사실상 존재하지 않습니다. 더 이상 미루지 않고, 지금부터라도 **공통 라이브러리** 전략을 적극적으로 추진해보고 싶었습니다.

## 사설 라이브러리 저장소

조직 내부에서 개발한 **공통 라이브러리**를 함부로 `Maven Central`과 같은 공개 저장소에 배포할 수 없습니다. 그래서 `사설 라이브러리 저장소(Private Repository)`가 반드시 필요합니다.

`Nexus`는 `사설 라이브러리 저장소`를 운영할 수 있는 대표적인 도구입니다. 

```
서비스 A ─┐
서비스 B ─┼── 공통 라이브러리 ── Nexus
서비스 C ─┘
```

우선 개인 노트북에 `사설 라이브러리 저장소`를 구축하고, 간단한 라이브러리를 만들어서 배포하는 테스트를 해봤습니다.

## 테스트 환경

테스트 환경을 만들기 위해 노트북에 VMware로 가상머신을 생성하고, Ubuntu Linux와 `Nexus`를 설치했습니다. 

```
+-----------------------------+
| Windows PC                  |
| IP : 10.68.65.226           |
|                             |
| VMware Host                 |
+-------------+---------------+
              |
              | Virtual Network
              |
+-------------v---------------+
| Ubuntu Linux VM             |
| IP : 10.68.65.187           |
|                             |
| Nexus Repository            |
| Port : 8081                 |
| http://10.68.65.187:8081    |
+-----------------------------+
```

## Nexus Repository 설치

`Docker`를 사용해서 `Nexus`를 설치하려고 이미지 파일을 가상머신(Ubuntu Linux VM)으로 가져왔습니다.

```bash
$ sudo docker pull sonatype/nexus3:latest

latest: Pulling from sonatype/nexus3
575f766103b9: Pull complete
4638e3415987: Pull complete
87650cc837b9: Pull complete
4591fb595303: Pull complete
3cd2f89bf989: Pull complete
1cb34de82e1f: Pull complete
db016631b702: Download complete
Digest: sha256:cb94c17229a34d20365345dfa28552ee462cf79c77dd2fadbd98422e5a439bc9
Status: Downloaded newer image for sonatype/nexus3:latest
docker.io/sonatype/nexus3:latest
```

이미지 파일을 정상적으로 가져왔는지 확인도 했습니다. 

```bash
$ sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

`Nexus` 데이터를 저장할 폴더(/nexus-data)를 가상머신에 생성하고, `Docker` 컨테이너와 연결시켰습니다.  

```bash
$ sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

## Nexus 접속

컨테이너가 정상적으로 실행되고 있다면, 브라우저에서 아래 주소로 `Nexus`에 접속할 수 있습니다.

```
http://10.68.65.187:8081
```

최초 접속이면, admin의 초기 비밀번호가 저장된 파일의 경로가 나타납니다. 해당 파일을 읽고 초기 비밀번호를 확인하고, 로그인하면 됩니다.

<img class="main-image" src="/assets/images/nexus1.png" alt="Nexus Repository">

로그인에 성공하면, Welcome 메시지와 함께 비밀번호 변경 절차를 반드시 수행해야 합니다.

<img class="main-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

maven-releases와 maven-snapshots 두 개의 Repository를 사용할 예정이었는데, 다행히 필요한 Repository는 기본적으로 생성되어 있었습니다.
<img class="main-image" src="/assets/images/nexus4.png" alt="Nexus Repository">


