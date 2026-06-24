---
layout: post
title: "[학습기록] Vault를 이용한 KMS 대체 검토: (2) KV Engine을 이용한 암호화 키 저장과 조회 실습"
date: 2026-04-10
description: HashiCorp Vault의 KV Engine에 대한 개념을 공부하고, 이를 이용해서 암호화 키 저장과 조회 실습하는 과정을 정리한 글입니다. 원래 포스팅했던 내용을 나중에 리라이팅하면서 추가되었습니다.
image: /assets/images/hashicorp.jpg
categories: 학습기록
tags:
  - Vault
  - HashiCorp
  - KMS
  - Security
  - Docker
---

보안 관점에서는 암호화 키를 애플리케이션으로 전달하지 않고, 애플리케이션과 분리된 외부 시스템에서 암·복호화를 수행하는 방식이 더 권장됩니다.

하지만 KMS를 암호화 키를 보관하는 중앙 저장소로만 활용하는 경우도 있습니다.

애플리케이션이 필요한 시점에 KMS에서 암호화 키를 조회한 뒤, 내부에서 직접 암·복호화 연산을 수행하는 방식입니다. 구조가 단순하고 사용하기 쉬운 장점은 있습니다. 하지만, 암호화 키가 애플리케이션 영역으로 전달되기 때문에 노출 위험이 존재합니다.

Vault의 KV Engine은 다양한 형태의 시크릿을 저장하고 관리하는 기능을 제공합니다.

암호화 키 역시 시크릿의 한 유형에 해당됩니다. 따라서 KV Engine을 활용하면 KMS의 암호화 키 저장소 역할은 충분히 대체할 수 있을 것으로 판단했습니다.

---

## 1. KV Engine

KV(Key-Value) Engine은 Vault에서 가장 기본적으로 제공하는 시크릿 저장소입니다.

이름 그대로 Key와 Value 형태로 데이터를 저장하고 조회할 수 있으며, 하나의 경로(Path) 아래에 여러 하위 속성을 저장할 수도 있습니다.

암호화 키뿐만 아니라 API Key·DB 계정 정보·인증서와 같은 다양한 시크릿을 관리하는 데 사용할 수 있습니다.

예를 들어 다음과 같은 형태로 데이터를 저장할 수 있습니다.

```json
{
  "username": "admin",
  "password": "password123"
}
```

물론, 암호화 키를 저장하는 용도로도 활용할 수 있습니다.

```json
{
  "key": "testkeytestkey00"
}
```

KV Engine은 버전 관리와 접근 제어 기능도 함께 제공하기 때문에, 중앙 집중형 시크릿 관리 저장소로 활용할 수 있습니다.

따라서 암호화 키 저장소를 구현하는 데에도 적합한 기능이라고 판단했습니다.

---

## 2. 실습: 암호화 키 저장과 조회

지금까지 KV Engine의 역할과 동작 방식을 개념적으로 살펴봤습니다.

이제 직접 암호화 키를 저장하고, 조회해보도록 하겠습니다.

실습을 통해 KV Engine의 시크릿 관리 기능을 활용하면 암호화 키의 저장과 조회가 가능하다는 점을 확인할 수 있습니다.

---

### 2.1. KV Engine 활성화

KV Engine을 사용하기 위해서는 먼저 엔진을 활성화해야 합니다. 

KV Engine에는 Version 1과 Version 2가 존재합니다. Version 1은 시크릿 저장·조회 기능을 제공하며, Version 2는 시크릿의 버전 관리 기능까지 추가로 제공합니다. 그래서 운영 환경에서는 일반적으로 Version 2 사용이 권장됩니다.

엔진을 활성화하려면 Vault CLI를 사용해야 합니다. 지난 포스팅에서 이야기했던 것처럼 호스트에는 CLI가 설치되어 있지 않기 때문에, 컨테이너 내부에 다시 접속했습니다. 

```bash
$ sudo docker exec -it vault sh
```

> 이후 실습도 모두 컨테이너 내부의 CLI를 사용해서 진행되었습니다.

컨테이너 내부에 접속한 상태에서 아래 명령으로 KV Version 2 Engine을 활성화하려고 했습니다.

```nohighlight
/ # vault secrets enable -path=keys kv-v2
```

하지만 기대하지 않았던 오류가 발생했습니다. 오류 내용은 아래와 같았습니다.

```nohighlight
Error enabling: Error making API request.

URL: POST http://127.0.0.1:8200/v1/sys/mounts/keys
Code: 403. Errors:

* permission denied
```

Vault CLI는 기본적으로 VAULT_TOKEN 환경변수에 설정된 인증 토큰을 사용합니다. 개발 모드로 실행할 때 root 토큰은 생성되지만, Vault CLI가 자동으로 해당 값을 환경변수에 설정해주지는 않습니다. 따라서 별도로 VAULT_TOKEN 환경변수를 설정하지 않으면 인증에 실패하게 됩니다.

