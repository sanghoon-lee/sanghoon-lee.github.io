---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (4) 핵심 구현체"
date: 2026-01-27
description: 애플리케이션 레벨에서 DB 컬럼 데이터를 암호화하는 토이프로젝트를 수행하는 과정을 정리한 글입니다. 4/5
image: /assets/images/jpa.jpg
categories: 토이프로젝트
---

지난 포스팅들을 통해서 DB 레벨 암호화와 애플리케이션 레벨 암호화의 차이를 공부하고, 애플리케이션 레벨 암호화의 구현 방식을 알아봤습니다. 그리고 토이프로젝트의 구조도 설명했습니다.

이번에는 어떤 흐름으로 애플리케이션 레벨 암호화가 동작하는지 핵심 구현체 위주로 살펴보도록 하겠습니다.

---

## 1. AttributeConverter의 구현

`AttributeConverter` 인터페이스는 엔티티의 필드와 DB 테이블의 컬럼 사이에서 데이터 변환 과정을 개발자가 직접 정의할 수 있도록 제공되는 확장 포인트입니다.

```java
public interface AttributeConverter<X, Y> {

    Y convertToDatabaseColumn(X attribute);

    X convertToEntityAttribute(Y dbData);
}
```

* X : 엔티티 필드의 데이터 타입
* Y : DB 컬럼에 저장되는 값의 데이터 타입

이 인터페이스를 상속받아 구현하면 개발자가 정의한 데이터 변환 과정이 JPA의 데이터 처리 절차에 통합되어 자동으로 수행됩니다.

| 구분 | 데이터 저장 시 | 데이터 조회 시 |
| --- | --- | --- |
| 호출 메서드 | convertToDatabaseColumn() | convertToEntityAttribute() |
| 변환 방향 | 평문 → 암호문 | 암호문 → 평문 |
| 호출 시점 | DB에 저장하기 직전 | DB에서 읽은 직후 |
| 결과 | DB에는 암호문 저장 | 엔티티에는 평문 설정 |

토이 프로젝트에서는 `AttributeConverter`를 상속받은 `BaseEncryptConverter`를 정의했습니다. 이후 기능 확장 가능성을 고려해, 데이터 변환 과정과 암·복호화 로직을 분리했습니다.  

---

### 1.1. BaseEncryptConverter 클래스

```java
@RequiredArgsConstructor
public abstract class BaseEncryptConverter<T>
        implements AttributeConverter<T, String> {
    protected final CryptoEngine cryptoEngine;

    @Override
    public String convertToDatabaseColumn(String attribute) {
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
    public String convertToEntityAttribute(String dbData) {
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

    protected abstract String convertToString(String attribute);

    protected abstract String convertToAttribute(String decrypted);
}
```

`BaseEncryptConverter`는 JPA의 데이터 처리 절차의 확장 포인트로서의 역할에만 충실하고, 실제 암·복호화 로직의 실행은 뒤에서 살펴볼 `CryptoEngine`에 위임하고 있습니다. 

이렇게 역할이 분리되면 암호화 알고리즘이 변경되더라도 `BaseEncryptConverter`의 변경을 최소화할 수 있고, 다른 컨버터에서도 암호화 알고리즘을 재사용하는 것이 용이합니다.

`BaseEncryptConverter`를 확장해서 문자열 데이터 타입을 처리하는 `StringEncryptConverter`를 별도로 정의했습니다. 

---

### 1.2. StringEncryptConverter 클래스

```java
/**
 * autoApply=false를 설정해서 암호화가 필요한 필드에만 명시적으로 적용되도록 했습니다. 필요하지 않은 필드에 자동 적용되는 위험을 방지하기 위한 선택입니다.
 */
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

토이 프로젝트에서는 문자열(String) 타입만 처리해도 충분하지만, 실무 환경에 적용하려면 다양한 데이터 타입을 처리할 수 있어야 합니다. 이러한 문제는 `StringEncryptConverter`처럼 `BaseEncryptConverter`를 확장해서 특정 데이터 타입에 종속적인 처리를 구현하는 방식으로 쉽게 해결될 수 있습니다.

---

### 1.3. crypto.converter 패키지

`BaseEncryptConverter`와 `StringEncryptConverter`이 포함된 crypto.converter 패키지의 구조는 다음과 같습니다.

```text
├─ crypto
│  ├─ converter
│  │  ├─ BaseEncryptConverter.java   
│  │  └─ StringEncryptConverter.java 
│  ├─ core
│  ├─ key
│  └─ exception
```

---

## 2. CryptoEngine의 구현 

`CryptoEngine`은 인터페이스로 선언하고, 실제 암·복호화 로직은 이 인터페이스를 상속받은 클래스에서 구현하도록 했습니다.

```java
public interface CryptoEngine {
    String encrypt(String plainText);
    String decrypt(String cipherText);
}
```
이렇게 역할을 인터페이스로 분리한 이유는 암·복호화 로직의 변경 가능성을 구조적으로 흡수하기 위함입니다. 다음과 같은 요구사항이 생기더라도 기존 구조를 크게 변경하지 않고 대응할 수 있습니다.

* 암호화 알고리즘 교체
* 테스트용 더미 구현체 추가
* 환경별(개발/운영) 구현 분리

`CryptoEngine`의 구현체로 AES-GCM 암호화 알고리즘을 사용하는 `AesGcmCryptoEngine`를 정의했습니다. 여기에는 랜덤 IV 생성해서 GCM Parameter Spec으로 초기화하고, 최종 결과를 Base64로 인코딩하는 등의 로직이 구현되어 있습니다. 

---

### 2.1. AesGcmCryptoEngine 클래스

```java
@Component
public class AesGcmCryptoEngine implements CryptoEngine{
    ...
    ...
    private final KeyProvider keyProvider;
    ...
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

