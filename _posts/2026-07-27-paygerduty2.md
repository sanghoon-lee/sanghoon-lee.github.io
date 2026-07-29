---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (2) Spring Boot 애플리케이션에 Scouter Agent 연동"
date: 2026-07-27
series: paygerduty
description: Spring Boot 애플리케이션에 Scouter Agent를 연동하고, Docker Compose로 여러 인스턴스를 실행해 Agent가 정상적으로 로드되는 과정을 정리합니다.
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Server`를 구성했다고 해서 CPU 사용률, Heap 메모리, Active Service, XLog와 같은 애플리케이션의 운영 정보가 자동으로 수집되지는 않습니다.

이러한 운영 정보를 수집하려면 모니터링 대상 애플리케이션에 `Scouter Agent`를 연동해야 합니다.

`Scouter Agent`가 수집한 정보는 `Scouter Server`로 전송되며, `Scouter Client`를 통해 실시간으로 확인할 수 있습니다.

---

## 1. Scouter Agent 준비

먼저 애플리케이션에 `Scouter Agent`를 연동하기 위해 필요한 파일을 다운로드하고, 설정 파일을 준비했습니다.

---

### 1.1. Scouter Agent 다운로드

`Scouter` 프로젝트의 공식 GitHub 저장소의 주소는 다음과 같습니다.

* [https://github.com/scouter-project/scouter](https://github.com/scouter-project/scouter)

저장소의 `release` 브랜치로 이동해서 `2.17.1` 태그를 선택했습니다.

<img class="sub-image" src="/assets/images/scouter-release-tags.jpg" alt="Scouter 프로젝트 Release 브랜치의 Tag 목록">

그리고 페이지 하단에 위치한 **Assets**에서 `scouter-all-2.17.1.tar.gz` 파일을 다운로드했습니다. 

<img class="sub-image" src="/assets/images/scouter-assets.jpg" alt="Scouter 2.17.1 배포 버전의 Assets 목록">

압축을 해제하면 `scouter/agent.java` 디렉터리에서 `Scouter Agent` 연동에 필요한 `scouter.agent.jar` 파일을 확인할 수 있습니다.

**참고**

> `scouter-all-2.17.1.tar.gz`에는 `Scouter Server`와 `Java Agent`, `Host Agent` 등 모니터링 환경을 구성하는 데 필요한 파일이 포함되어 있습니다. 이번에는 애플리케이션 연동에 필요한 `scouter.agent.jar`만 사용합니다.

---

### 1.2. scouter.agent.jar 파일 배치

`Scouter Agent`는 JVM이 시작될 때 `-javaagent` 옵션을 통해 애플리케이션과 함께 로드됩니다. 따라서 애플리케이션이 실행되는 환경에서 쉽게 참조할 수 있는 위치에 `scouter.agent.jar` 파일을 두는 것이 좋습니다.

이전 포스팅에서 언급했던 것처럼 새로운 애플리케이션을 별도로 만들지 않고, **Nginx로 Reverse Proxy 구성하기** 시리즈에서 사용했던 애플리케이션(`simple-api`)을 그대로 활용했습니다. 

참고로 GitHub 저장소는 다음과 같습니다.

* [https://github.com/sanghoon-lee/simple-api](https://github.com/sanghoon-lee/simple-api)

편의상 프로젝트 루트 디렉터리에 `agent` 디렉터리를 생성한 뒤, `scouter.agent.jar` 파일을 복사했습니다.

**참고: Nginx로 Reverse Proxy 구성하기 시리즈**

> [(1) 개념과 동작 원리](https://sanghoon-lee.github.io/2026/06/03/nginx-rp/)<br>
> [(2) 애플리케이션 서버 구현](https://sanghoon-lee.github.io/2026/06/04/nginx-rp2/)<br>
> [(3) Reverse Proxy 서버 구축 ](https://sanghoon-lee.github.io/2026/06/07/nginx-rp3/)<br>
> [(4) 애플리케이션 서버 보호 기능 검증](https://sanghoon-lee.github.io/2026/06/08/nginx-rp4/)<br>
> [(5) 로드밸런싱 기능 검증](https://sanghoon-lee.github.io/2026/06/14/nginx-rp5/)<br>
> [(6) SSL 종료 기능 검증](https://sanghoon-lee.github.io/2026/06/14/nginx-rp6/)<br>
> [(7) 정적 콘텐츠 처리 기능 검증](https://sanghoon-lee.github.io/2026/06/17/nginx-rp7/)<br><br>

---

### 1.3. scouter.conf 작성

`Scouter Agent`는 `scouter.conf` 파일을 통해 `Scouter Server`의 주소와 애플리케이션 이름 등 필요한 정보를 설정합니다.

이번 실습에서는 `agent` 디렉터리에 `scouter.conf` 파일을 생성하고 다음과 같이 작성했습니다.

```properties
net_collector_ip=192.168.56.13
net_collector_udp_port=6100
net_collector_tcp_port=6100
obj_host=app-server
obj_type=tomcat
```

각 설정의 의미는 다음과 같습니다.

| 설정                       | 설명                            |
| ------------------------ | ----------------------------- |
| `net_collector_ip`       | Scouter Server 주소             |
| `net_collector_udp_port` | Collector UDP 포트              |
| `net_collector_tcp_port` | Collector TCP 포트              |
| `obj_host`               | `Scouter`에서 애플리케이션이 속한 호스트를 구분하기 위한 이름 |
| `obj_type`               | 애플리케이션 종류                     |


Spring Boot 애플리케이션은 내장 Tomcat을 사용하므로 `obj_type`을 `tomcat`으로 설정했습니다.

반면 `Scouter Client`에서 애플리케이션 인스턴스를 구분하는 `obj_name`은 `scouter.conf`에서 설정하지 않고, `Docker Compose`에서 설정했습니다.

**소스코드 프로젝트 구조**

```nohighlight
simple-api
├── Dockerfile
├── agent
│   ├── scouter.agent.jar
│   └── scouter.conf
├── build.gradle
└── src
```

---

## 2. 애플리케이션 Docker 이미지 준비

`Scouter Agent`를 준비한 이후에는 Docker 이미지에 포함시켜야 했습니다. 이를 위해 기존의 Dockerfile을 수정하고 새로운 이미지를 빌드했습니다.

---

### 2.1. Dockerfile 수정

GitHub 저장소에 있는 `simple-api`의 `Dockerfile`은 아래와 같이 작성되어 있습니다.

**변경 전 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar

ENTRYPOINT ["java", "-jar", "simple-api.jar"]
```

