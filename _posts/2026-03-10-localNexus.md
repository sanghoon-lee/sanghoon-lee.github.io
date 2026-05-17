---
layout: post
title: "[학습기록] Nexus로 내부 라이브러리 저장소 구축해보기"
date: 2026-03-10
description: Nexus로 라이브러리 저장소를 구축해본 과정을 정리해서 작성한 포스팅입니다.
image: /assets/images/nexus2.jpg
categories: 학습기록
---

일반적으로 소프트웨어 개발은 다음과 같은 과정을 거쳐 진행됩니다.

* 문제의 정의
* 문제의 구조화와 세분화
* 해결책 도출
* 구현과 검증

그래서 소프트웨어 개발은 **코드를 작성하는 작업**을 넘어 **논리적 사고**를 바탕으로 비즈니스 문제를 해결하는 행위로 설명할 수 있습니다.

---

## 1. 개발자의 지식과 경험

개발자들은 소프트웨어를 개발하는 과정에서 수 많은 문제와 마주하게 됩니다. 그리고 문제를 해결하는 과정에서 자연스럽게 **지식**과 **경험**을 습득하면서 성장하게 됩니다.

하지만 **지식**과 **경험**이 조직이 아닌 개발자 개인에게만 머무르게 된다면 어떻게 될까요? 

시간이 흐를수록 역량이 뛰어난 몇몇 개발자에 대한 의존도는 커지게 될 것입니다. 그리고 개발자의 이탈이 곧 조직의 큰 손실로 이어지게 되는 문제가 발생하게 됩니다.

이런 구조에서는 개발자가 이탈하게 되면 **지식**과 **경험**도 함께 사라지기 때문입니다. 

그래서 개발자 개인의 **지식**과 **경험**이 조직 전체로 전파되는 선순환 구조를 만드는 것이 필요합니다.

---

## 2. 공통 라이브러리 활용이 필요한 이유

소프트웨어를 개발하다 보면 다음과 같이 매번 반복적으로 등장하는 문제들이 있습니다. 

* 공통 로깅 처리
* 인증 및 보안 처리
* 외부 시스템 연동 방식
* 데이터 암호화
* 예외 처리 규칙
* API 응답 구조

문제 해결 방식을 재활용하면 반복되는 문제는 검증된 방식으로 빠르게 해결할 수 있을 것입니다.

또한 개발자의 **지식**과 **경험**이 개인에게만 머무르지 않고, 조직 전체로 공유되는 효과도 기대할 수 있습니다.

그렇게 문제 해결 방식의 재활용과 확장이 용이하도록 표준화되고, 코드로 정리된 형태가 **공통 라이브러리**입니다.

**공통 라이브러리**를 적극적으로 활용하는 조직에서는 개발자가 핵심 비즈니스 로직의 구현에 보다 집중할 수 있습니다. 반복되는 공통 코드는 라이브러리 내부로 감춰지기 때문에 코드가 간결해지고 복잡도도 낮출 수 있습니다.

결과적으로 소프트웨어 개발 품질과 생산성은 높아지고, 유지보수가 용이해지면서 조직 전체의 경쟁력 강화에도 도움이 됩니다.

---

## 3. 내부 라이브러리 저장소(Private Repository)의 필요성

**공통 라이브러리**를 적극적으로 활용하려면 **내부 라이브러리 저장소(Artifact Repository)**의 구축이 필요합니다. 조직 내부에서 개발한 코드를 함부로 Maven Central과 같은 공개 저장소에 배포할 수 없기 때문입니다. 

<img class="main-image" src="/assets/images/nexus2.jpg" alt="nexus repository">

**Nexus**는 **내부 라이브러리 저장소**를 구축하는데 활용할 수 있는 대표적인 도구입니다. 

```nohighlight
서비스 A ─┐
서비스 B ─┼── 공통 라이브러리 ── Nexus
서비스 C ─┘
```

---

## 4. 테스트 환경

개인 노트북에 **라이브러리 저장소**를 구축하고, 간단한 라이브러리를 만들어서 배포해보는 테스트를 수행하기 위한 환경을 만들었습니다. 

테스트 환경은 다음과 같습니다.

```nohighlight
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

---

## 5. Nexus 설치

 **Nexus**는 도커(Docker)를 이용해서 설치했습니다. 설치 절차는 다음과 같습니다.

---

### 5.1. 이미지 파일 다운로드

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
---

### 5.2. 다운로드 확인

```bash
$ sudo docker image ls

IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
sonatype/nexus3:latest   cb94c17229a3       1.12GB          399MB
```

---

### 5.3. 컨테이너 실행

```bash
$ sudo docker run --name nexus -d -p 5000:5000 -p 8081:8081 -v /nexus-data:/nexus-data -u root sonatype/nexus3