    ...
}
```

구현체의 역할은 명확합니다.

* 평문 문자열을 암호문으로 변환
* 암호문을 다시 평문으로 복호화
* 암호화 알고리즘의 세부 구현

**“어떤 알고리즘을 사용하는지”, “IV를 어떻게 생성하는지”** 등과 같은 구현과 관련된 세부적인 사항을 구현체 밖에서는 전혀 알 필요가 없습니다. 이번에는 암호화 알고리즘 자체를 구현하는 것이 목적이 아니었기 때문에, 이미 충분히 검증된 알고리즘을 선택했습니다. 

---

### 2.2. crypto.core 패키지

`CryptoEngine`와 `AesGcmCryptoEngine`이 포함된 crypto.core 패키지의 구조는 다음과 같습니다.

```
├─ crypto
│  ├─ converter
│  ├─ core
│  │  ├─ AesGcmCryptoEngine.java   
│  │  └─ CryptoEngine.java 
│  ├─ key
│  └─ exception
```

---

## 3. KeyProvider의 구현

`CryptoEngine`과 구현체인 `AesGcmCryptoEngine`의 책임은 암·복호화 알고리즘의 수행으로 한정됩니다. 암·복호화에 사용되는 키 관리까지 책임지지는 않습니다.  

키 관리는 인터페이스로 선언된 `KeyProvider`를 상속받아 구현된 클래스에서 책임지도록 설계했습니다.

```java
public interface KeyProvider {
    String getBase64Key();
}
```

암·복호화 알고리즘의 수행과 키 관리의 책임이 분리되어 있는 지금과 같은 설계 구조에서는 나중에 키 관리 방식이 변경되더라도 변경이 최소화될 수 있습니다. 

토이 프로젝트에서는 `KeyProvider`의 구현체로 간단하게 프로퍼티 파일에서 키를 관리하는 `PropertyKeyProvider`를 정의했습니다.

---

### 3.1. PropertyKeyProvider 클래스

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

하지만, 실무에서는 `KMS`나 `HSM` 등과 같이 보안성이 보장되는 환경에서 키를 관리할 수 있어야 합니다. 그때는 `KeyProvider`를 상속받아 필요한 구현체를 구현해서 사용하면 됩니다.

---

### 3.2. crypto.key 패키지

`KeyProvider`와 `PropertyKeyProvider`가 포함된 crypto.key 패키지의 구조는 다음과 같습니다.

```
├─ crypto
│  ├─ converter
│  ├─ core
│  ├─ key
│  │  ├─KeyProvider.java
│  │  └─PropertyKeyProvider.java
│  └─ exception
```

참고로 `PropertyKeyProvider`가 키를 읽어올 수 있도록 application.properties 파일에 아래처럼 crypto.base64-key 속성이 추가해야 합니다.

```java
crypto.base64-key=[BASE64로 인코딩된 키 값]
```

---

## 4. 핵심 구현체의 책임 범위

지금까지 살펴본 핵심 구현체의 책임 범위는 다음과 같이 정리될 수 있습니다.

| 책임  | 인터페이스 | 구현체 |
| ---- | ----- | ---- |
| 암·복호화의 실행 시점 | AttributeConverter | BaseEncryptConverter, StringEncryptConverter |
| 암호화 알고리즘 | CryptoEngine | AesGcmCryptoEngine |
| 키 제공 | KeyProvider | PropertyKeyProvider |

<img class="main-image" src="/assets/images/jpa.jpg" alt="Spring Data JPA">

다음 포스팅에서는 실제로 어떻게 동작하는지 이야기를 이어가면서, 실무 환경에 사용하는데 어떤 제약사항이 있으며 해결책은 무엇인지 알아보도록 하겠습니다.

---

## 5. 포스팅 시리즈

* [[토이 프로젝트] DB 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[토이 프로젝트] DB 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] DB 컬럼 암호화: (3) 프로젝트 구조](https://sanghoon-lee.github.io/2026/01/26/JPACryptoConverter3/)
* [토이 프로젝트] DB 컬럼 암호화: (4) 핵심 구현체
* [[토이 프로젝트] DB 컬럼 암호화:  (5) 동작의 흐름 및 제약사항](https://sanghoon-lee.github.io/2026/01/31/JPACryptoConverter5/)

---

## 6. 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

---

#JPA #DB #암호화 #보안 #토이프로젝트 #AttributeConverter #구현체
