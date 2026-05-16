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

<img class="main-image" src="/assets/images/jmeter.png" alt="Apache JMeter">

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

## 3. 테스트 실행

### 3.1. GUI에서 테스트 실행

지금까지 구성된 테스트를 실행해서 정상적으로 동작하는 것을 확인했습니다. 

<img class="sub-image" src="/assets/images/summaryreport.png" alt="Apache JMeter Summary Test">

Summary Report를 보면 테스트에 사용된 Sample은 10000이고, Throughput(TPS)는 1061.9로 측정된 것을 알 수 있습니다.

---

### 3.2. CLI에서 테스트 실행

GUI에서 테스트가 실행되면 실시간으로 화면에 결과를 출력하고, Listener를 지속적으로 갱신하는 과정에서 추가적인 CPU 및 메모리 자원이 사용됩니다.

특히 대량의 부하를 발생시키는 환경에서는 GUI 렌더링 및 결과 수집 과정 자체가 성능에 영향을 줄 수 있습니다.

따라서 실제 TPS 및 응답시간 측정 결과가 왜곡될 가능성이 존재합니다.

그래서 GUI보다는 CLI에서 테스트를 수행하는 것이 일반적으로 권장됩니다.

CLI에서 테스트를 수행하려면 터미널에서 다음과 같이 명령을 입력하면 됩니다.

```bash
$ jmeter -n -t {테스트 파일명(예:test.jmx)} -l result.jtl -e -o report
```

다음은 제 개인 노트북에서 테스트를 실행한 결과입니다.

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

평균 응답시간이 0ms로 표시되는 것은 실제 응답시간이 존재하지 않는다는 의미가 아니라, 응답속도가 매우 짧아 ms 단위에서 반올림되어 출력된 결과입니다.

---

### 3.3. Thread 증가 기반 부하 테스트

Vault Transit Engine은 애플리케이션 외부에서 암호화 연산을 수행하는 구조이기 때문에, 동시 요청 증가에 따른 응답시간 변화와 처리량 특성을 확인하는 것이 중요했습니다.

이를 위해 JMeter의 Thread 수를 단계적으로 증가시키면서 여러 차례 부하 테스트를 수행하였고, 각 단계별 응답시간과 TPS 변화를 비교해보았습니다.

---

## 4. 테스트 결과 보고서

CLI에서 테스트가 완료되면, report 폴더에 웹 페이지 형식의 결과 보고서가 생성됩니다.

테스트를 수행하는 것도 중요하지만 결과를 해석하는 것도 매우 중요합니다.

특히 암호화 연산을 외부 시스템에서 처리하는 구조에서는 일부 요청의 지연이 전체 서비스 응답성에 영향을 줄 수 있기 때문에, 평균 응답시간보다 Percentile 기반의 응답 특성을 함께 확인해야 합니다.

결과 리포트의 Statistics 항목에서는 90th pct, 95th pct, 99th pct와 같은 응답시간 분포 지표를 제공하는데, 이는 전체 요청 중 대부분의 요청이 어느 수준의 응답시간 안에서 처리되는지를 보여줍니다.

90th pct는 전체 요청 중 90%의 요청이 해당 응답시간 이하로 처리되었음을 의미합니다. 95th pct 역시 전체 요청의 95%가 해당 시간 이하로 처리되었다는 의미입니다.

실제 운영 환경에서는 평균 응답시간보다 95th pct를 더 중요하게 보는 경우가 많습니다. 일부 요청만 매우 느려지더라도 사용자 입장에서는 성능 저하로 체감될 수 있기 때문입니다.

---

## 5. 마무리

이번 테스트를 통해 Vault Transit Engine이 생각보다 안정적인 응답 특성과 처리량을 제공한다는 점을 확인할 수 있었습니다.

특히 암호화 연산을 애플리케이션 외부 시스템으로 분리하는 구조에서도 충분한 수준의 TPS와 응답시간 특성을 보여주었으며, Thread 증가 상황에서도 비교적 안정적인 동작을 확인할 수 있었습니다.

물론 실제 운영 환경에서는 네트워크 구조, CPU 자원, 동시 사용자 수, 암호화 데이터 크기 등에 따라 결과가 달라질 수 있기 때문에 추가적인 검증이 필요할 수 있습니다.

하지만 이번 테스트를 통해 Vault Transit Engine 기반의 암호화 연산 외부화 구조가 충분히 현실적인 선택지가 될 수 있다는 가능성을 확인할 수 있었습니다.

---

#JMeter #부하테스트 #성능테스트 #vault #TPS #transitengine