하지만 `Scouter Agent`를 사용하려면 `scouter.agent.jar`와 설정 파일인 `scouter.conf`가 Docker 이미지에 함께 포함되어야 합니다. 

또한 JVM이 시작될 때 `-javaagent` 옵션을 통해 Agent가 함께 실행되도록 설정해야 합니다. 

우선 `scouter.agent.jar`와 `scouter.conf`가 이미지에 포함되도록 `Dockerfile`을 수정했습니다. `-javaagent`를 비롯한 JVM 실행 옵션은 여러 인스턴스에 서로 다른 값을 적용할 수 있도록 `Docker Compose`에서 설정했습니다.

**변경 후 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar
COPY agent/scouter.agent.jar agent/scouter.agent.jar
COPY agent/scouter.conf agent/scouter.conf

ENTRYPOINT ["java", "-jar", "simple-api.jar"]
```

---

### 2.2. Docker 이미지 빌드

`Dockerfile`의 수정을 마치고 `simple-api-scouter`라는 이름으로 새로운 Docker 이미지를 빌드했습니다.

```bash
docker build -t simple-api-scouter .
```

**참고**

> Docker는 이전 빌드 결과를 캐시로 재사용할 수 있습니다. 변경한 파일이 이미지에 반영되지 않은 것으로 보이는 경우 `docker build --no-cache -t simple-api-scouter .` 명령으로 캐시를 사용하지 않고 다시 빌드할 수 있습니다.

---

## 3. Docker Compose로 애플리케이션 실행

`docker run` 명령으로 애플리케이션 컨테이너를 실행해도 됩니다. 

하지만 이번에는 동일한 이미지로 두 개의 컨테이너를 실행하면서 각 인스턴스에 서로 다른 `obj_name`을 설정해야 합니다. 또한 JVM 시작 시 `-javaagent` 옵션도 함께 전달해야 합니다. 이러한 실행 옵션을 한 곳에서 관리하기 위해 `Docker Compose`를 사용했습니다.

---

### 3.1. Docker Compose 구성

`Docker Compose`를 사용해서 애플리케이션을 실행하기 위해 `docker-compose.yml` 파일을 아래와 같이 작성했습니다.

```yml
services:
  simple-api-1:
    image: simple-api-scouter
    container_name: api-01
    ports:
      - "8081:8000"
    environment:
      SERVER_NAME: api-01
      JAVA_TOOL_OPTIONS: >-
        -javaagent:/app/agent/scouter.agent.jar
        -Dscouter.config=/app/agent/scouter.conf
        -Dobj_name=simple-api-1

  simple-api-2:
    image: simple-api-scouter
    container_name: api-02
    ports:
      - "8082:8000"
    environment:
      SERVER_NAME: api-02
      JAVA_TOOL_OPTIONS: >-
        -javaagent:/app/agent/scouter.agent.jar
        -Dscouter.config=/app/agent/scouter.conf
        -Dobj_name=simple-api-2
