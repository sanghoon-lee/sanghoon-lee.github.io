---
layout: post
title: "JMeter를 이용한 Vault Transit Engine 성능 테스트"
date: 2026-05-15
description: JMeter를 이용해서 Vault Transit Engine의 성능을 테스트해 본 과정을 정리해서 작성한 글입니다.
image: /assets/images/hashicorp.jpg
categories: 학습기록
---

시크릿 관리 통제 강화와 애플리케이션 내부에서 수행하던 암호화 연산의 외부화를 핵심 방향으로 Vault 기반 보안체계 구축을 검토하게 되었습니다.

사내 관련 업무 담당자들과 논의하는 과정에서 과거 HSM을 도입해서 암호화 연산의 외부화를 시도했으나 성능 이슈와 운영 안정성 문제로 실패했던 사례가 있었다는 이야기를 들을 수 있었습니다.

암호화 연산을 외부화하여 애플리케이션과 분리하는 구조에서는 성능과 응답 안정성이 매우 중요한 요소입니다. 따라서 Vault Transit Engine의 성능 특성과 기능적 적합성을 사전에 검증할 필요가 있었습니다.

---


## 1. Apache JMeter

<img class="main-image" src="/assets/images/jmeter2.png" alt="Apache JMeter">

과거 정부R&D과제를 수행하면서 KOLAS(Korea Laboratory Accreditation Scheme) 공인시험기관을 통해 소프트웨어의 성능과 기능을 시험하고, 공인 성적서를 발급받아 제출했던 경험이 있습니다. 당시 성능 테스트 도구로 Apache JMeter를 주로 사용했습니다. 

> KOLAS: 한국인정기구(Korea Laboratory Accreditation Scheme)의 약자로, 국제 표준에 따라 시험 교정 검사 기관의 기술 능력과 품질 시스템을 평가하여 공인하는 산업통상자원부 산하 국가기구입니다.

Apache JMeter는 다양한 시스템의 부하 및 성능 테스트에 널리 사용되는 대표적인 오픈소스 성능 테스트 도구이며, 공인시험 환경에서도 활용되고 있습니다.

---

## 2. 테스트 구성

테스트를 위해서 최신 버전인 Apache JMeter 5.6.3 바이너리 파일을 다운로드 받아서 설치했습니다.

<img class="sub-image" src="/assets/images/jmeter_app.png" alt="Apache JMeter 실행화면">


그리고 Test Plan을 다음과 같이 구성했습니다.

```nohighlight
Test Plan
 └─ Thread Group
     ├─ HTTP Header Manager
     ├─ HTTP Request
     ├─ Response Assertion
     └─ Summary Report
```

---

### 2.1. Thread Group

* Number Of Thread: 10
* Ramp-up Period: 10
* Loop Count: 1000

> Ramp-up Period는 설정된 Thread를 한 번에 실행하지 않고 일정 시간에 걸쳐 점진적으로 증가시키는 역할을 합니다. 이를 통해 순간적인 부하 집중을 방지하고 보다 현실적인 부하 발생 상황을 구성할 수 있습니다.

Vault Transit Engine의 동시 요청 증가에 따른 응답시간 변화와 처리량 특성을 확인하고 싶었습니다.

개인 노트북에서는 Thread Group의 Number of Thread의 값을 10으로 설정해서 테스트를 했지만, 회사에 구축한 개발환경에서는 50, 100, 200, 500, 5000으로 단계적으로 증가시키면서 테스트를 수행했습니다.

---

### 2.2. HTTP Header Manager

* Content-Type: application/json
* X-Vault-Token: {Vault Token}

> X-Vault-Token에는 AppRole을 통해서 미리 발급받아 놓은 토큰을 입력하면 됩니다.

---

### 2.3. HTTP Request

**Basic**

* Protocol: http
* Server Name of IP: 192.168.56.101
* Port Number: 8200
* Method: POST
* Path: /v1/transit/encrypt/test-key

> test-key는 Transit Engine에 미리 생성해둔 암호화 키 이름입니다.

**Body Data**

```json
{
  "plaintext": "MDEwMTIzNDU2Nzg="
}
```

> "MDEwMTIzNDU2Nzg="는 01012345678을 Base64 인코딩한 값입니다.

---

### 2.4. Response Assertion

* Field to Test: Response Code
* Pattern Matching Rules: Equals
* Patterns to Test: 200

---

## 3. 테스트 실행 방법

Apache JMeter는 GUI(Graphical User Interface) 모드와 CLI(Command Line Interface) 모드로 테스트를 수행할 수 있습니다.

GUI 모드는 화면을 통해 Test Plan을 구성하고, 결과를 실시간으로 확인할 수 있기 때문에 테스트 시나리오를 작성하거나 동작을 검증할 때 주로 사용됩니다.

반면 CLI 모드는 별도의 화면 출력 없이 명령어 기반으로 테스트를 수행하는 방식입니다. 대량의 부하를 발생시키는 실제 성능 테스트는 일반적으로 CLI 모드에서 실행하는 것이 권장됩니다.

GUI 모드에서는 테스트 실행과 함께 화면 렌더링 및 결과 수집을 위한 추가적인 CPU 및 메모리 자원이 사용될 수 있고, 이로 인해 실제 TPS 및 응답시간 측정 결과가 왜곡될 가능성이 존재하기 때문입니다.

