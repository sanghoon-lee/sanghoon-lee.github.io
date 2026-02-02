---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (5) 동작의 흐름 및 제약사항"
date: 2026-01-31
categories: 토이프로젝트
---

지난 포스팅까지 다룬 내용을 바탕으로,
이번 포스팅에서는 실제 코드가 어떤 흐름으로 동작하는지를
차례대로 따라가 보려고 합니다.

그에 앞서, 먼저 엔티티가 어떻게 정의되어 있고
그중 어떤 필드가 암호화 대상인지를 정리해보겠습니다.

## 엔티티의 정의

애플리케이션 레벨 암호화의 동작을 확인하기 위해 다음과 같은 구조의 엔티티를 사용했습니다.

| 필드명 | 데이터 타입 | 내용 | 암호화 대상 |
| --- | --- | --- | ---- |
| id | Long | ID | X |
| phoneNumber | String | 전화번호(예 : 010-1234-5678) | O |
| userName | String | 이름(예 : 홍길동) | X |
| sex | int | 성별 - 1:남성,0:여성 | X |
| age | int | 나이 | X |

전화번호는 개인정보에 해당하기 때문에, 실무에서도 당연히 암호화가 필요한 대상입니다. 그래서 암호화 대상 속성으로 전화번호(`phoneNumber`)를 선택했습니다.

`Account` 엔티티는 다음과 같이 정의되어 있습니다.

```java
@Entity
public class Account extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 저장 시 암호화, 조회 시 복호화
    @Convert(converter = StringEncryptConverter.class)
    private String phoneNumber;

    private String userName;
    private int sex;
    private int age;
}
```

## 엔티티 저장 시 암호화 흐름

JPA를 사용하면 서비스 계층에서는 엔티티를 DB에 저장하면서 암호화를 의식할 필요가 없습니다. 아래 코드처럼 엔티티를 생성해서 값을 설정하고, save() 메서드를 호출하는 것으로 끝입니다.

```java
Account account = new Account();
account.setPhoneNumber("010-1234-5678");
account.setUserName("홍길동");
account.setSex(1);
account.setAge(30);

accountRepository.save(account);
```

하지만 엔티티가 DB에 저장되려는 순간, @Convert 어노테이션이 선언된 필드는 등록된 AttributeConverter가 자동으로 개입합니다.

이 과정을 정리하면 다음과 같습니다.

| 순서 | 처리 단계              | 설명                             |
| -- | ------------------ | ------------------------------ |
| 1  | 서비스 계층             | 엔티티 생성 및 값 설정                  |
| 2  | 리포지토리              | `save()` 호출                    |
| 3  | JPA                | 엔티티를 DB 컬럼 값으로 변환              |
| 4  | AttributeConverter | `convertToDatabaseColumn()` 호출 |
| 5  | CryptoEngine       | 평문 → 암호문 변환                    |
| 6  | DB                 | 암호문이 컬럼에 저장                    |

이 과정에서 핵심은 4번 단계입니다.

phoneNumber 필드는 @Convert로 선언되어 있기 때문에,
DB에 저장되기 직전에 StringEncryptConverter의
convertToDatabaseColumn() 메서드가 호출됩니다.

**StringEncryptConverter의 convertToDatabaseColumn 메서드**
```java
@Override
public String convertToDatabaseColumn(String attribute) {
    return cryptoEngine.encrypt(attribute);
}
```

이 메서드는 엔티티에 설정된 평문을 입력값으로 받아,
CryptoEngine을 통해 암호화를 수행한 뒤 암호문 값을 반환합니다.

그 결과로 서비스 계층에서는 평문을 다루지만, DB에는 암호문이 저장될 수 있습니다.

## 엔티티 조회 시 복호화 흐름

서비스 계층에서는 조회 시에도 복호화를 전혀 의식할 필요가 없습니다. 아래 코드처럼 리포지토리를 통해 엔티티를 조회하면,
엔티티의 필드에는 평문 값이 설정된 상태로 반환됩니다.

```java
Account account = accountRepository.findById(id)
        .orElseThrow();

String phoneNumber = account.getPhoneNumber();
```

하지만 DB에는 phoneNumber 컬럼이 암호문 형태로 저장되어 있습니다. 이 암호문이 엔티티에 매핑되는 과정에서, @Convert 어노테이션이 선언된 필드에 대해 AttributeConverter가 자동으로 개입합니다.

이 과정을 정리하면 다음과 같습니다.

| 순서 | 처리 단계                | 설명                              |
| -- | -------------------- | ------------------------------- |
| 1  | 리포지토리                | `findById()` 호출                 |
| 2  | JPA                  | DB로부터 컬럼 값 조회                   |
| 3  | JPA                  | 컬럼 값을 엔티티 필드로 매핑                |
| 4  | `AttributeConverter` | `convertToEntityAttribute()` 호출 |
| 5  | `CryptoEngine`       | 암호문 → 평문 변환                     |
| 6  | 엔티티                  | 복호화된 평문 값 설정                    |


저장과 마찬가지로, 조회 시 흐름에서도 핵심은 4번 단계입니다.

phoneNumber 필드는 @Convert로 선언되어 있기 때문에,
DB에서 값을 읽은 직후 StringEncryptConverter의
convertToEntityAttribute() 메서드가 호출됩니다.

**StringEncryptConverter의 convertToEntityAttribute() 메서드**
```java
@Override
public String convertToEntityAttribute(String dbData) {
    return cryptoEngine.decrypt(dbData);
}
```

이 메서드는 DB로부터 조회된 암호문을 입력값으로 받아,
CryptoEngine을 통해 복호화를 수행한 뒤 평문 값을 반환합니다.

그 결과로 서비스 계층이나 도메인 모델에서는
암호화 여부를 전혀 의식하지 않고도, DB로부터
항상 평문 데이터를 조회할 수 있습니다.

## 관련 포스팅

* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)