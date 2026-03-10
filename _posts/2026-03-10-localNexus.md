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

소프트웨어 개발 조직의 `기술력`은 어떻게 측정할 수 있을까요?**

물론, 다양한 측정 지표들이 사용될 수 있을 것입니다.

각 조직은 영위하고 있는 사업의 구조나 방식 등에 따라 고유한 비즈니스 문제가 존재하기 마련입니다. 이러한 문제를 해결할 수 있는 방법을 공개된 외부 라이브러리에서 찾는 것은 거의 불가능합니다. 따라서 대부분 조직 내부의 `지식`과 `경험`을 최대한 활용해서 문제를 해결하는데 필요한 코드를 직접 구현해야 합니다. 

또한, 개발을 하다 보면 비슷한 문제를 자주 만나기도 합니다. 예를 들면 다음과 같은 것들입니다.

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

과거에 이미 해결된 문제를 또다시 새롭게 해결하려고 것은 생산성 측면에서 매우 비효율적입니다. 

그래서 대부분의 조직에서는 공통적으로 활용할 수 있는 `지식`과 `경험`을 코드로 축적해서 라이브러리 형태로 배포하는 방법을 고민하게 됩니다. 이렇게 하면, 코드를 재사용할 수 있어서 개발자는 새로운 문제의 해결에 보다 집중할 수 있습니다. 그리고 반복적인 코드는 라이브러리 내부로 감춰지기 때문에, 코드의 가독성을 높이는데도 도움이 됩니다. 결국 개발 생산성은 높아지고, 유지보수는 용이해지게 됩니다.

그래서 저는 **"얼마나 많은 지식과 경험이 재사용 가능한 코드로 축적되어 있는가?"**가 그 조직의 `기술력`을 나타내는 가장 중요한 지표라고 생각합니다.

## 우리 조직의 현실

하지만 현재 제가 근무하고 있는 개발 조직에는 공통 라이브러리라는 것이 사실상 존재하지 않습니다.

그래서 여러 서비스에 유사한 코드들이 다수 존재합니다. 하나의 서비스 내에서도 동일한 기능을 개발자마다 각각 구현해서 사용한 경우도 쉽게 찾아볼 수 있습니다. 조직이 보유하고 있는 `지식`과 `경험`이 코드로 축적되지 못한 결과입니다.

생산성 저하 뿐만 아니라, `지식`과 `경험`이 조직의 자산으로 축적되지 못하는 구조라는 점에서 큰 문제라고 생각합니다. 

## 새로운 시작

이전부터 공통 라이브러리 전략에 대해서 관심을 가지고 있었습니다. 다만, 기존 시스템을 이어받아 유지보수를 해야 하는 입장이었기 때문에 적극적인 시도를 하기 어려웠던 것 뿐입니다.

최근 조직 개편과 함께 새로운 시스템의 구축을 준비하는 역할을 맡게 되었습니다. 적극적으로 새로운 시도를 해볼 수 있는 기회가 생긴 것입니다.

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

이번 실습은 윈도우가 설치된 노트북에서 진행했습니다. VMware로 가상 머신을 생성하고, Ubuntu Linux와 Nexus Repository를 설치했습니다. 

정리하자면, 실습 환경의 구성은 다음과 같습니다.

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

Nexus는 Docker를 사용해서 설치했습니다. 먼저, Docker에서 사용할 Nexus 이미지 파일을 로컬로 가져오는 것부터 시작했습니다.

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

그리고, Nexus 이미지를 정상적으로 가져왔는지도 확인했습니다.

```bash
$sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

이후 Nexus Repository 구성을 위해서 Docker 컨테이너를 실행했습니다.

```bash
sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

컨테이너가 정상적으로 실행되면 브라우저에서 아래 주소로 Nexus에 접속할 수 있습니다.

```
http://10.68.65.187:8081
```

아래 그림처럼 접속이 된다면 Nexus는 정상적으로 동작하는 것입니다. admin 계정의 최초 비밀번호는
/nexus-data

<img class="main-image" src="/assets/images/nexus1.png" alt="Nexus Repository">

처음 접속하면

<img class="main-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

<img class="main-image" src="/assets/images/nexus4.png" alt="Nexus Repository">


