---
layout: post
title: "[토이프로젝트] Vault를 이용한 KMS 대체 검토: (2) KV Engine을 이용한 암호화 키 저장과 조회 실습"
date: 2026-04-10
series: vault-kms
description: HashiCorp Vault의 KV Secrets Engine을 이용해 암호화 키를 저장하고 조회하는 방법을 실습하고, 암호화 키 저장소로 활용할 수 있는지 검토한 과정을 정리한 글입니다.
image: /assets/images/hashicorp.jpg
categories: 토이프로젝트
tags:
  - 보안
  - 인프라
  - 자바
---

실제 운영 환경에서는 KMS를 암호화 키를 보관하는 중앙 저장소로만 활용하는 경우도 있습니다. 애플리케이션이 필요한 시점에 KMS에서 암호화 키를 조회한 뒤, 내부에서 직접 암·복호화 연산을 수행하는 방식입니다. 구조가 단순하고 사용하기 쉬운 장점은 있지만, 암호화 키가 애플리케이션 영역으로 전달되기 때문에 노출 위험이 존재합니다.

암호화 키의 노출 지점을 줄이려면 키를 애플리케이션에 전달하지 않고, 애플리케이션과 분리된 외부 시스템에서 암·복호화 연산을 수행하는 방식이 유리합니다.

Vault의 KV Secrets Engine(이하, "KV Engine")은 다양한 형태의 시크릿을 저장하고 관리하는 기능을 제공합니다. 암호화 키 역시 시크릿의 한 유형에 해당됩니다. 따라서 KV Engine을 암호화 키 저장소로 활용할 수 있는지 직접 확인해보기로 했습니다.

---

## 1. KV Engine

KV(Key-Value) Engine은 Vault에서 가장 기본적으로 제공하는 시크릿 저장소입니다.

이름 그대로 키와 값(Key-Value) 형태로 데이터를 저장하고 조회할 수 있으며, 하나의 경로(Path)에 여러 필드를 함께 저장할 수도 있습니다.

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
  "key": "testkeytestkey00",
  "algorithm": "SEED"
}
```

KV Engine은 버전 관리와 Vault의 정책 기반 접근 제어를 함께 활용할 수 있어 중앙 집중형 암호화 키 저장소로 활용할 수 있습니다.

---

## 2. 실습: 암호화 키 저장과 조회

지금까지 KV Engine의 역할을 개념적으로 살펴봤습니다. 이제 KV Engine을 활성화하고 암호화 키를 직접 저장한 뒤 정상적으로 조회되는지 확인해보겠습니다.

---

### 2.1. KV Engine 활성화

KV Engine에는 `KV v1`과 `KV v2`가 존재합니다. 

* `KV v1`: 시크릿 저장·조회 기능을 제공합니다.
* `KV v2`: 시크릿을 수정할 때마다 새로운 데이터 버전을 생성합니다. 

띠리사 이전 버전의 값을 조회하거나 잘못 변경된 값을 복구해야 하는 경우에는 `KV v2`가 적합합니다.

KV Engine을 사용하려면 먼저 엔진을 활성화해야 합니다. 

엔진을 활성화하려면 Vault CLI를 사용해야 하지만, 지난 포스팅에서 이야기했던 것처럼 호스트에는 CLI가 설치되어 있지 않습니다. 따라서 CLI를 사용하기 위해 컨테이너에 접속해야 했습니다. 

```bash
$ sudo docker exec -it vault sh
```

> 이후 실습도 모두 컨테이너에 접속한 상태에서 Vault CLI를 사용해서 진행되었습니다.

컨테이너에 접속한 상태에서 아래 명령으로 `KV v2` 활성화를 시도했습니다.

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

Vault CLI는 기본적으로 `VAULT_TOKEN` 환경변수에 설정된 인증 토큰을 사용합니다. 개발 모드로 실행할 때 root 토큰은 생성되지만, Vault CLI가 자동으로 해당 값을 환경변수에 설정해주지는 않습니다. 따라서 `VAULT_TOKEN`을 설정하거나 `vault login` 명령으로 로그인하지 않으면 권한이 필요한 명령을 실행할 수 없습니다.

이번 실습에서는 아래와 같이 root 토큰을 `VAULT_TOKEN` 환경변수에 설정했습니다.

```nohighlight
/ # export VAULT_TOKEN=root
```

이후 다시 명령을 실행해서 정상적으로 KV Engine을 활성화할 수 있었습니다.

```nohighlight
/ # vault secrets enable -path=keys kv-v2
Success! Enabled the kv-v2 secrets engine at: keys/
```

KV Engine을 활성화하면서 `-path=keys` 옵션을 사용했습니다. 이 옵션은 Secrets Engine 자체가 마운트될 경로를 의미합니다. 이후 시크릿은 `keys/` 경로 아래에 저장하고 조회할 수 있습니다.

---

### 2.2. 암호화 키 저장

이제 실제로 암호화 키를 저장해보겠습니다. 암호화 키도 일반적인 시크릿 정보와 동일한 방식으로 저장하면 됩니다.

`KV v2`는 JSON 형태의 데이터를 저장할 수 있습니다. CLI에서는 `key=value` 형식으로 데이터를 입력하며, Vault API에서는 이를 JSON 형태의 데이터로 처리합니다.

실습에서는 `keys/coupon` 경로에 `testkeytestkey00`이라는 암호화 키 값과 알고리즘 정보(SEED)를 함께 저장했습니다. 여기서 `keys/`는 KV Engine이 마운트된 경로이고, `coupon`은 시크릿 경로입니다.

```nohighlight
/ # vault kv put keys/coupon key=testkeytestkey00 algorithm=SEED
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

응답 결과를 보면 암호화 키와 관련 정보를 포함한 시크릿이 정상적으로 저장되었으며 현재 버전이 1인 것을 확인할 수 있습니다. `KV v2`는 시크릿을 수정할 때마다 새로운 버전을 생성하므로, 이전 버전의 값을 조회하거나 복구할 수 있습니다.

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
algorithm   SEED
```

응답 결과는 크게 `Metadata` 영역과 `Data` 영역으로 구분됩니다.

`Metadata` 영역에는 시크릿의 생성 시각과 데이터 버전 등의 관리 정보가 포함되어 있으며, `Data` 영역에는 실제로 저장한 암호화 키와 알고리즘 정보가 포함되어 있습니다.

이처럼 하나의 시크릿 경로에 암호화 키와 알고리즘 등의 관련 정보를 함께 저장할 수 있으며, `KV v2`가 관리하는 버전 정보는 `Metadata` 영역에서 별도로 확인할 수 있습니다.

---

## 3. 마무리

KV Engine은 암호화 키뿐만 아니라, 다양한 시크릿을 중앙에서 저장하고 관리할 수 있는 기능을 제공합니다.

실습을 통해 Vault의 KV Engine이 암호화 키 저장소 역할을 수행할 수 있음을 확인했습니다. 다만 이 방식에서는 애플리케이션이 암호화 키를 직접 조회하고 메모리에 보유해야 하므로, 키가 노출될 수 있는 지점을 완전히 제거할 수는 없습니다.

암호화 키를 애플리케이션에 전달하지 않으려면 Vault와 같은 외부 시스템이 암·복호화 연산까지 수행해야 합니다. Vault는 이를 위해 Transit Secrets Engine을 제공합니다.

다음 포스팅에서 Vault의 Transit Secrets Engine을 이용해 암호화 키를 외부로 노출하지 않고 암·복호화를 수행해보도록 하겠습니다.