f3d2c2f6a043712721849013a8ff457432f480c590dfb9121e42df2229734030
```

**Nexus**의 웹 접속을 위해 8081 포트를 사용했고, **/nexus-data** 디렉터리를 볼륨으로 연결해서 저장소 데이터를 유지했습니다.

---

## 6. Nexus 접속

컨테이너가 정상적으로 실행되면 웹 브라우저를 통해 **Nexus**에 접속할 수 있습니다. 

최초 접속인 경우, 로그인 창에 admin계정의 초기 비밀번호가 저장된 파일 경로가 나타납니다. 해당 경로에서 비밀번호를 확인하고, 로그인하면 됩니다.

**최초 접속시 로그인 창**
<img class="sub-image" src="/assets/images/nexus1.png" alt="Nexus Repository">

로그인에 성공하면, 아래 그림처럼 Welcome 메시지가 나타납니다. 

**Welcome 메시지**
<img class="sub-image" src="/assets/images/nexus2.png" alt="Nexus Repository">

초기 비밀번호로 로그인하면 비밀번호를 변경해야 합니다.

**비밀번호 변경**
<img class="sub-image" src="/assets/images/nexus3.png" alt="Nexus Repository">

---

## 7. Nexus 저장소의 종류

**Nexus**에서 저장소(Repository) 목록을 확인하면 기본적으로 필요한 저장소들은 이미 생성되어 있습니다.

**Nexus Repository 목록**
<img class="sub-image" src="/assets/images/nexus4.png" alt="Nexus Repository">

저장소는 역할에 따라 다음과 같이 Type이 구분됩니다. 

* hosted
* group
* proxy

---

### 7.1. hosted

hosted 저장소는 직접 라이브러리를 업로드하여 관리합니다. **Nexus**를 설치하면, 다음과 같은 hosted 저장소가 기본으로 생성됩니다.

* maven-releases: 정식 버전의 라이브러리를 배포
* maven-snapshots: 개발중인 버전의 라이브러리를 배포

즉, 조직 내부에서 만든 라이브러리를 배포할 때 사용할 수 있습니다. 사실 이번 포스팅에서 다루고 있는 **내부 라이브러리 저장소**가 hosted 저장소에 해당됩니다. 

---

### 7.2. proxy

proxy 저장소는 외부 저장소를 대신 접근해주는 캐시 역할을 합니다. Gradle이나 Maven으로 라이브러리를 다운로드할 때, 보통 다음과 같은 공개 저장소가 사용됩니다.

```
https://repo.maven.apache.org/maven2
```

이때 proxy 저장소를 구성해서 사용하면, 다음과 같은 구조로 동작하게 됩니다.

```
개발자 → Nexus Proxy 저장소(내부) → Maven Central 저장소(외부)
```

라이브러리를 요청하면 최초에는 Maven Central에서 가져오지만, 그 이후에는 **Nexus**의 Proxy 저장소에 캐시된 라이브러리를 제공받게 됩니다.

이 방식은 다음과 같은 장점이 있습니다.

* 외부 저장소 의존성 감소
* 다운로드 속도 향상
* 외부 네트워크 장애 대응

---

### 7.3 group

group 저장소는 이름처럼 여러 저장소를 하나의 저장소처럼 묶어주는 역할을 합니다.

예를 들어 다음과 같은 저장소들이 있다고 가정해보겠습니다.

* maven-releases (hosted)
* maven-snapshots (hosted)
* maven-central (proxy)

이들을 하나의 group 저장소로 묶으면 다음과 같이 사용할 수 있습니다.

```
개발자 → maven-public(group)
           ├ maven-releases
           ├ maven-snapshots
           └ maven-central
```

이렇게 구성하면, 개발자는 여러 저장소를 각각 설정할 필요가 없습니다. group 저장소 하나만 설정하면 됩니다.

---

## 8. 라이브러리 배포

**Nexus**에는 기본적으로 **nx-admin**과 **nx-anonymous** 두 개의 Role이 정의되어 있습니다.

<img class="sub-image" src="/assets/images/security_roles.jpg" alt="Nexus Roles 목록">

**nx-admin** Role의 Privileges는 all로 지정되어 있습니다. 로그인에 사용한 admin 계정의 Role이 **nx-admin**으로 부여되어 있습니다. 즉, admin 계정은 라이브러리 배포를 포함한 모든 권한을 가지고 있다는 의미입니다. 

보통은 배포 권한을 가진 계정을 별도로 만들어서 사용합니다. 하지만 이번에는 라이브러리 배포 과정을 확인하는 것이 주 목적이기 때문에, admin 계정을 사용했습니다.

---

### 8.1. simple-lib: 라이브러리 코드 작성

라이브러리 배포를 위해서 아주 간단한 Java 프로젝트를 하나 만들었습니다.

프로젝트 이름은 simple-lib입니다.

**settings.gradle**
```
rootProject.name = 'simple-lib'
```

이 프로젝트에 SimpleLib 클래스를 선언하고, 두 개의 숫자를 입력받아 합을 반환하는 코드를 작성했습니다.

**SimpleLib.java**
```java
package sanghoon.study.lib;

