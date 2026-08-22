---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (2) Spring Boot 애플리케이션에 Scouter Agent 연동"
date: 2026-07-27
series: paygerduty
description: Spring Boot 애플리케이션에 Scouter Agent 2.20.0을 연동하고, Docker Compose로 여러 인스턴스를 실행해 Agent 설정과 로드 여부를 확인하는 과정을 정리합니다.
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Server`를 구성했다고 해서 애플리케이션의 운영 정보가 자동으로 수집되지는 않습니다. 모니터링 대상 애플리케이션에 `Scouter Agent`가 연동되어야 운영 정보가 수집될 수 있습니다.

---

## 1. Scouter Agent 준비

`Scouter Agent`를 연동하려면 `scouter.agent.jar` 파일이 필요합니다. 이전 포스팅에서 Scouter 2.20.0 배포 파일인 `scouter-all-2.20.0.tar.gz`를 다운로드 했습니다.

이 파일의 압축을 해제하면 `scouter/agent.java` 디렉터리에서 `scouter.agent.jar` 파일을 찾을 수 있습니다.

**참고**

> [Scouter 2.20.0 배포 파일(`scouter.agent.jar`) 다운로드 방법](https://sanghoon-lee.github.io/2026/07/26/scouter/#scouter-download)

---

### 1.1. Scouter Agent 파일 배치

`Scouter Agent`는 JVM이 시작될 때 `-javaagent` 옵션을 통해 모니터링 대상 애플리케이션과 함께 로드됩니다. 따라서 Docker 이미지를 생성할 때 Agent 파일도 함께 포함될 수 있도록 애플리케이션 프로젝트에 `agent` 디렉터리를 추가했습니다.

**참고**

> [모니터링 환경 구성](https://sanghoon-lee.github.io/2026/07/26/scouter/#environment)

그리고 `agent` 디렉터리에 `scouter.agent.jar` 파일을 복사했습니다. 

---

### 1.2. scouter.conf 작성

 `Scouter Agent`가 운영 정보를 `Scouter Server`로 전송하는 데 필요한 설정은 `scouter.conf` 파일에서 정의하게 됩니다.

`agent` 디렉터리에 `scouter.conf` 파일을 아래와 같이 작성했습니다.

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

두 컨테이너는 동일한 애플리케이션의 인스턴스이므로 `obj_host`는 `app-server`로 동일하게 설정했습니다. 대신 각 컨테이너의 `obj_name`을 `api-01`, `api-02`로 다르게 지정해 개별 인스턴스를 구분하도록 구성했습니다.

또한 Spring Boot 애플리케이션은 내장 Tomcat을 사용하므로 `obj_type`을 `tomcat`으로 설정했습니다.

---

## 2. 애플리케이션 Docker 이미지 준비

애플리케이션과 함께 `Scouter Agent`를 실행하려면 `scouter.agent.jar`와 `scouter.conf` 파일도 Docker 이미지에 포함되어야 합니다. 이를 위해 `simple-api` 프로젝트에서 사용하던 `Dockerfile`을 수정하고, 이미지를 새로 빌드했습니다.

---

### 2.1. 애플리케이션 프로젝트 구조

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

### 2.2. Dockerfile 수정

GitHub 저장소에 등록되어 있는 `simple-api`의 `Dockerfile`은 아래와 같이 작성되어 있습니다.

**변경 전 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar

ENTRYPOINT ["java", "-jar", "simple-api.jar"]
```

그래서 `scouter.agent.jar`와 `scouter.conf` 파일이 이미지에 포함되도록 `Dockerfile`을 아래와 같이 수정했습니다.

**변경 후 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar
COPY agent/scouter.agent.jar agent/scouter.agent.jar
COPY agent/scouter.conf agent/scouter.conf

ENTRYPOINT ["java", "-jar", "simple-api.jar"]
```

`Dockerfile`의 `ENTRYPOINT`에는 `-javaagent` 옵션을 직접 추가하지 않았습니다. 동일한 이미지를 사용하는 각 컨테이너에 서로 다른 `obj_name`을 설정하기 위해 Agent 실행 옵션은 `docker-compose.yml`의 `JAVA_TOOL_OPTIONS`를 통해 전달하도록 구성했습니다.

---

### 2.3. Docker 이미지 빌드

이어서 수정된 `Dockerfile` 파일을 이용해서 `simple-api-scouter`라는 이름의 새로운 Docker 이미지를 빌드했습니다.

```bash
docker build -t simple-api-scouter .
```

**참고**

> Docker는 이전 빌드 결과를 캐시로 재사용할 수 있습니다. 변경한 파일이 이미지에 반영되지 않은 것으로 보이는 경우 `docker build --no-cache -t simple-api-scouter .` 명령으로 캐시를 사용하지 않고 다시 빌드할 수 있습니다.

---

## 3. 애플리케이션 실행

`docker run` 명령으로 애플리케이션의 컨테이너를 실행해도 되지만, 실행 옵션을 한 곳에서 관리하기 위해 `Docker Compose`를 사용했습니다.

동일한 이미지로 두 개의 컨테이너를 실행하지만, Scouter에서는 각각을 별도의 모니터링 대상으로 식별해야 합니다. 따라서 컨테이너별로 서로 다른 `obj_name`을 설정하고, JVM 시작 시 `-javaagent` 옵션을 전달하도록 구성했습니다.

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
        -Dobj_name=api-01

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
        -Dobj_name=api-02
```

