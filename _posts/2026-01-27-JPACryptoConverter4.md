---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기"
date: 2026-01-27
categories: 토이프로젝트
---

지난 포스팅들을 통해서 애플리케이션 레벨 암호화의 구현을 검토하게 된 계기와 구현 방식으로 JPA의 `AttributeConverter`를 선택하게 된 이유, 그리고 전체 프로젝트 구조까지 정리해봤습니다.

이제 어떤 코드 흐름을 통해 애플리케이션 레벨 암호화가 동작하는지를 핵심 코드 중심으로 이야기해볼 차례입니다. 복잡한 세부 구현사항을 일일이 설명하기 보다는 `AttributeConverter`를 중심으로 각 구성 요소가 어떤 역할을 맡고, 어떻게 연결되어 있는지에 초점을 맞춰 설명하겠습니다.

## AttributeConverter의 구현

`AttributeConverter` 인터페이스는 엔티티의 필드와 DB 테이블의 컬럼 사이에서 데이터 변환 과정을 개발자가 직접 정의할 수 있도록 제공되는 확장 포인트입니다.

```java
public interface AttributeConverter<X, Y> {

    Y convertToDatabaseColumn(X attribute);

    X convertToEntityAttribute(Y dbData);
}
```
* X : 엔티티 필드의 데이터 타입
* Y : DB 컬럼에 저장되는 값의 데이터 타입

이 인터페이스를 상속받아 구현하면, JPA가 데이터를 저장하거나 조회하는 시점에 아래와 같은 흐름으로 데이터 변환 과정이 자동으로 수행됩니다.

| 구분 | 데이터 저장 시 | 데이터 조회 시 |
| --- | --- | --- |
| 호출 메서드 | `convertToDatabaseColumn()` | `convertToEntityAttribute()` |
| 변환 방향 | 평문 → 암호문 | 암호문 → 평문 |
| 호출 시점 | DB에 저장하기 직전 | DB에서 읽은 직후 |
| 결과 | DB에는 암호문 저장 | 엔티티에는 평문 설정 |

이번 토이 프로젝트에서는 `AttributeConverter`를 상속받아
`BaseEncryptConverter`를 정의했습니다. 나중에 기능이 확장될 가능성에 대비해서 데이터 변환 과정의 확장 포인트와 실제 암·복호화 로직을 분리했습니다.  

**BaseEncryptConverter**
```java
public abstract class BaseEncryptConverter<T>
        implements AttributeConverter<T, String> {

    protected final CryptoEngine cryptoEngine;

    protected BaseEncryptConverter(CryptoEngine cryptoEngine) {
        this.cryptoEngine = cryptoEngine;
    }

    @Override
    public String convertToDatabaseColumn(T attribute) {
        if (attribute == null) {
            return null;
        }
        return cryptoEngine.encrypt(convertToString(attribute));
    }

    @Override
    public T convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        return convertToEntityValue(cryptoEngine.decrypt(dbData));
    }

    protected abstract String convertToString(T attribute);

    protected abstract T convertToEntityValue(String decrypted);
}
```

`BaseEncryptConverter`에서는 암·복호화 로직이 수행되는 시점은 결정하지만, 실제 처리는 뒤에서 살펴볼 `CryptoEngine`에 위임하고 있습니다.

이렇게 역할을 분리할 이유는 다음과 같습니다.

* `BaseEncryptConverter`는 JPA 확장 포인트로서의 역할에만 집중
* 암호화 알고리즘 변경 시 `BaseEncryptConverter`의 변경 최소화
* 암·복호화 로직을 여러 컨버터에서 재사용 가능

`BaseEncryptConverter`를 확장해서 문자열 데이터 타입을 처리하는 `StringEncryptConverter`를 별도로 정의했습니다. 이렇게 하면, 공통적인 데이터 변환 흐름은 그대로 유지한 채 다양한 데이터 타입에 대해서 종속적인 처리를 확장해나갈 수 있습니다.

**StringEncryptConverter**
```java
@Converter
public class StringEncryptConverter
        extends BaseEncryptConverter<String> {

    public StringEncryptConverter(CryptoEngine cryptoEngine) {
        super(cryptoEngine);
    }

    @Override
    protected String convertToString(String attribute) {
        return attribute;
    }

    @Override
    protected String convertToEntityValue(String decrypted) {
        return decrypted;
    }
}
```

토이 프로젝트에서는 문자열(String) 타입만 처리해도 충분합니다. 하지만, 실무 환경에서는 다양한 데이터 타입을 처리할 수 있어야 합니다. 

`BaseEncryptConverter`와 `StringEncryptConverter`이 포함된 crypto 패키지의 구조는 다음과 같습니다.
```
├─ crypto
│  ├─ converter
│  │  ├─ BaseEncryptConverter.java   
│  │  └─ StringEncryptConverter.java 
│  ├─ core
│  ├─ key
│  └─ exception
```

## Entity 코드 정의

토이 프로젝트에서 사용할 엔티티는 domain 패키지에 위치한 `Account` 클래스에서 정의했습니다. 

| 필드명 | 데이터 타입 | 내용 | 암호화 대상 |
| --- | --- | --- | ---- |
| id | Long | ID | X |
| phoneNumber | String | 전화번호(예 : 010-1234-5678) | O |
| userName | String | 이름(예 : 홍길동) | X |
| sex | int | 성별 - 1:남성,0:여성 | X |
| age | int | 나이 | X |


최초 엔티티는 다음과 같이 정의했습니다.

```java
public class Account extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 저장시 암호화, 조회시 복호화
    @Convert(converter = StringEncryptConverter.class)
    private String phoneNumber;

    private String userName;
    private int sex;
    private int age;
    ...
```





## 포스팅 시리즈

* [[토이 프로젝트] DB 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[토이 프로젝트] DB 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] DB 컬럼 암호화: (3) 프로젝트 구조 살펴보기](https://sanghoon-lee.github.io/2026/01/26/JPACryptoConverter3/)
* [토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기

## 관련 포스팅

* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)
