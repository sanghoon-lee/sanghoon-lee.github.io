---
layout: post
title: "[학습기록] JPA @MappedSuperclass로 공통 엔티티 속성 관리하기"
date: 2026-01-07
description: JPA @MappedSuperclass의 개념과 동작 원리를 정리하고, BaseEntity를 활용한 공통 엔티티 속성 관리 방법과 @Entity 상속과의 차이점을 정리합니다.
image: /assets/images/springboot.png
categories: 학습기록
tags:
  - 자바
  - 스프링
  - DB
---

Spring을 기반으로 애플리케이션을 개발하다 보면 다양한 어노테이션을 사용하게 됩니다. 특히 JPA에서 제공하는 어노테이션은 단순한 설정 이상의 의미를 가지고 있기 때문에 내부 동작과 사용 목적을 정확히 이해하는 것이 중요합니다.

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

하지만 저의 경우에는 사용법을 정확히 알고 사용하기 보다는 대부분 경험에 의존해서 습관적으로 사용하고 있었습니다. 그러다보니 실질적으로 다른 사람에게 자신있게 사용법을 설명할 수 있는 어노테이션이 많지 않았습니다.

@MappedSuperclass은 실무에서 자주 사용하는 어노테이션입니다. 하지만 **"어떤 경우에 사용해야 되는지?"**, **"어떻게 동작하는지?"** 등 머릿속에 개념이 명확하게 정리되어 있지 않았습니다. 

---

## 1. 엔티티와 속성

데이터 구조 설계와도 관련성이 있는 내용이라 우선 **엔티티(Entity)**와 **속성(Attribute)**에 대한 정의부터 명확히 확인하고 시작해야 할 것 같습니다. 

**엔티티**
> 애플리케이션에서 관리하는 데이터 객체로 JPA에서는 해당 객체를 데이터베이스 테이블과 매핑하여 영속적으로 관리합니다.

**속성** 
> 엔티티 객체의 상태를 표현하는 구성 요소이며, DB 관점에서는 컬럼(Column)에 해당합니다. 

아래 예제처럼 회원정보라는 엔티티에는 식별자뿐만 아니라, 이름, 전화번호, 이메일주소와 같은 여러 속성이 존재할 수 있습니다.
```
회원정보 (엔티티)
   ├─ id (PK, 식별자)
   ├─ 이름 (속성)
   ├─ 전화번호 (속성)
   ├─ 이메일주소 (속성)
   ... 
```

엔티티와 테이블은 동일한 개념은 아니지만, 일반적인 단일 테이블 매핑 구조에서는 하나의 엔티티가 하나의 테이블과 대응되는 경우가 많습니다. 

---

## 2. @MappedSuperclass의 사용 목적

실무에서 데이터 구조를 설계하다보면 여러 **엔티티**에서 공통적으로 사용할 수 있는 **속성**들이 보입니다.

다음은 공통적으로 사용할 수 있는 대표적인 속성입니다.

* 생성일(createdAt)
* 변경일(updatedAt)
* 생성자(createdBy)
* 변경자(updatedBy)

공통적으로 사용할 수 있는 속성을 각각의 엔티티마다 정의해서 사용하게 되면, 다음과 같은 문제가 발생하게 됩니다. 

* 중복 코드 증가
* 유지보수 비용 증가 
* 컬럼 정책 변경 시 누락 위험

이러한 문제를 회피하기 위해서 아래 예제 코드처럼 @MappedSuperclass로 클래스를 선언할 수 있습니다.

```java
@MappedSuperclass
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Entity
public class ChildEntity extends BaseEntity{
    private String userName;
    private Integer age;
}

```

@MappedSuperclass는 엔티티 간 공통 매핑 정보를 상속하기 위한 기능입니다. 해당 클래스 자체는 엔티티가 아니지만, 상속받은 자식 엔티티의 테이블에 부모 클래스의 속성이 함께 매핑됩니다. 따라서 ChildEntity는 BaseEntity로부터 createdAt과 updatedAt 속성을 상속받아 사용할 수 있습니다.

---

## 3. @Entity와 @MappedSuperclass의 차이점

JPA는 @Entity로 선언된 객체를 영속성 관리 대상으로 인식합니다. 일반적으로 엔티티는 데이터베이스 테이블과 매핑되며, JPA 설정에 따라 해당 테이블 구조가 관리됩니다.

반면, @MappedSuperclass로 선언된 클래스는 엔티티로 인식되지 않으며 자체적인 테이블도 생성되지 않습니다. 다만 상속받은 자식 엔티티에서는 부모 클래스의 속성이 함께 매핑됩니다.