`docker-compose.yml` 파일에서 정의한 두 컨테이너는 동일한 `simple-api-scouter` 이미지를 사용합니다. 하지만 일부 속성은 다른 값으로 설정해야 합니다.

| 구분                 | `simple-api-1` 서비스   | `simple-api-2` 서비스   |
| ------------------ | -------------- | -------------- |
| 컨테이너 이름            | `api-01` | `api-02` |
| 포트 매핑              | `8081` → `8000` | `8082` → `8000` |
| Scouter `obj_name` | `api-01` | `api-02` |

`JAVA_TOOL_OPTIONS`는 JVM이 시작될 때 자동으로 읽는 환경변수입니다. 이를 이용해 `Dockerfile`의 `ENTRYPOINT`를 변경하지 않고도 `-javaagent`와 Scouter 관련 시스템 프로퍼티를 JVM에 전달할 수 있습니다.

`-javaagent`는 `Scouter Agent`를 JVM에 로드하고, `-Dscouter.config`는 Agent 설정 파일의 위치를 지정합니다. `-Dobj_name`은 두 애플리케이션 인스턴스를 Scouter에서 구분하기 위해 각각 다른 값으로 설정했습니다.

---

### 3.2. 컨테이너 실행

만약 **Nginx로 Reverse Proxy 구성하기** 시리즈에서 사용했던 `api-01`과 `api-02` 컨테이너가 남아 있다면 먼저 삭제해야 합니다. Docker에서는 같은 이름의 컨테이너를 중복해서 생성할 수 없으므로 기존 컨테이너가 남아 있으면 Docker Compose 실행 과정에서 이름 충돌이 발생하기 때문입니다.

기존 컨테이너가 남아 있었기 때문에, 다음 명령으로 삭제했습니다.

```bash
$ docker rm -f api-01 api-02
```

이후 새로 빌드한 `simple-api-scouter` 이미지를 사용해 컨테이너를 실행했습니다.

```bash
$ docker compose up -d
```

---

### 3.3. Scouter Agent 로드 확인

애플리케이션 시작 로그를 통해 `api-01`과 `api-02` 두 개의 컨테이너에 `Scouter Agent`가 모두 정상적으로 로드된 것을 확인할 수 있었습니다.

```bash
$ docker logs api-01
```

```text
Picked up JAVA_TOOL_OPTIONS: -javaagent:/app/agent/scouter.agent.jar -Dscouter.config=/app/agent/scouter.conf -Dobj_name=api-01
  ____                  _
 / ___|  ___ ___  _   _| |_ ___ _ __
 \___ \ / __/   \| | | | __/ _ \ '__|
  ___) | (_| (+) | |_| | ||  __/ |
 |____/ \___\___/ \__,_|\__\___|_|
 Open Source S/W Performance Monitoring
 Scouter version 2.20.0

20260822 06:25:08 [SCOUTER] Version 2.20.0 2023-05-29 05:14 GMT_ENV_java8plus
20260822 06:25:08 [SCOUTER] loaded by system classloader
20260822 06:25:08 [SCOUTER] jar:file:/app/agent/scouter.agent.jar
20260822 06:25:09 [SCOUTER] objType:tomcat
20260822 06:25:09 [SCOUTER] objName:/78afd06c5c17/api-01
20260822 06:25:09 [A113] Counter Collector Started (#19)
20260822 06:25:09 [A113] InteractionCounter Collector Started (#19)
20260822 06:25:09 [SCOUTER] Configure -Dscouter.config=/app/agent/scouter.conf
20260822 06:25:09 [A100] agent boot seed=x1k4k2a0r1
20260822 06:25:10 [A119] Agent UDP local.port=0
```

설정한 `JAVA_TOOL_OPTIONS`가 JVM에 그대로 전달되었고, `scouter.agent.jar`가 시스템 클래스 로더를 통해 로드되었습니다. `objType`이 `tomcat`으로 설정되고 `objName` 끝에 `api-01`이 표시되는 것을 통해 `Scouter Agent` 설정도 정상적으로 적용되었음을 확인할 수 있었습니다.

**참고**

> 로그의 `Agent UDP local.port=0`은 Collector의 UDP 포트가 아니라 Agent가 사용하는 로컬 UDP 포트를 의미합니다. `scouter.conf` 파일의 `net_collector_udp_port=6100`은 Scouter Agent가 데이터를 전송할 대상인 `Scouter Server`의 UDP 포트이므로 서로 다른 값입니다.
>
> 또한 Docker 환경에서는 Scouter Agent 로그의 `objName`에 컨테이너의 `hostname`과 `-Dobj_name`으로 설정한 값이 함께 표시될 수 있습니다.
> 위 로그에서도 `objName`이 `/78afd06c5c17/api-01`로 출력되었습니다. `78afd06c5c17`은 Docker 컨테이너에 부여된 `hostname`이며, `api-01`은 JVM 시스템 프로퍼티인 `-Dobj_name`으로 지정한 값입니다.

---

## 4. 다음 포스팅

이번 포스팅에서는 Spring Boot 애플리케이션에 `Scouter Agent`를 연동하고, 애플리케이션 실행 시 Agent가 정상적으로 로드되는 것까지 확인했습니다.

다음 포스팅에서는 Windows용 `Scouter Client`를 설치하고 `Scouter Server`에 접속하여, 두 애플리케이션 컨테이너의 운영 정보가 정상적으로 수집되고 있는지 확인해보겠습니다.
