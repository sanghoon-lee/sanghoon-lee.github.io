---
layout: post
title: "[토이프로젝트] JPA DB 컬럼 암호화: (4) AttributeConverter와 AES-GCM 암호화 구현"
date: 2026-01-27
description: JPA AttributeConverter를 활용한 DB 컬럼 암호화 기능을 구현합니다. BaseEncryptConverter, AES-GCM 기반 CryptoEngine, KeyProvider의 역할과 실제 Java 코드를 살펴봅니다.
image: /assets/images/jpa.jpg
series: jpa-converter
categories: 토이프로젝트
tags:
    - 자바
    - 보안
    - 데이터
redirect_from:
    - /2026/01/27/JPACryptoConverter4/
---

이전 포스팅에서는 `JPA`의 `AttributeConverter`를 활용한 DB 컬럼 암호화 기능을 구현하기 위해 프로젝트의 구조를 설계했습니다.

핵심은 데이터 변환, 암·복호화 처리, 암호화 키 제공 기능을 서로 다른 구성 요소로 분리하는 것이었습니다.

이번에는 이러한 설계를 실제 코드로 어떻게 구현했는지 핵심 인터페이스와 클래스를 중심으로 살펴보겠습니다.

---

## 1. AttributeConverter의 구현

`AttributeConverter`는 `Entity` 속성과 DB 컬럼 사이에서 데이터 변환을 수행할 수 있도록 `JPA`에서 제공하는 인터페이스입니다.

```java
public interface AttributeConverter<X, Y> {

    Y convertToDatabaseColumn(X attribute);

    X convertToEntityAttribute(Y dbData);
}
```

* X : 엔티티 필드의 데이터 타입
* Y : DB 컬럼에 저장되는 값의 데이터 타입

| 메서드 | 역할 |
| --- | ---- |
| `convertToDatabaseColumn()` | `Entity` 속성의 평문을 암호문으로 변환 |
| `convertToEntityAttribute()` | DB에서 조회한 암호문을 평문으로 변환 |

---

### 1.1. BaseEncryptConverter 클래스

`BaseEncryptConverter`는 데이터 타입과 관계없이 공통으로 사용할 수 있는 암·복호화 처리 흐름을 정의한 추상 클래스입니다.

```java
@RequiredArgsConstructor
public abstract class BaseEncryptConverter<T> implements AttributeConverter<T, String> {
    protected final CryptoEngine cryptoEngine;

    @Override
    public String convertToDatabaseColumn(T attribute) {
        if (attribute == null) {
            return null;
        }

        try {
            return cryptoEngine.encrypt(convertToString(attribute));
        } catch (Exception e) {
            throw new CryptoException("Failed to encrypt attribute", e);
        }
    }

    @Override
    public T convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }

        try {
            String decrypted = cryptoEngine.decrypt(dbData);
            return convertToAttribute(decrypted);
        } catch (Exception e) {
            throw new CryptoException("Failed to decrypt attribute", e);
        }
    }

    protected abstract String convertToString(T attribute);

    protected abstract T convertToAttribute(String decrypted);
}
```

`BaseEncryptConverter`는 `Entity` 속성을 문자열로 변환한 후 `CryptoEngine`에 암호화를 위임합니다. 조회할 때는 복호화한 문자열을 다시 `Entity` 속성의 타입으로 변환합니다.

이렇게 구성하면 `JPA`의 데이터 변환과 실제 암·복호화 알고리즘을 분리할 수 있습니다. `CryptoEngine`의 인터페이스가 유지된다면 암호화 알고리즘이 변경되더라도 `BaseEncryptConverter`를 수정할 필요가 없습니다.

또한 `convertToString()`과 `convertToAttribute()`를 추상 메서드로 정의했기 때문에, 데이터 타입에 따른 변환 처리는 하위 클래스에서 구현할 수 있습니다.

---

### 1.2. StringEncryptConverter 클래스

문자열 타입의 컬럼 암호화를 위해 `BaseEncryptConverter`를 상속받은 하위 클래스인 `StringEncryptConverter`를 구현했습니다.

