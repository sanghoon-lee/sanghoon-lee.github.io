---
layout: post
title: "[토이프로젝트] Scouter와 PagerDuty로 장애 감지부터 On-call까지: (2) Spring Boot 애플리케이션에 Scouter Agent 연동과 모니터링 시작"
date: 2026-07-27
series: paygerduty
description: 
image: /assets/images/monitoring.jpg
categories: 토이프로젝트
tags:
  - 인프라
  - 자바
---

`Scouter Server`를 구성했다고 해서 애플리케이션의 상태가 자동으로 수집되지는 않습니다.

CPU 사용률, Heap 메모리, Active Service, XLog와 같은 운영 정보를 수집하려면 모니터링 대상 애플리케이션에 `Scouter Agent`를 연동해야 합니다.

`Scouter Agent`가 연동되면 `Scouter Client`를 통해 해당 애플리케이션의 다양한 운영 정보를 실시간으로 확인할 수 있습니다.

---

## 1. Scouter Agent 준비

먼저 `Scouter Agent`를 다운로드한 뒤, 애플리케이션과 연동에 필요한 파일을 준비했습니다.

---

### 1.1. Scouter Agent 다운로드

`Scouter` 프로젝트의 공식 GitHub 저장소의 주소는 다음과 같습니다.

* [https://github.com/scouter-project/scouter](https://github.com/scouter-project/scouter)

공식 GitHub 저장소의 `release` 브랜치에서 `2.17.1` 태그를 선택했습니다.

<img class="sub-image" src="/assets/images/scouter-release-tags.jpg" alt="Scouter 프로젝트 Release 브랜치의 Tag 목록">

그리고 페이지 하단에 위치한 **Assets**에서 `scouter-all-2.17.1.tar.gz` 파일을 다운로드했습니다. 

<img class="sub-image" src="/assets/images/scouter-assets.jpg" alt="Scouter 2.17.1 배포 버전의 Assets 목록">

압축을 해제하면 `scouter/agent.java` 디렉터리에서 `Scouter Agent` 연동에 필요한 `scouter.agent.jar` 파일을 확인할 수 있습니다.

**참고**

> `scouter-all-2.17.1.tar.gz`에는 Server, Client, Agent 등 Scouter를 구성하는 모든 구성 요소가 포함되어 있습니다. 이번 실습에서는 Agent 연동에 필요한 `scouter.agent.jar`만 사용합니다.

---

### 1.2. scouter.agent.jar 파일 배치

`Scouter Agent`는 JVM이 시작될 때 `-javaagent` 옵션을 통해 애플리케이션과 함께 로드됩니다. 따라서 애플리케이션이 실행되는 환경에서 쉽게 참조할 수 있는 위치에 `scouter.agent.jar` 파일을 두는 것이 좋습니다.

이전 포스팅에서 언급했던 것처럼 새로운 애플리케이션을 별도로 만들지 않고, **Nginx로 Reverse Proxy 구성하기** 시리즈에서 사용했던 애플리케이션(`simple-api`)을 그대로 활용했습니다. 

참고로 GitHub 저장소는 다음과 같습니다.

* [https://github.com/sanghoon-lee/simple-api](https://github.com/sanghoon-lee/simple-api)

편의상 프로젝트 루트 디렉터리에 `agent` 디렉터리를 생성한 뒤, `scouter.agent.jar` 파일을 복사했습니다.

---

### 1.3. scouter.conf 작성

`Scouter Agent`는 `scouter.conf` 파일을 통해 `Scouter Server`의 주소와 애플리케이션 이름 등 필요한 정보를 설정합니다.

이번 실습에서는 `agent` 디렉터리에 `scouter.conf` 파일을 생성하고 다음과 같이 작성했습니다.

```properties
net_collector_ip=192.168.56.13
net_collector_tcp_port=6100
obj_host=app-server
obj_type=tomcat
```

각 설정의 의미는 다음과 같습니다.

|설정|설명|
|---|---|
|`net_collector_ip`|Scouter Server 주소|
|`net_collector_tcp_port`|Collector TCP 포트|
|`obj_host`|`Scouter Client`에서 표시할 호스트 이름|
|`obj_type`|애플리케이션 종류|

Spring Boot 애플리케이션은 내장 Tomcat을 사용하므로 `obj_type`을 `tomcat`으로 설정했습니다.

반면 `Scouter Client`에서 애플리케이션 인스턴스를 구분하는 `obj_name`은 `scouter.conf`에서 설정하지 않았습니다. 동일한 Docker 이미지를 사용해 여러 인스턴스를 실행할 예정이므로 컨테이너 실행 시 환경변수로 각각 다른 값을 주입하기로 했습니다.

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

## 2. Docker 이미지 구성

`Scouter Agent`를 준비한 이후에는 Docker 이미지에 포함시켜야 했습니다. 이를 위해 기존의 Dockerfile을 수정하고 새로운 이미지를 빌드했습니다.

---

### 2.1. Dockerfile 수정

GitHub 저장소에 있는 `simple-api`의 Dockerfile은 아래와 같이 작성되어 있습니다.

**변경 전 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar

ENTRYPOINT ["java", "-jar", "simple-api.jar"]
```

하지만 `Scouter Agent`를 사용하려면 `scouter.agent.jar`와 설정 파일인 `scouter.conf`가 Docker 이미지에 함께 포함되어야 합니다. 

또한 JVM이 시작될 때 `-javaagent` 옵션을 통해 Agent가 함께 실행되도록 설정해야 합니다. 

추가로 하나의 Docker 이미지로 여러 인스턴스를 실행하기 위해 `obj_name`은 `SCOUTER_OBJ_NAME` 환경변수로 전달하도록 구성했습니다.

이를 위해 `Dockerfile`을 아래와 같이 수정했습니다.

**변경 후 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/simple-api-0.0.1-SNAPSHOT.jar simple-api.jar
COPY agent/scouter.agent.jar agent/scouter.agent.jar
COPY agent/scouter.conf agent/scouter.conf

ENTRYPOINT [
    "sh",
    "-c",
    "java \
    -javaagent:/app/agent/scouter.agent.jar \
    -Dscouter.config=/app/agent/scouter.conf \
    -Dobj_name=${SCOUTER_OBJ_NAME} \
    -jar simple-api.jar"
]
```

---

### 2.2. Docker 이미지 빌드

Dockerfile 수정을 마치고, `simple-api-scouter`라는 이름으로 새로운 도커 이미지를 빌드했습니다.

```bash
docker build -t simple-api-scouter .
```

---

### 2.3. 컨테이너 실행