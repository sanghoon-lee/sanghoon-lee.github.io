---
layout: post
title: "[학습기록] Java에서 AES-GCM 안전하게 구현하기: 실수하기 쉬운 4가지"
date: 2026-03-21
categories: 학습기록
---

이번 포스팅은 Java를 기준으로 AES-GCM을 안전하게 사용하는 방법을 정리한 글입니다.

## 1. AES-GCM 기본 구조

AES-GCM은 다음과 같은 요소로 구성됩니다.

* Key: 암호화에 사용하는 비밀키
* Nonce(Initial Value for Counter): 한 번만 사용하는 초기값
* Plaintext: 암호화 대상 데이터(평문)
* Ciphertext: 암호화 결과(암호문)
* Authentication Tag: 무결성 검증 값

AES-GCM은 데이터를 단순히 암호화하는 것이 아니라, 암호화와 동시에 무결성까지 검증할 수 있는 AEAD 방식입니다. 즉, 복호화 과정에서 데이터가 변경되었는지도 함께 확인할 수 있습니다.

## 2. Java에서 AES-GCM 구현

Java에서는 `Cipher` 클래스를 이용해 AES-GCM을 구현할 수 있습니다. 

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

## 3. AES-GCM 사용 시 주의사항

AES-GCM을 사용할 때 가장 중요한 것은 올바르게 사용하는 것입니다.

다음은 실무에서 반드시 지켜야 할 핵심 사항입니다.

### 3.1. Nonce 재사용 금지

같은 키에서 동일한 Nonce를 절대 재사용하면 안 됩니다.

Nonce가 재사용되면 동일한 키스트림이 생성되며,
암호문을 통해 평문 간의 관계가 노출될 수 있습니다.

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

### 3.2. 인증 태그 검증 필수

AES-GCM에서는 복호화 과정에서 인증 태그 검증이 자동으로 수행됩니다.

만약, 데이터가 변조되었다면 `AEADBadTagException`이 발생합니다. 이 경우 해당 데이터는 신뢰할 수 없는 데이터로 간주해야 합니다.
따라서 이 예외를 반드시 처리해야 하며, 단순히 무시하거나 복호화를 계속 진행해서는 안 됩니다.

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

AES-GCM은 암호문뿐 아니라 인증 태그까지 함께 검증하기 때문에, 암호문이 조금이라도 변경되면 복호화에 실패합니다.

실제로 위에서 예제로 보여줬던 AesGcmExample 클래스의 코드 일부를 아래와 같이 수정해서 암호문을 변조해보면, `AEADBadTagException` 예외가 발생하는 것을 확인할 수 있습니다.

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

**인증 태그(Authentication Tag)**

> AES-GCM에서 생성되는 인증 태그는 데이터가 암호화된 이후 변경되지 않았음을 검증하기 위한 값입니다.
> 즉, “이 데이터가 중간에 변경되지 않았음을 보장하는 값”이라고 볼 수 있습니다.
> 암호화 과정에서 평문과 키를 기반으로 함께 생성되며, 복호화 시 동일한 방식으로 다시 계산하여 값이 일치하는지 확인합니다.
> 만약 암호문이 중간에 변조되었다면, 인증 태그 검증에 실패하고 예외가 발생합니다. 결과적으로 인증 태그는 데이터의 무결성을 보장하는 핵심 요소입니다.

### 3.3. Nonce 길이 (12 bytes 권장)

GCM에서는 96bit(12 bytes) 길이의 Nonce 사용이 권장됩니다. 이는 성능과 보안 측면에서 최적화된 길이이며, 대부분의 구현에서 표준으로 사용됩니다.

### 3.4. 인증 태그 길이(128 bit 권장)

Java에서 AES-GCM을 사용할 때 다음과 같이 설정합니다.

```java
GCMParameterSpec spec = new GCMParameterSpec(128, nonce);
```

여기서 128은 인증 태그의 길이를 의미하며, 단위는 bit입니다.

AES-GCM에서는 96, 104, 112, 120, 128 bit 등의 태그 길이를 사용할 수 있지만,
실무에서는 128bit를 사용하는 것이 표준입니다.

그 이유는 다음과 같습니다.

* 충분한 보안성 제공 (위변조 공격에 대한 강한 내성)
* 대부분의 라이브러리와 프로토콜에서 기본값으로 사용
* 성능 대비 보안 수준이 가장 적절

태그 길이를 줄이면 성능이나 데이터 크기 측면에서 이점이 있을 수 있지만, 무결성 검증의 강도가 낮아지기 때문에 일반적인 시스템에서는 권장되지 않습니다. 따라서 특별한 이유가 없다면, AES-GCM에서는 128bit 인증 태그를 사용하는 것이 가장 안전한 선택입니다.

## 4. 인증 태그를 포함한 데이터 구조

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

아래는 조금 더 실무적인 관점에서 AES-GCM을 어떻게 구현하는지 보여주는 코드입니다.

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

## 5. 정리

AES-GCM은 강력한 암호화 방식이지만,
잘못된 사용은 심각한 보안 취약점으로 이어질 수 있습니다.

특히 다음 네 가지는 반드시 지켜야 합니다.

* Nonce는 절대 재사용하지 않는다
* 인증 태그 검증을 반드시 수행한다
* 권장되는 Nonce 길이(12 bytes)를 사용한다
* 권장되는 인증 태그 길이(128 bit)를 사용한다

이러한 기본 원칙만 지켜도 AES-GCM을 안전하게 사용할 수 있습니다.

즉, AES-GCM의 안전성은 알고리즘 자체뿐 아니라 Nonce와 인증 태그를 어떻게 다루느냐에 달려 있습니다.

## 관련 포스팅

* [[학습기록] 대칭키 암호화 모드 비교: 실무에서 GCM을 선택해야 하는 이유](https://sanghoon-lee.github.io/2026/03/21/aes-gcm/)

