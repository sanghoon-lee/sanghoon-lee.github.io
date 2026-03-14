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

그래서 소프트웨어 개발은 **코드를 작성하는 작업**을 넘어 **논리적 사고**를 바탕으로 비즈니스 문제를 해결하는 행위로 설명할 수 있습니다.

## 소프트웨어 개발 조직의 기술력

개발자들은 소프트웨어를 개발하는 과정에서 수 많은 문제를 마주하게 됩니다. 그리고 그 문제를 해결하는 과정에서 자연스럽게 **지식**과 **경험**을 습득하게 됩니다. 

하지만, **지식**과 **경험**이 팀의 지적 자산으로 축적되지 못하면 어떻게 될까요? 시간이 흐를수록 점점 개인적인 역량이 뛰어난 특정 개발자에 대한 의존도가 커질 수밖에 없을 것입니다. 문제 해결의 방식이 팀이 아닌 개인에게만 머무르는 구조이기 때문입니다. 

혹시라도 의존도가 높은 개발자가 팀을 떠나기라도 한다면, 어떻게 될까요? 그 개발자가 해결했던 수많은 문제의 해결 방식들도 그대로 사라지게 될 것입니다. 

그래서 개발자들의 **지식**과 **경험**을 체계적으로 팀에 축적하는 것은 매우 중요합니다. 

팀에서 활용할 수 있도록 축적된 **지식**과 **경험**의 크기와 깊이가 결국 소프트웨어 개발 조직의 `기술력`이라고 생각합니다. 

## 기술력을 활용하는 방식: 공통 라이브러리

그렇다면 소프트웨어 개발 조직의 `기술력`은 어떤 방식으로 활용할 수 있을까요?

대표적인 방식 중 하나가 바로 **공통 라이브러리**입니다. 

소프트웨어를 개발하다 보면 프로젝트마다 반복적으로 등장하는 문제들이 있습니다. 주로 다음과 같은 것들입니다.

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

이러한 문제들의 해결 방식을 코드 형태로 정리해서 재사용할 수 있도록 만들어 낸 결과물이 바로 **공통 라이브러리**입니다. 

과거에 해결했던 문제는 **공통 라이브러리**를 활용하면, 새로운 개발 프로젝트에서 코드를 다시 작성할 필요가 없습니다. 이미 검증되고, 일관된 방식으로 쉽고 빠르게 문제를 해결할 수 있습니다. 

반복되는 코드는 **공통 라이브러리** 내부에 감춰지기 때문에, 전체적인 코드는 간결해지고 복잡도를 낮출 수 있습니다. 또한, 개발자는 핵심 비즈니스 로직의 구현에 더 많은 시간을 집중할 수 있습니다. 결과적으로 소프트웨어 개발 품질과 생산성은 높아지고, 유지보수의 난이도는 낮아지게 될 것입니다.

## 사설 라이브러리 저장소(Private Repository)의 필요성

사실 오래전부터 **공통 라이브러리**를 적극적으로 이용해서 개발 생산성을 향상시키는 방법을 고민하고 있었습니다. 

**공통 라이브러리**를 활용하려면 `사설 라이브러리 저장소(Private Repository)`가 반드시 필요합니다.내부에서 개발한 코드를 함부로 `Maven Central`과 같은 공개 저장소에 배포할 수 없기 때문입니다. 

`Nexus`는 `사설 라이브러리 저장소`를 운영하기 위해 활용할 수 있는 대표적인 도구입니다. 

```
서비스 A ─┐
서비스 B ─┼── 공통 라이브러리 ── Nexus
서비스 C ─┘
```

## 테스트 환경

개인 노트북에 `사설 라이브러리 저장소`를 구축하고, 간단한 라이브러리를 만들어서 배포해보려고 테스트 환경을 만들었습니다. 

`사설 라이브러리 저장소`는 노트북에 VMware로 생성한 가상머신에 `Nexus`를 설치해서 구축했습니다.

테스트 환경은 다음과 같습니다.
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

## Nexus 설치

Docker를 이용해서 `Nexus`를 설치했습니다. 설치 절차는 다음과 같습니다.

**이미지 파일 가져오기**
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

**이미지 파일 확인**
```bash
$ sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

**컨테이너 실행**
```bash
$ sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

`Nexus Repositoy`의 데이터를 저장할 폴더(**/nexus-data**)를 가상머신에 생성하고, 컨테이너와 연결시켰습니다.  

## Nexus 접속

컨테이너가 정상적으로 실행되고 있으면 브라우저에서 아래 주소로 `Nexus`에 접속할 수 있습니다. 

```
http://10.68.65.187:8081
```

최초 접속인 경우, 로그인 창에 admin계정의 초기 비밀번호가 저장된 파일 경로가 나타납니다. 해당 경로에서 비밀번호를 확인하고, 로그인하면 됩니다.

**최초 접속시 로그인 창**
<img class="main-image" src="/assets/images/nexus1.png" alt="Nexus Repository">

로그인에 성공하면, 아래 그림처럼 Welcome 메시지가 나타납니다. 

**Welcome 메시지**
<img class="main-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

초기 비밀번호로 로그인하면 비밀번호를 변경해야 합니다.

**비밀번호 변경**
<img class="main-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

별도로 설정하지 않아도 기본적으로 필요한 Repository는 이미 생성되어 있었습니다.

**Nexus Repository 목록**
<img class="main-image" src="/assets/images/nexus4.png" alt="Nexus Repository">

## SimpleLib

테스트를 위해 아래처럼 간단한 코드를 작성했습니다. **라이브러리**로 만들어 다른 프로젝트에서 사용할 수 있도록 jar파일로 빌드하고, `Nexus Repositoty`에 배포해보도록 하겠습니다.

```java
package sanghoon.study.lib;

public class SimpleLib {
    public int sum(int x,int y){
        return x+y;
    }
}
```