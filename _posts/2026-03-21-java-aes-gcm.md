---
layout: post
title: "[학습기록] Java에서 AES-GCM 안전하게 사용하는 방법: 실수하기 쉬운 4가지"
date: 2026-03-21
description: Java를 기준으로 AES-GCM을 안전하게 사용하는 방법을 정리한 포스팅입니다.
image: /assets/images/lock.png
categories: 학습기록
---

AES-GCM 모드는 강력한 암호화 방식입니다. 하지만 잘못된 사용은 심각한 보안 취약점으로 이어질 수 있습니다.

가장 중요한 것은 올바르게 사용하는 것입니다.

---

## 1. 기본 구조

AES-GCM 모드는 다음과 같은 요소로 구성됩니다.

* Key: 암호화에 사용하는 비밀키
* Nonce(Initial Value for Counter): 한 번만 사용하는 초기값
* Plaintext: 암호화 대상 데이터(평문)
* Ciphertext: 암호화 결과(암호문)
* Authentication Tag: 무결성 검증 값

암호화와 동시에 무결성까지 검증할 수 있는 AEAD 방식이기 때문에, 복호화 과정에서 데이터가 변경되었는지도 함께 확인할 수 있습니다.

---

## 2. 예제 코드

다음은 AES-GCM 암호화 모드를 사용하는 방법을 보여주기 위해 Java로 작성한 예제 코드입니다.

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;
import java.util.Arrays;
import javax.crypto.AEADBadTagException;
import java.nio.charset.StandardCharsets;

public class AesGcmExample {

    public static void main(String[] args) throws Exception {

        // 1. Key 생성
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        SecretKey key = keyGen.generateKey();

        // 2. Nonce 생성 (12 bytes 권장)
        byte[] nonce = new byte[12];
        SecureRandom random = new SecureRandom();
        random.nextBytes(nonce);

        // 3. Cipher 설정 (암호화)
        Cipher encryptCipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, nonce);
        encryptCipher.init(Cipher.ENCRYPT_MODE, key, spec);

        // 4. 암호화
        byte[] plaintext = "Hello AES-GCM".getBytes(StandardCharsets.UTF_8);
        byte[] ciphertext = encryptCipher.doFinal(plaintext);

        System.out.println("Encrypted: " + Arrays.toString(ciphertext));

        // =========================
        // 복호화
        // =========================

        Cipher decryptCipher = Cipher.getInstance("AES/GCM/NoPadding");
        decryptCipher.init(Cipher.DECRYPT_MODE, key, spec);

        try {
            byte[] decrypted = decryptCipher.doFinal(ciphertext);
            System.out.println("Decrypted: " + new String(decrypted, StandardCharsets.UTF_8));

        } catch (AEADBadTagException e) {
            // 인증 태그 검증 실패 (데이터 변조)
            System.out.println("데이터가 변조되었습니다. 복호화 실패");

        } catch (Exception e) {
            // 기타 오류
            System.out.println("복호화 중 오류 발생: " + e.getMessage());
        }
    }
}
```

---

## 3. 주의사항

AES-GCM 모드를 언잔하게 사용하기 위해서 다음 세 가지는 반드시 지켜야 합니다.

* Nonce 재사용 금지
* 인증 태그 검증 필수
* 권장되는 Nonce 길이(12 bytes)와 인증 태그 길이(128 bit) 사용

---

### 3.1. Nonce 재사용 금지

같은 키에서 동일한 Nonce를 절대 재사용하면 안 됩니다.

Nonce가 재사용되면 동일한 키스트림이 생성되며, 암호문을 통해 평문 간의 관계가 노출될 수 있습니다.

이는 암호화 자체가 무력화되는 수준의 심각한 문제입니다.

**올바른 사용법**
```java
byte[] nonce = new byte[12];
new SecureRandom().nextBytes(nonce);
```

**잘못된 사용법**
```java
byte[] nonce = new byte[12]; // 항상 동일한 값
```

---

### 3.2. 인증 태그 검증 필수

AES-GCM 모드의 복호화 과정에서 인증 태그 검증이 자동으로 수행됩니다.

> AES-GCM에서 생성되는 인증 태그는 데이터가 암호화된 이후 변경되지 않았음을 검증하기 위한 값입니다.

Java에서는 복호화 과정에서 데이터 변조가 감지되면 **AEADBadTagException**이 발생합니다. 

이 예외가 발생하면 신뢰할 수 없는 데이터에 해당되기 때문에, 무시하거나 복호화를 계속 진행해서는 안 됩니다.

**올바른 사용법**
```java
decryptCipher.init(Cipher.DECRYPT_MODE, key, spec);
try {
    byte[] decrypted = decryptCipher.doFinal(ciphertext);
        
    ...

} catch (AEADBadTagException e) {
        
    ...

}
```

**잘못된 사용법**

```java
// 예외를 처리하지 않으면 변조 여부를 구분하지 못하고, 상위 로직에서 오류를 부적절하게 처리할 수 있음
cipher.init(Cipher.DECRYPT_MODE, key, spec);
byte[] decrypted = cipher.doFinal(ciphertext);
```

예제 코드(AesGcmExample 클래스)의 코드 일부를 아래와 같이 수정해서 변조된 암호문을 복호화하면 **AEADBadTagException** 예외가 발생하는 것을 확인할 수 있습니다.

**암호문 변조**
```java

    ...
        // 4. 암호화
        byte[] plaintext = "Hello AES-GCM".getBytes(StandardCharsets.UTF_8);
        byte[] ciphertext = encryptCipher.doFinal(plaintext);

        System.out.println("Encrypted: " + Arrays.toString(ciphertext));

        // =========================
        // 복호화
        // =========================

        ciphertext[0] ^= 1;  // 암호문 일부 변조 (테스트)

        Cipher decryptCipher = Cipher.getInstance("AES/GCM/NoPadding");
        decryptCipher.init(Cipher.DECRYPT_MODE, key, spec);
    ...

