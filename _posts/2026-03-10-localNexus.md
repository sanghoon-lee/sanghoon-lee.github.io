---
layout: post
title: "[학습기록] 사설 라이브러리 저장소 구축을 위해 Nexus 설치"
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

소프트웨어 개발 조직의 `“기술력”`을 어떻게 측정할 수 있을까요?**

다양한 측정 지표들을 사용할 수 있을 것입니다. 하지만, 저는 그 중에서 **"얼마나 공통 코드가 축적되어 재사용되고 있는가?"**가 가장 기본적인 지표가 된다고 생각합니다.

개발을 하다 보면 비슷한 문제를 반복해서 만나게 됩니다. 예를 들면 다음과 같은 것들입니다.

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

이러한 문제들은 특정 서비스에만 존재하는 것이 아닙니다. 대부분의 서비스에서 반복적으로 등장합니다.

동일한 문제를 매번 해결하는 것은 생산성 측면에서 매우 비효율적입니다. 그래서 대부분의 조직에서는 공통적이고 반복되는 기능들은 라이브러리 형태로 만들어 재사용하게 됩니다. 핵심 비즈니스 로직의 구현에 집중하고, 반복적인 코드는 숨겨서 코드 가독성을 높일 수 있기 때문입니다. 

또한 조직마다 비즈니스 도메인과 관련된 고유한 로직이 존재하기도 합니다.
이러한 로직은 공개된 외부 라이브러리에서는 찾을 수 없기 때문에, 조직 내부에서 직접 구현하고 관리해야 합니다.

공통되고 반복되는 기능과 조직 고유의 비즈니스 로직을 라이브러리로 정리하고, 이를 여러 서비스에서 재사용할 수 있도록 만드는 것이 기술력을 축적하는 한 가지 방법이라고 생각합니다.

## 우리 조직의 현실

하지만 현재 제가 근무하고 있는 개발 조직에는 공통 라이브러리라는 개념이 사실상 존재하지 않습니다.

유사한 코드가 여러 서비스에 반복되어 존재합니다. 하나의 서비스 안에서도 동일한 기능을 개발자마다 각각 구현해서 사용하는 경우도 있습니다. 공통적인 기능들이나 회사 고유의 비즈니스 로직이 코드 형태로 축적되지 못하면서, 동일한 문제를 여러 번 반복해서 해결해야 하는 상황이 발생하는 것입니다.

개발 과정에서 얻은 경험이 조직의 자산으로 축적될 수 없는 구조라는 점에서 큰 문제라고 생각합니다.

## 새로운 시작

최근 조직 개편과 함께 새로운 시스템의 구축을 준비하고 있습니다. 기존 시스템을 그대로 이어가는 것이 아니라, 여러 가지 기술적인 요소를 새롭게 설계할 수 있는 기회가 생긴 것입니다.

이 시점에 다음과 같은 질문을 하게 되었습니다.

> 우리가 만든 코드를 어떻게 조직의 자산으로 축적할 수 있을까?

그 답으로 찾은 것이 바로 공통 라이브러리 전략입니다. 

## 사설 라이브러리 저장소

Java 생태계에서는 Maven Central 같은 공개 저장소를 통해 수많은 라이브러리가 공유되고 있습니다.

하지만, 조직 내부에서 개발한 공통 라이브러리를 함부로 외부 공개 저장소에 배포할 수 없습니다. 그래서 사설 라이브러리 저장소(Private Repository)가 필요합니다.

사설 라이브러리 저장소를 운영할 수 있는 대표적인 도구가 바로 Nexus입니다. 

이번 글에서는 그 과정의 첫 단계로
사설 라이브러리 저장소를 구축해보는 과정을 정리해보려고 합니다.

```
서비스 A ─┐
서비스 B ─┼── 공통 라이브러리 ── Nexus
서비스 C ─┘
```

## 테스트 환경

이번 실습은 윈도우가 설치된 노트북에서 진행했습니다.
VMware로 가상 머신을 생성한 뒤, Ubuntu Linux를 설치하고 그 위에 Nexus Repository를 구성했습니다.

구조는 다음과 같습니다.

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

## 넥서스(Nexus) 설치

이번 실습에서는 Docker를 이용해 Nexus Repository를 설치했습니다.

먼저 Nexus 이미지를 로컬로 가져왔습니다.

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

이미지가 정상적으로 내려받아졌는지 확인했습니다.

```bash
$sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

이후 Nexus 컨테이너를 실행했습니다.

도커 컨테이너 실행
```bash
sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

컨테이너가 정상적으로 실행되면 브라우저에서 아래 주소로 Nexus에 접속할 수 있습니다.

```
http://10.68.65.187:8081
```
<img class="main-image" src="/assets/images/nexus1.png" alt="Nexus Repository">


<img class="main-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus4.png" alt="Nexus Repository">


