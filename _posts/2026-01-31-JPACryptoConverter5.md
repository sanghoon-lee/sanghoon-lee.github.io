---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (5) 동작의 흐름 및 제약사항"
date: 2026-01-31
categories: 토이프로젝트
---

<style>
.main-image {
  width: 100%;
  max-width: 800px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

지난 포스팅까지 다룬 내용을 바탕으로,
이번 포스팅에서는 실제 코드가 어떤 흐름으로 동작하는지를
차례대로 따라가 보려고 합니다.

먼저 엔티티가 어떻게 정의되어 있으며, 그 중에서 어떤 필드가 암호화 대상인지를 살펴보도록 하겠습니다.

## 엔티티의 정의

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

    ...
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

하지만, 엔티티가 DB에 저장되려는 순간에 @Convert 어노테이션이 선언된 필드는 등록된 AttributeConverter가 자동으로 개입하게 됩니다.

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

## 암호화된 컬럼에서 조회의 한계

애플리케이션 레벨에서 암·복호화를 자동화하는 흐름은
비교적 깔끔하게 구현할 수 있었습니다. 하지만, 이 방식에도 명확한 한계가 존재합니다.

전화번호(`phoneNumber`)는 DB에 암호문 형태로 저장되기 때문에,
다음과 같은 조회 조건을 직접 사용할 수 없습니다.

* `WHERE phone_number = '010-1234-5678'`
* `LIKE`, `PREFIX` 검색
* 인덱스를 활용한 동등 비교 검색

이는 애플리케이션 레벨 암호화 방식의 구조적인 특성으로,
JPA의 `AttributeConverter`로는 해결할 수 없는 영역입니다.

즉, **암호화된 컬럼은 조회 조건으로 사용하기 어렵다**는 점은
명확한 한계였습니다. 

### 토이 프로젝트에서의 해결방식

이러한 한계를 극복하기 위해서, 이번 토이 프로젝트에서는 암호문과 함께 **조회용 식별자 역할을 하는 해시 값**을 별도로 저장하는 방식을 선택했습니다.

그래서 암호화된 전화번호(`phoneNumber`)의 조회용 식별자로 사용될 해시값(`phoneNumberHash`)이 엔티티에 추가되었습니다. 

| 필드명 | 데이터 타입 | 내용 | 암호화 대상 |
| --- | --- | --- | ---- |
| id | Long | ID | X |
| phoneNumber | String | 전화번호(예 : 010-1234-5678) | O |
| **phoneNumberHash** | String | phoneNumber의 해시값 | X |
| userName | String | 이름(예 : 홍길동) | X |
| sex | int | 성별 - 1:남성,0:여성 | X |
| age | int | 나이 | X |

Account 엔티티의 최종 정의는 다음과 같습니다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
@Table(
        uniqueConstraints = {
                @UniqueConstraint(name="uk_account_phonenumber_hash", columnNames="phoneNumberHash")
        },
        indexes = {
                @Index(name="idx_account_phonenumber_hash", columnList="phoneNumberHash")
        }
)

public class Account extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 저장시 암호화, 조회시 복호화
    @Convert(converter = StringEncryptConverter.class)
    @Column(nullable = false, length = 512) // 암호문 길이 감안 (여유있게)
    private String phoneNumber;

    @Column(nullable = false, length = 64)
    private String phoneNumberHash;

    private String userName;
    private int sex;
    private int age;

    @Builder
    public Account(String phoneNumber,String phoneNumberHash,String userName,int sex,int age){
        this.phoneNumber = phoneNumber;
        this.phoneNumberHash = phoneNumberHash;
        this.userName = userName;
        this.sex = sex;
        this.age = age;
    }
}
```

전화번호(`phoneNumber`)의 중복 저장을 방지하도록 해시값(`phoneNumberHash`)을 Unique Key로 설정하고, 인덱스도 구성했습니다. 

이렇게 하면 다음과 같은 장점을 얻을 수 있습니다.

* 개인정보 보호를 유지
* `WHERE phone_number_hash = ?` 조건으로 정확한 조회 가능
* 유니크 인덱스를 통한 중복 전화번호 방지 가능

서비스 계층에서도 입력받은 전화번호의 해시값을 계산해서 조회할 수 있도록 아래처럼 코드를 작성했습니다.

```java
@Transactional(readOnly = true)
    public AccountResponse findByPhoneNumber(String phoneNumber) {
        String normalizedPhoneNumber = PhoneNumberNormalizer.normalize(phoneNumber);
        String hashNormalizedPhoneNumber = phoneNumberHashService.hashNormalizedPhoneNumber(normalizedPhoneNumber);

        Account account = accountRepository.findByPhoneNumberHash(hashNormalizedPhoneNumber)
                .orElse(null);

        if (account == null) {
            return null;
        }

        return AccountResponse.of(account);
    }