```

---

### 3.3. 권장되는 Nonce 길이(12 bytes)와 인증 태그 길이(128 bit) 사용

AES-GCM 모드에서는 특별한 이유가 없다면 96bit(12 bytes) 길이의 Nonce와 128bit 길이의 인증 태그 사용이 권장됩니다. 

그 이유는 다음과 같습니다.

* 충분한 보안성 제공 (위변조 공격에 대한 강한 내성)
* 대부분의 라이브러리와 프로토콜에서 기본값으로 사용
* 성능 대비 보안 수준이 가장 적절

성능과 보안 측면에서 최적화된 길이이며, 대부분의 구현에서 표준으로 사용됩니다.

---

## 4. 참고: 인증 태그를 포함한 데이터 구조

Java에서 AES-GCM으로 암호화를 수행하면, `cipher.doFinal()`의 결과에는 **암호문과 인증 태그가 함께 포함**됩니다.

즉, 반환되는 데이터는 다음과 같은 구조를 가집니다.

```text
[ ciphertext | authentication tag ]
```

다만, 이 결과에는 Nonce가 포함되어 있지 않습니다.

복호화를 수행하기 위해서는 동일한 Nonce가 반드시 필요하기 때문에, 실무에서는 다음과 같이 Nonce를 함께 저장하거나 전송해야 합니다.

```text
[nonce | ciphertext | authentication tag]
```

즉, 이 구조는 Java가 자동으로 만들어주는 것이 아니라, 복호화를 위해 개발자가 직접 구성해야 하는 데이터 포맷입니다.

다음은 조금 더 실무적인 관점에서 AES-GCM을 어떻게 구현하는지 보여주는 예제 코드입니다.

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.Arrays;

public class AesGcmPayloadExample {

    public static void main(String[] args) throws Exception {
        // 1. AES Key 생성
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        SecretKey key = keyGen.generateKey();

        // 2. Nonce 생성 (12 bytes 권장)
        byte[] nonce = new byte[12];
        new SecureRandom().nextBytes(nonce);

        // 3. 평문
        byte[] plaintext = "Hello AES-GCM".getBytes(StandardCharsets.UTF_8);

        // 4. 암호화
        Cipher encryptCipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, nonce);
        encryptCipher.init(Cipher.ENCRYPT_MODE, key, spec);

        // doFinal() 결과 = [ciphertext | authentication tag]
        byte[] cipherTextWithTag = encryptCipher.doFinal(plaintext);

        System.out.println("nonce length = " + nonce.length);
        System.out.println("plaintext length = " + plaintext.length);
        System.out.println("cipherTextWithTag length = " + cipherTextWithTag.length);
        System.out.println("cipherTextWithTag = " + Arrays.toString(cipherTextWithTag));

        // 5. 실무용 저장/전송 포맷 구성
        // [nonce | ciphertext | tag]
        byte[] payload = ByteBuffer
                .allocate(nonce.length + cipherTextWithTag.length)
                .put(nonce)
                .put(cipherTextWithTag)
                .array();

        System.out.println("payload length = " + payload.length);
        System.out.println("payload = [nonce | ciphertext | tag]");

        // -----------------------------------------
        // 복호화
        // -----------------------------------------

        // 6. payload에서 nonce와 [ciphertext|tag] 분리
        ByteBuffer buffer = ByteBuffer.wrap(payload);

        byte[] extractedNonce = new byte[12];
        buffer.get(extractedNonce);

        byte[] extractedCipherTextWithTag = new byte[buffer.remaining()];
        buffer.get(extractedCipherTextWithTag);

        // 7. 복호화
        Cipher decryptCipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec decryptSpec = new GCMParameterSpec(128, extractedNonce);
        decryptCipher.init(Cipher.DECRYPT_MODE, key, decryptSpec);

        byte[] decrypted = decryptCipher.doFinal(extractedCipherTextWithTag);

        System.out.println("decrypted = " + new String(decrypted, StandardCharsets.UTF_8));
    }
}

```

* nonce: 복호화 시 동일하게 사용해야 하는 값
* ciphertext: 암호화된 데이터
* authentication tag: 무결성 검증 값

---

## 5. 관련 포스팅

* [[학습기록] 대칭키 암호화 모드 비교: 실무에서 GCM을 선택해야 하는 이유](https://sanghoon-lee.github.io/2026/03/21/aes-gcm/)

---

#암호화 #AES #ECB #CBC #GCM #java