```java
@Converter(autoApply=false) 
@Component
public class StringEncryptConverter extends BaseEncryptConverter<String>{
    public StringEncryptConverter(CryptoEngine cryptoEngine) {
        super(cryptoEngine);
    }

    @Override
    protected String convertToString(String attribute) {
        return attribute;
    }

    @Override
    protected String convertToAttribute(String decrypted) {
        return decrypted;
    }
}
```

문자열은 암·복호화 과정에서 별도의 타입 변환이 필요하지 않으므로 입력값을 그대로 반환합니다.

`@Converter(autoApply = false)`를 지정한 이유는 모든 `String` 타입의 속성에 암호화 기능이 자동으로 적용되는 것을 방지하기 위해서입니다.

대신 암호화가 필요한 `Entity` 속성에 아래와 같이 `@Convert`를 명시적으로 선언하도록 했습니다.

```java
@Convert(converter = StringEncryptConverter.class)
private String phoneNumber;
```

이렇게 하면 어떤 속성에 암호화 기능이 적용되는지 `Entity` 정의에서 확인할 수 있습니다.

물론 개발자가 `@Convert` 선언을 누락하면 해당 속성에 암호화가 적용되지 않을 수 있으므로 암호화 대상 컬럼을 검증하는 절차도 필요합니다.

또한 `@Component`를 통해 `Spring Bean`으로 등록하고 생성자에서 `CryptoEngine`을 주입받도록 구성했습니다.

하지만 `Spring` 의존성 주입은 사용하는 `JPA` 구현체 및 `Spring` 연동 방식에 따라 달라질 수 있으므로 실제 실행 환경에서 정상적으로 동작하는지 확인해야 합니다.

---

## 2. CryptoEngine의 구현 

`CryptoEngine`은 실제 데이터의 암·복호화 처리를 담당하는 구성 요소입니다.

```java
public interface CryptoEngine {
    String encrypt(String plainText);
    String decrypt(String cipherText);
}
```

`BaseEncryptConverter`는 `CryptoEngine` 인터페이스에 의존하므로 구체적인 암호화 알고리즘을 알 필요가 없습니다.

이러한 구조에서는 암호화 처리 방식이 변경되거나 테스트용 구현체가 필요한 경우에도 `Converter`의 변경을 최소화할 수 있습니다.

실제 구현체로는 AES-GCM 알고리즘을 사용하는 `AesGcmCryptoEngine`을 정의했습니다.

AES-GCM은 암호화된 데이터의 기밀성과 무결성을 함께 보호할 수 있는 인증 암호화(Authenticated Encryption) 방식입니다.

아래는 AES-GCM 암호화 처리와 관련된 부분만 발췌한 것으로 일부 필드와 상수, 보조 메서드 및 복호화 구현은 생략했습니다.

```java
@Component
public class AesGcmCryptoEngine implements CryptoEngine{
    // 상수 및 전역변수 선언 생략
    
    public AesGcmCryptoEngine(KeyProvider keyProvider){
        this.keyProvider = keyProvider;
    }

    @Override
    public String encrypt(String plainText) {
        if(plainText==null)
            return null;

        try{
            byte[] keyBytes = decodeAndValidateKey(keyProvider.getBase64Key());

            byte[] iv = new byte[IV_LENGTH_BYTES];
            secureRandom.nextBytes(iv);

            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            SecretKeySpec keySpec = new SecretKeySpec(keyBytes, "AES");
            GCMParameterSpec gcmSpec = new GCMParameterSpec(GCM_TAG_LENGTH_BITS, iv);
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, gcmSpec);

            byte[] cipherBytes = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

            // 복호화에 필요한 IV를 암호문 및 인증 태그와 함께 저장
            byte[] out = new byte[iv.length + cipherBytes.length];
            System.arraycopy(iv, 0, out, 0, iv.length);
            System.arraycopy(cipherBytes, 0, out, iv.length, cipherBytes.length);

            // IV와 암호화 결과를 Base64로 인코딩하여 반환
            // 향후 키 교체를 지원하려면 키 버전 식별자를 추가할 수 있음
            return Base64.getEncoder().encodeToString(out);
        } catch(Exception e){
            throw new CryptoException("Failed to encrypt",e);
        }
    }
}
```

암호화 처리 과정은 다음과 같습니다.