```

엔티티 저장 시 흐름을 정리하면 다음과 같습니다.

1. 서비스 계층에서 전화번호 평문 입력
2. 전화번호 해시 값 생성 → `phoneNumberHash`에 설정
3. `phoneNumber` 필드는 `AttributeConverter`를 통해 자동 암호화
4. DB에는
   * 암호문 전화번호
   * 해시된 전화번호
   가 함께 저장

이렇게 역할을 분리함으로써,

* **암호화 컬럼은 보호 목적**
* **해시 컬럼은 조회 목적**

이라는 책임을 명확히 나눌 수 있었습니다.

이 방식은 실무에서도 자주 사용되는 패턴이지만, 다음과 같은 점은 반드시 고려해야 합니다.

* 해시 알고리즘 선택 (충분한 충돌 저항성)
* 솔트 적용 여부
* 해시 값 유출 시의 영향 범위
* 해시 컬럼에도 개인정보로 분류될 가능성

## 테스트 결과

**API를 통해 설정된 엔티티의 값을 DB에 저장**

```bash
URL: POST /api/v1/account
Content-Type: application/json

Request Body:
{
    "phoneNumber":"010-1234-5678",
    "userName":"홍길동",
    "sex":1,
    "age":45
}

Response Body:
{
    "code": 200,
    "status": "OK",
    "message": "OK",
    "data": {
        "id": 1,
        "phoneNumber": "01012345678",
        "userName": "홍길동",
        "sex": 1,
        "age": 45
    }
}
```

**API를 통해 DB에서 전화번호로 엔티티를 조회**

```bash
URL: GET http://127.0.0.1:8080/api/v1/account?phoneNumber=01012345678
Content-Type: application/json

Response Body:
{
    "code": 200,
    "status": "OK",
    "message": "OK",
    "data": {
        "id": 1,
        "phoneNumber": "01012345678",
        "userName": "홍길동",
        "sex": 1,
        "age": 45
    }
}
```

**H2 콘솔에서 직접 DB에 저장된 값을 조회**

<img class="main-image" src="/assets/images/h2console_log.png" alt="H2 콘솔 화면">

## 마무리

DB의 데이터 처리 과정에 암·복호화 로직의 실행을 자연스럽게 통합시킬 수 있도록 확장 포인트를 제공하는 `AttributeConverter`를 활용해서 애플리케이션 레벨 암호화를 간단하게 구현할 수 있었습니다. 하지만, 높은 수준의 보안성이 보장되는 것은 아닙니다. 그래서 실무에 적용하기 위해서는 암호화 알고리즘/키 길이/IV 처리/안전한 키 저장 등에 대한 충분한 검토가 필요합니다.

## 포스팅 시리즈

* [[토이 프로젝트] DB 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[토이 프로젝트] DB 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] DB 컬럼 암호화: (3) 프로젝트 구조 살펴보기](https://sanghoon-lee.github.io/2026/01/26/JPACryptoConverter3/)
* [[토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기](https://sanghoon-lee.github.io/2026/01/27/JPACryptoConverter4/)
* [토이 프로젝트] DB 컬럼 암호화:  (5) 동작의 흐름 및 제약사항

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

## 관련 포스팅

* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)