```

Docker Compose로 구성한 두 컨테이너는 동일한 `simple-api-scouter` 이미지를 사용합니다. 하지만 일부 속성은 다른 값으로 설정해야 합니다.

| 구분                 | `simple-api-1` 서비스   | `simple-api-2` 서비스   |
| ------------------ | -------------- | -------------- |
| 컨테이너 이름            | `api-01` | `api-02` |
| 포트 매핑              | `8081` → `8000` | `8082` → `8000` |
| Scouter `obj_name` | `simple-api-1` | `simple-api-2` |

---

### 3.2. 컨테이너 실행

만약 **Nginx로 Reverse Proxy 구성하기** 시리즈에서 사용했던 `api-01`과 `api-02` 컨테이너가 남아 있다면 먼저 삭제해야 합니다.

새로 빌드한 `simple-api-scouter` 이미지에서도 `api-01`, `api-02`라는 동일한 이름을 사용하기 때문입니다. Docker에서는 같은 이름의 컨테이너를 중복해서 생성할 수 없으므로 기존 컨테이너가 남아 있으면 Docker Compose 실행 과정에서 이름 충돌이 발생합니다.

그래서 `docker-compose.yml` 파일을 작성하고, 기존에 만들었던 컨테이너를 삭제했습니다.

```bash
$ docker rm -f api-01 api-02
```

이후, 새로 빌드한 `simple-api-scouter` 이미지로 컨테이너를 실행했습니다.

```bash
$ docker compose up -d
```

---

### 3.3. 컨테이너 실행 상태 확인

다음 명령으로 `api-01`과 `api-02` 컨테이너가 모두 정상적으로 실행되고 있는 것을 확인할 수 있었습니다.

```bash
$ docker compose ps
```

또한 애플리케이션 시작 로그를 통해 `Scouter Agent`가 정상적으로 로드된 것도 확인했습니다.

```bash
$ sudo docker logs api-01
```

```text
Picked up JAVA_TOOL_OPTIONS: -javaagent:/app/agent/scouter.agent.jar -Dscouter.config=/app/agent/scouter.conf -Dobj_name=simple-api-1
  ____                  _
 / ___|  ___ ___  _   _| |_ ___ _ __
 \___ \ / __/   \| | | | __/ _ \ '__|
  ___) | (_| (+) | |_| | ||  __/ |
 |____/ \___\___/ \__,_|\__\___|_|
 Open Source S/W Performance Monitoring
 Scouter version 2.17.1

20260728 14:49:31 [SCOUTER] loaded by system classloader
20260728 14:49:31 [SCOUTER] jar:file:/app/agent/scouter.agent.jar
20260728 14:49:31 [SCOUTER] Version 2.17.1 2022-03-27 04:35 GMT_ENV_java8plus
20260728 14:49:32 [SCOUTER] objType:tomcat
20260728 14:49:32 [SCOUTER] objName:/535c1cd092b1/simple-api-1
20260728 14:49:32 [A113] Counter Collector Started (#19)
20260728 14:49:32 [A113] InteractionCounter Collector Started (#19)
20260728 14:49:32 [SCOUTER] Configure -Dscouter.config=/app/agent/scouter.conf
20260728 14:49:32 [A100] agent boot seed=x1jqkj7jgk
20260728 14:49:33 [A119] Agent UDP local.port=0
```

로그에서 `JAVA_TOOL_OPTIONS`가 JVM에 전달된 것과 `scouter.agent.jar`가 시스템 클래스 로더를 통해 로드된 것을 확인할 수 있습니다. 또한 `objType`이 `tomcat`으로 설정되고, `objName` 끝에 `simple-api-1`이 표시되는 것으로 보아 Scouter Agent 설정도 정상적으로 적용되었습니다.

**참고**

> 로그의 `Agent UDP local.port=0`은 Collector의 UDP 포트가 아니라 Agent가 사용하는 로컬 UDP 포트를 의미합니다. `scouter.conf` 파일의 `net_collector_udp_port=6100`은 Scouter Agent가 데이터를 전송할 대상인 Scouter Server의 UDP 포트이므로 서로 다른 값입니다.

**참고**

> Docker 환경에서는 Scouter Agent 로그의 `objName`에 컨테이너의 `hostname`과 `-Dobj_name`으로 설정한 값이 함께 표시될 수 있습니다.
>
> 위 로그에서도 `objName`이 `/535c1cd092b1/simple-api-1`로 출력되었습니다. `535c1cd092b1`은 Docker 컨테이너에 부여된 `hostname`이며, `simple-api-1`은 JVM 시스템 프로퍼티인 `-Dobj_name`으로 지정한 값입니다.


---

## 4. 다음 포스팅

`Scouter Client`를 사용하려면 운영체제에 맞는 Client를 별도로 다운로드해야 합니다. 

다음 포스팅에서는 Windows 운영체제용 Client를 설치하고, `Scouter Client`를 통해 수집된 애플리케이션의 운영 정보를 실시간으로 확인해보도록 하겠습니다.
