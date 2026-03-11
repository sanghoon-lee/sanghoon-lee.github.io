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

그렇다면, 소프트웨어를 개발하는 조직의 `기술력`은 무엇으로 정의할 수 있을까요?

## 소프트웨어 개발 조직의 기술력

개발자의 개인적인 역량만으로 그 조직의 `기술력`을 설명하기는 어렵습니다. 아무리 뛰어난 개발자들이 모여 있더라도 그들의 **지식**과 **경험**이 공유되지 못한다면, 결국 개인의 자산으로만 남을 뿐입니다. 

조직의 `기술력`이 되려면 개인의 자산이 조직의 자산으로 축적되는 구조가 만들어져야 합니다.

예를 들어, 어떤 개발팀에서 다음과 같은 문제를 한 번 해결했다고 가정해보겠습니다.

* 외부 시스템 연동 시 장애 대응 방식
* 공통적인 인증 처리 방법
* 로그 표준화 방식
* 데이터 암호화 처리 방식
* 예외 처리 규칙

이러한 문제를 해결하는 과정에서 습득된 개인의 지식과 경험을 재사용 가능한 형태로 정리한다면,
조직 전체가 필요할 때 언제든지 활용할 수 있는 자산이 됩니다. 이러한 자산이 축적될수록 **“처음부터 다시 구현하는 일”**은 점점 줄어들면서 개발 생산성은 빠르게 증가할 것입니다. 이것이 바로 조직의 `기술력`입니다.

결국, `기술력`이 높은 조직일수록 개인보다는 조직에 축적된 **지식**과 **경험**을 기반으로 문제를 해결하게 됩니다.

## 조직의 기술력이 축적되는 방식: 공통 라이브러리

소프트웨어를 개발하다 보면 프로젝트마다 반복적으로 등장하는 문제들이 있습니다.

주로 다음과 같은 것들입니다.

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

이러한 문제를 해결하기 위한 기능들은 대부분 핵심 비즈니스 로직은 아닙니다. 그래서 매번 새롭게 설계하고, 다시 구현할 필요가 없습니다. 개발자의 시간만 낭비할 뿐입니다. 또한 동일한 기능임에도 프로젝트마다 구현 방식이 달라지면서 일관성 없는 시스템이 만들어질 가능성도 높아집니다.

그래서 이러한 기능들은 공통 라이브러리 형태로 정리해 두고, 새로운 프로젝트에서도 이미 검증되고 일관된 방식으로 문제를 해결해야 합니다. 이렇게 되면, 개발자는 핵심 비즈니스 로직의 구현에 더 많은 시간을 집중할 수 있게 됩니다. 핵심 비즈니스 로직이 아닌 공통 기능들은 공통 라이브러리 내부로 감춰지기 때문에, 코드는 보다 간결해지게 됩니다. 

당연히 개발 생산성은 높아지고, 유지보수는 또한 훨씬 용이해집니다. 

공통 라이브러리는 단순히 코드를 재사용하기 위한 도구가 아니라, 조직의 **지식**과 **경험**이 축적된 결과물입니다.

## 우리 조직의 현실

하지만 현재 제가 속한 조직에서는 공통 라이브러리라는 것이 사실상 존재하지 않습니다.

물론, 이전부터 공통 라이브러리 전략에 대해서 관심을 가지고 있었습니다. 적극적으로 추진하기 어려운 상황이었을 뿐입니다.

최근 조직 개편과 함께 새로운 시스템을 구축하는 프로젝트를 시작하게 되었습니다. 새로운 시도를 해볼 수 있는 작은 기회가 생긴 것입니다.

## 사설 라이브러리 저장소

Java 생태계에서는 Maven Central 같은 공개 저장소를 통해 수많은 라이브러리가 공유되고 있습니다.

하지만, 조직 내부에서 개발한 공통 라이브러리를 함부로 외부에 있는 공개 저장소에 배포할 수 없습니다. 그래서 **사설 라이브러리 저장소(Private Repository)**가 필요한 것입니다.

`Nexus`는 사설 라이브러리 저장소를 운영할 수 있는 대표적인 도구입니다. 

```
서비스 A ─┐
서비스 B ─┼── 공통 라이브러리 ── Nexus
서비스 C ─┘
```

## 테스트 환경

공통 라이브러리 전략을 추진하려면 사설 라이브러리 저장소를 구축하고, 운영해야 합니다. 그래서 개인 노트북에 사설 라이브러리 저장소를 구축해보고, 테스트로 만든 간단한 라이브러리를 배포해봤습니다. 

노트북에 VMware로 가상 머신을 생성하고, Ubuntu Linux와 Nexus Repository를 설치했습니다. 테스트 환경의 구성은 다음과 같습니다.

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

`Nexus`는 `Docker`를 사용해서 설치했습니다. 먼저, `Docker`에서 사용할 `Nexus` 이미지 파일을 로컬로 가져왔습니다.

```bash
$sudo docker pull sonatype/nexus3:latest

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

혹시 몰라서 로컬로 `Nexus` 이미지 파일을 잘 가져왔는지 다시 한번 확인했습니다. 

```bash
$sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

이미지 파일이 존재하는 것을 확인하고, `Docker` 컨테이너를 실행했습니다.

```bash
sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

컨테이너가 정상적으로 실행되면 브라우저에서 아래 주소로 `Nexus`에 접속할 수 있습니다.

```
http://10.68.65.187:8081
```

아래 그림처럼 접속이 되면 정상적으로 동작하는 것입니다. 최초 접속이면, admin의 초기 비밀번호가 저장된 파일의 경로가 나타납니다. 해당 파일을 읽고 초기 비밀번호를 확인하고, 로그인하면 됩니다.

<img class="main-image" src="/assets/images/nexus1.png" alt="Nexus Repository">

로그인에 성공하면, Welcome 메시지와 함께 비밀번호 변경 절차를 반드시 수행해야 합니다.

<img class="main-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

maven-releases와 maven-snapshots 두 개의 Repository를 사용할 예정이었는데, 다행히 필요한 Repository는 기본적으로 생성되어 있었습니다.
<img class="main-image" src="/assets/images/nexus4.png" alt="Nexus Repository">


