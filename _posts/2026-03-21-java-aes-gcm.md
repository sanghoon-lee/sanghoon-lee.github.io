---
layout: post
title: "[학습기록] Java AES-GCM 구현 시 실수하기 쉬운 4가지"
date: 2026-03-21
description: Java에서 AES-GCM을 구현할 때 주의해야 할 Nonce 재사용, 인증 태그 검증, 권장 길이와 Payload 구성 방법을 정리합니다.
image: /assets/images/lock.png
categories: 학습기록
tags:
  - 보안
  - 자바
---

[[학습기록] AES 암호화 모드 비교: ECB, CBC, GCM 차이와 GCM을 사용하는 이유](https://sanghoon-lee.github.io/2026/03/21/aes-gcm/)을 포스팅하면서 실제 보안 수준은 알고리즘뿐 아니라 어떤 암호화 모드를 선택하느냐에 따라 달라진다는 점을 살펴봤습니다.

하지만 안전한 암호화 모드를 선택했다고 해서 구현까지 자동으로 안전해지는 것은 아닙니다. Nonce를 재사용하거나 인증 태그 검증 실패를 잘못 처리하면 심각한 보안 문제가 발생할 수 있기 때문입니다.

그래서 이번 포스팅을 통해 Java에서 `AES-GCM`을 구현할 때 실수하기 쉬운 네 가지 사례와 안전하게 사용하는 방법을 정리해봤습니다.

---

## 1. 사례 1: 같은 키에서 Nonce 재사용

같은 키에서 동일한 Nonce가 재사용되면 동일한 키스트림이 생성될 수 있으며, 암호문 사이의 관계를 이용해 평문 정보를 유추하거나 인증 태그를 위조할 가능성이 생깁니다.

일반적인 애플리케이션에서는 12바이트 Nonce를 `SecureRandom`으로 매번 새롭게 생성하는 방식을 사용할 수 있습니다. 

**올바른 사용법**
```java
byte[] nonce = new byte[12];
new SecureRandom().nextBytes(nonce);
```

**잘못된 사용법**
```java
byte[] nonce = new byte[12]; // 항상 동일한 값
```

다만 여러 서버가 같은 키를 사용하는 분산 환경에서는 난수 생성만으로 끝내지 않고, 전체 시스템에서 Nonce가 중복되지 않도록 생성·관리 방식을 함께 설계해야 합니다.

---

## 2. 사례 2: AEADBadTagException 무시

`GCM` 모드에서는 복호화 과정에서 인증 태그 검증이 자동으로 수행됩니다. 

Java에서는 복호화 과정에서 인증 태그 검증에 실패하면 `AEADBadTagException`이 발생합니다. 이는 암호문이나 인증 태그가 변경되었거나, 잘못된 키 또는 Nonce를 사용했거나, 저장·전송 과정에서 데이터가 손상되었을 가능성을 의미합니다.

따라서 이 예외를 무시하거나 해당 데이터를 정상적인 복호화 결과처럼 처리해서는 안 됩니다.


**올바른 사용법**
```java
decryptCipher.init(Cipher.DECRYPT_MODE, key, spec);
try {
    return decryptCipher.doFinal(ciphertext);
} catch (AEADBadTagException e) {
    throw new SecurityException("인증 태그 검증에 실패했습니다.", e);    
}
```

**참고**

> 인증 태그는 암호문이 변경되지 않았는지 검증하고, 올바른 키와 파라미터로 생성된 데이터인지 확인하는 데 사용됩니다.

---

## 3. 사례 3: Nonce와 인증 태그 길이 임의 변경

`AES-GCM`에서는 특별한 요구사항이 없다면 96비트(12바이트) Nonce와 128비트 인증 태그를 사용하는 것이 일반적입니다.

96비트 Nonce는 `GCM`에서 권장되는 길이이며, 별도의 복잡한 변환 없이 초기 카운터 블록을 구성할 수 있습니다. 또한 인증 태그 길이를 임의로 줄이지 않고 128비트를 사용하는 것은 가장 호환성이 좋고 안전한 선택입니다.

```java
private static final int NONCE_LENGTH_BYTES = 12;
private static final int TAG_LENGTH_BITS = 128;

GCMParameterSpec spec = new GCMParameterSpec(TAG_LENGTH_BITS, nonce);
```

---

## 4. 사례 4: 암호화 결과만 저장하고 Nonce 누락

Java에서 `GCM` 모드로 암호화 연산을 수행하면 `cipher.doFinal()`의 결과에 **암호문과 인증 태그가 함께 포함**됩니다.

즉, 반환되는 데이터는 다음과 같은 구조를 가집니다.

```text
[ ciphertext | authentication tag ]
```

하지만 이 결과에는 Nonce가 포함되어 있지 않습니다. 복호화를 수행하기 위해서는 동일한 Nonce가 반드시 필요하기 때문에 실무에서는 다음과 같이 Nonce를 함께 저장하거나 전송해야 합니다. 이 구조는 Java가 자동으로 만들어주는 것이 아니라, 복호화를 위해 개발자가 직접 구성해야 하는 데이터 포맷입니다.

```text
[ nonce | ciphertext | authentication tag ]
```

운영 환경에서는 향후 키 교체와 데이터 포맷 변경을 고려해 포맷 버전과 키 식별자를 함께 저장하는 구조도 고려할 수 있습니다.

```text
[ version | keyId | nonce | ciphertext | authentication tag ]
```

**참고**

> Nonce는 비밀값이 아니므로 암호문과 함께 저장하거나 전송해도 됩니다. 중요한 것은 외부에 숨기는 것이 아니라, 같은 키에서 중복되지 않도록 생성하고 복호화할 때 정확히 같은 값을 사용하는 것입니다.

---

## 5. 마무리

`AES-GCM`은 데이터의 기밀성과 무결성을 함께 제공하는 안전한 암호화 방식이지만, 사용 방법이 잘못되면 이러한 장점을 제대로 활용할 수 없습니다.

같은 키에서 Nonce가 중복되지 않도록 관리하고, 인증 태그 검증 실패를 정상적인 복호화 결과처럼 처리해서는 안 됩니다. 또한 권장되는 Nonce와 인증 태그 길이를 사용하고, 복호화에 필요한 Nonce를 암호문과 함께 저장해야 합니다.

Java에서 `Cipher.doFinal()`을 호출하면 반환되는 결과에는 암호문과 인증 태그가 함께 포함됩니다. Nonce는 포함되지 않는다는 점을 주의해야 합니다. 

운영 환경에서는 키 교체와 데이터 포맷 변경까지 고려해 데이터 포맷 버전과 키 식별자를 함께 관리하는 것이 바람직합니다. 