아래와 같이 root 토큰을 환경변수의 값으로 설정해야 합니다.

```nohighlight
/ # export VAULT_TOKEN=root
```

이후 다시 명령을 실행해서 정상적으로 KV Engine을 활성화할 수 있었습니다.

```nohighlight
/ # vault secrets enable -path=keys kv-v2
Success! Enabled the kv-v2 secrets engine at: keys/
```

KV Engine을 활성화하면서 -path=keys 옵션을 사용했습니다. 이 옵션은 KV Engine이 사용할 경로(Path)를 지정하는 역할을 합니다. 이후 시크릿은 keys/ 경로 아래에 저장하고 조회할 수 있습니다.

---

### 2.2. 암호화 키 저장

이제 실제로 암호화 키를 저장해보겠습니다. KV Version 2 Engine은 JSON 형태의 데이터를 저장할 수 있습니다. 따라서 암호화 키도 일반적인 시크릿 정보와 동일한 방식으로 저장하면 됩니다.

실습에서는 아래와 같이 암호화 키를 저장해봤습니다. 시크릿 경로(키 이름)는 `coupon`이며, `testkeytestkey00`라는 암호화 키 값과 `seed`라는 알고리즘 정보를 함께 저장했습니다.

```nohighlight
/ # vault kv put keys/coupon key=testkeytestkey00 algorithm=seed
```

명령이 정상적으로 수행되고, 다음과 같은 응답을 확인할 수 있었습니다.

```nohighlight
== Secret Path ==
keys/data/coupon

======= Metadata =======
Key                Value
---                -----
created_time       2026-06-21T13:13:36.022399488Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

응답 결과를 보면 암호화 키와 관련 정보를 포함한 시크릿이 정상적으로 저장되었으며 현재 버전이 1인 것을 확인할 수 있습니다. 

KV Version 2 Engine은 시크릿을 수정할 때마다 새로운 버전을 생성하기 때문에 변경 이력을 관리할 수 있습니다.

---

### 2.3. 암호화 키 조회

암호화 키가 정상적으로 저장되었는지 확인하기 위해 조회를 수행했습니다.

```nohighlight
/ # vault kv get keys/coupon
```

응답은 아래와 같았습니다. 정상적으로 암호화 키가 저장된 것을 확인할 수 있었습니다.

```nohighlight
== Secret Path ==
keys/data/coupon

======= Metadata =======
Key                Value
---                -----
created_time       2026-06-21T13:13:36.022399488Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

=== Data ===
Key         Value
---         -----
key         testkeytestkey00
algorithm   seed
```

응답 결과는 크게 Metadata 영역과 Data 영역으로 구분됩니다.

Metadata 영역에는 생성 시각과 버전 정보 등이 포함되어 있으며, Data 영역에는 실제 저장된 시크릿 정보가 포함되어 있습니다.

Data 영역에는 단순히 암호화 키 값뿐만 아니라, 암호화 알고리즘과 같은 부가 정보도 함께 저장할 수 있습니다. 따라서 암호화 키와 관련된 설정 정보를 하나의 시크릿으로 함께 관리할 수 있습니다.

---

## 3. 마무리

KV Engine은 암호화 키뿐만 아니라, 다양한 시크릿을 중앙에서 저장하고 관리할 수 있는 기능을 제공합니다.

실습을 통해 Vault의 KV Engine만으로도 암호화 키를 중앙에서 저장하고 관리하는 기능은 충분히 구현할 수 있다는 사실을 확인할 수 있었습니다.

다만 이 방식은 실무에서 권장되지는 않습니다. 암호화 키가 결국은 애플리케이션으로 전달되어 외부에 노출 위험이 존재하기 때문입니다.

보다 안전하게 암호화 키를 보관하려면 KMS가 암·복호화 연산까지 수행할 수 있어야 합니다. 그래서 Transit Engine의 사용 방법도 알아볼 필요가 있었습니다.

다음 포스팅에서 Vault의 Transit Engine을 이용해 암호화 키를 외부로 노출하지 않고 암·복호화를 수행해보도록 하겠습니다.

---

## 4. 포스팅 목록

* [[학습기록] Vault를 이용한 KMS 대체 검토: (1) 개념정리와 설치과정](https://sanghoon-lee.github.io/2026/04/09/vault/)
* [학습기록] Vault를 이용한 KMS 대체 검토: (2) KV Engine을 이용한 암호화 키 저장과 조회 실습
* [[학습기록] Vault를 이용한 KMS 대체 검토: (3) Transit Engine을 이용한 암·복호화 실습](https://sanghoon-lee.github.io/2026/04/11/vault2/)
* [[학습기록] Vault를 이용한 KMS 대체 검토: (4) Java 애플리케이션에서 Vault Transit Engine 연동하기](https://sanghoon-lee.github.io/2026/04/18/vault3/)