1. `KeyProvider`를 통해 암호화 키를 전달받습니다.
2. Base64로 인코딩된 키를 디코딩하고 유효성을 검사합니다.
3. `SecureRandom`을 이용해 새로운 IV를 생성합니다.
4. `Cipher`를 AES-GCM 방식으로 초기화합니다.
5. 평문을 암호화하고 결과를 저장 가능한 형태로 변환합니다.

여기서 중요한 것은 IV의 관리입니다.

AES-GCM에서는 동일한 키로 암호화할 때 nonce(IV)가 재사용되지 않도록 해야 합니다.

또한 복호화할 때 암호화에 사용한 IV와 인증 태그가 필요하므로 암호문을 DB에 저장할 때 이러한 정보도 복원할 수 있는 형태로 관리해야 합니다.

Java의 AES-GCM에서는 일반적으로 `doFinal()` 결과에 암호문과 인증 태그만 포함됩니다. 따라서 `doFinal()`의 반환값 앞에 IV를 결합하고, 이를 Base64로 인코딩하여 DB에 저장할 문자열로 변환했습니다.

---

## 3. KeyProvider의 구현

`AesGcmCryptoEngine`은 암·복호화 처리를 수행하기 위해 암호화 키가 필요합니다. 하지만 암호화 키를 어디에 보관하고 어떻게 가져올 것인지까지 직접 처리하도록 만들고 싶지는 않았습니다.

그래서 키를 제공하는 역할을 별도의 `KeyProvider` 인터페이스로 분리했습니다.

```java
public interface KeyProvider {

    String getBase64Key();
}
```

`CryptoEngine`은 `KeyProvider`를 통해 암호화 키를 전달받으므로 키의 저장 위치를 직접 알 필요는 없습니다. 다만 현재 인터페이스는 Base64 문자열을 반환하도록 정의되어 있어 키의 표현 방식에는 의존하고 있습니다.

실제 구현체로는 설정 파일을 이용해 키를 제공하는 `PropertyKeyProvider`를 정의했습니다.

```java
@Component
@RequiredArgsConstructor
public class PropertyKeyProvider implements KeyProvider{
    private final CryptoProperties cryptoProperties;

    @Override
    public String getBase64Key() {
        return cryptoProperties.getBase64Key();
    }
}
```

`PropertyKeyProvider`는 `CryptoProperties`에 바인딩된 설정값을 반환하는 역할만 담당합니다.

이전 포스팅에서 살펴본 것처럼 `CryptoProperties`는 `@ConfigurationProperties`를 이용해 설정값을 전달받습니다.

```java
crypto.base64-key=<Base64로 인코딩된 테스트 키>
```

이번 프로젝트에서는 암·복호화 기능의 구현과 책임 분리를 검증하는 데 집중하기 위해 설정 파일을 사용했습니다.

---

## 4. 핵심 구현체의 책임 범위

지금까지 살펴본 핵심 구성 요소의 책임은 다음과 같습니다.

| 구성 요소 | 인터페이스 및 추상 클래스 | 실제 구현체 |
| --- | --- | --- |
| JPA 데이터 변환 | AttributeConverter, BaseEncryptConverter | StringEncryptConverter |
| 암·복호화 처리 | CryptoEngine | AesGcmCryptoEngine |
| 암호화 키 제공 | KeyProvider | PropertyKeyProvider |

이러한 구조에서 각 구성 요소는 자신이 담당하는 역할에 집중합니다.

이를 통해 비즈니스 로직과 암호화 처리를 분리하고, 암호화 알고리즘과 키 제공 방식의 변경이 다른 구성 요소에 미치는 영향을 줄이는 구조를 구현했습니다.

물론 이러한 책임 분리가 모든 암호화 요구사항을 해결해 주지는 않습니다.

암호화된 데이터의 검색, 키 교체 시 기존 데이터의 복호화, 예외 처리와 같은 문제는 별도로 검토해야 합니다.

---

## 5. 다음 포스팅

다음 포스팅에서는 실제 데이터를 저장하고 조회하는 과정에서 이러한 구성 요소가 어떻게 동작하는지 확인하겠습니다.

또한 `AttributeConverter`를 활용한 DB 컬럼 암호화를 실무에 적용할 때 어떤 제약사항이 발생할 수 있는지도 살펴보겠습니다.