| 구분            | @Entity | @MappedSuperclass |
| ------------- | ------- | ----------------- |
| JPA 관리 대상     | O       | X                 |
| 독립 엔티티 가능     | O       | X                 |
| 자체 테이블 매핑     | O       | X                 |
| 공통 속성 재사용     | 가능      | O                 |
| Repository 대상 | O       | X                 |
| JPQL 조회       | 가능      | 불가능               |
| 목적            | 엔티티 모델링 | 공통 매핑 정보 재사용      |


단순히 공통적으로 사용할 수 있는 속성만 제공하고, 다형성 조회(Polymorphic Query)는 필요하지 않은 상황이라면 @Entity가 아닌 @MappedSuperclass를 사용하는 것이 적합합니다. 

다형성 조회는 부모 엔티티 타입으로 조회했을 때 상속 관계에 있는 자식 엔티티까지 함께 조회할 수 있는 기능입니다. 

예를 들어 결제라는 부모(Payment) 엔티티와 카드결제(CardPayment), 포인트결제(PointPayment), 현금결제(CashPayment) 같은 자식 엔티티가 존재하는 상속 구조가 있다고 가정해보겠습니다.

```java
// 부모 엔티티
Payment

// 자식 엔티티
CardPayment
PointPayment
CashPayment
```

이런 구조에서는 부모 타입으로 조회하여 자식 엔티티에 해당되는 모든 결제 정보를 조회할 수 있습니다.

```jpql
select p from Payment p where p.amount > 10000
```

이처럼 부모와 자식 엔티티 간의 관계가 필요하다면 @MappedSuperclass가 아닌 JPA 엔티티 상속 전략을 사용해야 합니다.

---

## 4. @Entity와 @MappedSuperclass를 잘못 선택했을 때 발생하는 문제

@MappedSuperclass를 사용하는 것이 적합한 상황임에도 불구하고, @Entity를 사용하게 되면 다음과 같은 문제가 발생하게 됩니다.

---

### 4.1. 의도하지 않은 엔티티 관리

단순히 공통 속성을 재사용하기 위한 클래스라면 @Entity보다는 @MappedSuperclass가 적합합니다. 

@MappedSuperclass는 엔티티가 아니기 때문에 별도의 영속성 관리 대상이 되지 않습니다. 반면 @Entity로 선언하면 JPA가 관리하는 엔티티가 되기 때문에 상속 구조, 테이블 매핑 전략 등 추가적인 설계가 필요합니다.

아래 코드의 `BaseEntity` 클래스는 공통으로 사용할 수 있는 속성들을 정의하는 목적만 가지고 있습니다.  

```java
@Entity
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

이런 경우 @Entity 어노테이션을 사용하면 JPA가 관리하는 엔티티가 되면서 불필요한 관리 요소만 증가하게 될 수 있습니다.

---

### 4.2. 설계 혼란

JPA 관리 대상 엔티티가 되면 Repository를 생성할 수 있지만, BaseEntity는 실제 조회 대상이 아니라 공통 속성 관리를 위한 클래스입니다.

따라서 불필요한 엔티티와 Repository가 추가되면서 도메인 모델의 의도가 불명확해질 수 있습니다.

---

### 4.3. 엔티티 책임 혼란

공통 속성을 관리하기 위한 `BaseEntity`가 엔티티로 등록되면, 실제 도메인 모델인지 단순 공통 객체인지 구분이 어려워질 수 있습니다.

JPA에서는 엔티티가 의미 있는 도메인 객체를 표현해야 하기 때문에, 재사용 목적의 클래스와 엔티티 모델은 구분하는 것이 중요합니다.

---

## 5. 마무리

@MappedSuperclass는 공통 속성을 가진 부모 클래스를 정의하기 위한 JPA 기능입니다. 

특히 생성일, 수정일, 생성자 정보와 같이 여러 엔티티에서 반복적으로 사용하는 속성을 관리할 때 유용합니다. 다만 @MappedSuperclass는 엔티티 자체를 모델링하는 기능이 아니라, 단순히 매핑 정보를 재사용하기 위한 기능이라는 점을 이해해야 합니다.

만약 부모와 자식 엔티티 간의 관계를 표현하거나 다형성 조회가 필요하다면 @Entity 상속 전략을 검토해야 합니다.

결국 중요한 것은 어노테이션을 사용하는 것이 아니라, 해당 기능이 어떤 문제를 해결하기 위해 존재하는지 이해하고 적절한 상황에 사용하는 것입니다.