public class SimpleLib {
    public int sum(int x,int y){
        return x+y;
    }
}
```

> 실제 공통 라이브러리는 인증, 로깅, 보안, 외부 시스템 연동과 같은 다양하고 복잡한 기능이 포함될 수 있습니다. 

---

### 8.2. simple-lib: Gradle 설정

Gradle 공식 가이드에서는 Maven 저장소 배포에는 maven-publish 플러그인을 사용하도록 안내하고 있습니다.

maven-publish 플러그인은 publishing { publications { ... } repositories { ... } } 구조로 publication과 저장소를 정의합니다.

**build.gradle**
```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
}

group = 'sanghoon.study'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
    withSourcesJar()
}

repositories {
    mavenCentral()
}

publishing {
    publications {
        create('mavenJava', MavenPublication) {
            from components.java

            groupId = project.group
            artifactId = 'simple-lib'
            version = project.version
        }
    }

    repositories {
        maven {
            name = 'localNexus'
            url = uri('http://10.68.65.187:8081/repository/maven-snapshots/')
            allowInsecureProtocol = true

            credentials {
                username = findProperty('nexusUsername') ?: System.getenv('NEXUS_USERNAME')
                password = findProperty('nexusPassword') ?: System.getenv('NEXUS_PASSWORD')
            }
        }
    }
}
```

---

### 8.3. simple-lib: 배포 경로

위의 build.gradle 파일에 배포할 라이브러리의 경로가 다음과 같이 작성되어 있습니다. 

```
groupId    = sanghoon.study
artifactId = simple-lib
version    = 0.0.1-SNAPSHOT
```

simple-lib를 다른 프로젝트에서 사용하려면 아래와 같은 경로를 build.gradle 파일에 추가해주면 됩니다. 

```
implementation 'sanghoon.study:simple-lib:0.0.1-SNAPSHOT'
```

---

### 8.4. simple-lib: 배포하기

Gralde을 reload하고, publish task가 생성되었는지 확인합니다. 

publish task를 실행하면, Gradle이 프로젝트를 빌드한 뒤 지정된 저장소로 라이브러리가 배포됩니다.

**maven-snapshots 저장소 목록**
<img class="sub-image" src="/assets/images/simple_lib.jpg" alt="저장소에 simple-lib 추가">

---

## 9. 라이브러리 사용

이제 mavan-snapshots 저장소에 배포된 simple-lib 라이브러리를 가져오는 사용해 볼 차례입니다.

---

### 9.1. simple-calc: 테스트 코드 작성

이번에도 아주 간단한 Java 프로젝트를 하나 만들었습니다.

프로젝트 이름은 simple-calc입니다.

**settings.gradle**
```
rootProject.name = 'simple-calc'
```

이 프로젝트에 Main 클래스를 선언하고, simple-lib 라이브러리의 sum() 메서드를 호출하도록 코드를 작성했습니다.

**Main.java**
```java
import sanghoon.study.lib.SimpleLib;

public class Main {
    public static void main(String[] args) {
        SimpleLib simpleLib = new SimpleLib();

        System.out.println(simpleLib.sum(3,5));
    }
}
```

---

### 9.2. simple-calc: Gradle 설정

simple-lib를 사용하기 위해서는 build.gradle을 다음과 같이 작성해야 합니다.

**build.gradle**
```groovy
plugins {
    id 'java'
}

group = 'org.example'
version = '1.0-SNAPSHOT'

repositories {
    mavenCentral()

    maven {
        url = uri('http://10.68.65.187:8081/repository/maven-snapshots/')
        allowInsecureProtocol = true
    }
}

dependencies {
    testImplementation platform('org.junit:junit-bom:5.10.0')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    implementation 'sanghoon.study:simple-lib:0.0.1-SNAPSHOT'
}

test {
    useJUnitPlatform()
}
```

코드를 실행시키면 sum() 메서드가 정상적으로 호출되면서 기대했던 값인 8이 터미널로 출력되는 것을 확인할 수 있습니다.

---

#Nexus #라이브러리 #배포