---

### 3.1. GUI 모드

Test Plan을 구성하고, GUI 모드에서는 상단 메뉴에 있는 Run 버튼을 클릭해서 테스트를 실행할 수 있습니다. 

다음은 제 개인 노트북에서 테스트를 실행한 결과를 캡쳐한 화면입니다.

<img class="sub-image" src="/assets/images/summaryreport.png" alt="Apache JMeter Summary Test">

Summary Report에 Sample이 10000이고, Throughput(TPS)는 1061.9로 측정되어 있습니다.

---

### 3.2. CLI 모드

CLI 모드에서 테스트를 수행하려면 터미널에서 다음과 같이 명령을 입력하면 됩니다.

```bash
$ jmeter -n -t {테스트 파일명(예:test.jmx)} -l result.jtl -e -o report
```

테스트가 수행되면 아래와 같은 메시지가 터미널에 출력됩니다. 또한, report 폴더에 웹 페이지 형식의 결과 보고서가 생성됩니다.

```nohighlight
C:\test>jmeter -n -t test.jmx -l result.jtl -e -o report
WARN StatusConsoleListener The use of package scanning to locate plugins is deprecated and will be removed in a future release
WARN StatusConsoleListener The use of package scanning to locate plugins is deprecated and will be removed in a future release
WARN StatusConsoleListener The use of package scanning to locate plugins is deprecated and will be removed in a future release
WARN StatusConsoleListener The use of package scanning to locate plugins is deprecated and will be removed in a future release
Creating summariser <summary>
Created the tree successfully using test.jmx
Starting standalone test @ 2026 May 16 14:44:28 GMT+09:00 (1778910268756)
Waiting for possible Shutdown/StopTestNow/HeapDump/ThreadDump message on port 4445
summary +   1238 in 00:00:01 = 1059.0/s Avg:     0 Min:     0 Max:    21 Err:     0 (0.00%) Active: 1 Started: 2 Finished: 1
summary +   8762 in 00:00:08 = 1048.6/s Avg:     0 Min:     0 Max:     6 Err:     0 (0.00%) Active: 0 Started: 10 Finished: 10
summary =  10000 in 00:00:10 = 1049.8/s Avg:     0 Min:     0 Max:    21 Err:     0 (0.00%)
Tidying up ...    @ 2026 May 16 14:44:38 GMT+09:00 (1778910278357)
... end of run
```

> 출력 결과에 평균 응답시간이 0ms로 표시되는 것은 실제 응답시간이 존재하지 않는다는 의미가 아닙니다. 응답속도가 매우 짧아 ms 단위에서 반올림되어 출력된 결과입니다.

---

## 4. 테스트 결과의 해석

테스트를 수행하는 것도 중요하지만 결과를 해석하는 것도 매우 중요합니다.

특히 암호화 연산을 외부 시스템에서 처리하는 구조에서는 일부 요청의 지연이 전체 서비스 응답성에 영향을 줄 수 있기 때문에, 평균 응답시간보다 Percentile 기반의 응답 특성을 함께 확인해야 합니다.

Percentile 기반의 응답 특성은 결과 리포트의 Statistics 항목에서는 90th pct, 95th pct, 99th pct와 같은 응답시간 분포 지표를 통해 확인할 수 있습니다.

> 90th pct는 전체 요청 중 90%의 요청이 해당 응답시간 이하로 처리되었음을 의미합니다. 95th pct 역시 전체 요청의 95%가 해당 시간 이하로 처리되었다는 의미입니다.

실제 운영 환경에서는 평균 응답시간보다 95th pct를 더 중요하게 보는 경우가 많습니다. 일부 요청만 매우 느려지더라도 사용자 입장에서는 성능 저하로 체감될 수 있기 때문입니다.

이번 테스트를 통해 Vault Transit Engine이 생각보다 안정적인 응답 특성과 처리량을 제공한다는 점을 확인할 수 있었습니다.

기대했던 것 이상의 TPS와 응답 특성을 보여주었으며, Thread 증가 상황에서도 안정적으로 동작했습니다.

물론 실제 운영 환경에서는 네트워크 구조, CPU 자원, 동시 사용자 수, 암호화 데이터 크기 등에 따라 결과가 달라질 수 있기 때문에 추가적인 검증이 필요할 수 있습니다.

하지만 이번 테스트를 통해 Vault Transit Engine 기반의 암호화 연산 외부화가 충분히 현실적인 선택지가 될 수 있다는 가능성을 확인할 수 있었습니다.

---

## 5. 관련 포스팅

* [[학습기록] Vault를 이용한 KMS 대체 검토: (1) 개념정리와 설치과정](https://sanghoon-lee.github.io/2026/04/09/vault//)
* [[학습기록] Vault를 이용한 KMS 대체 검토: (2) Transit Engine을 이용한 암·복호화 실습](https://sanghoon-lee.github.io/2026/04/11/vault2/)
* [[학습기록] Vault를 이용한 KMS 대체 검토: (3) Java 애플리케이션에서 Vault 연동하기](https://sanghoon-lee.github.io/2026/04/18/vault3/)

---

#JMeter #부하테스트 #성능테스트 #HashiCorpVault #Vault #TPS #TransitEngine 