---
layout: post
title: "[학습기록] 스프링부트 @MappedSuperClass 사용법"
date: 2026-01-07
categories: java
---

<style>
.main-image {
  width: 50%;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

스프링부트는 개발 편의성을 보장하기 위해서 다양한 어노테이션을 제공하고 있습니다. 실력이 부족해서 그런지 대부분의 어노테이션에 대해서 정확히 알고 사용한다기 보다는 그냥 경험에 의존해서 습관적으로 사용하고 있는 것 같은 느낌이었습니다. 실질적으로 다른 사람에게 자신있게 사용법을 설명할 수 있는 어노테이션은 몇개 안되는 것 같습니다. 

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

@MappedSuperclass도 실무를 하다보면 자주 보게되는 어노테이션입니다. 하지만, 얼마전까지도 "어떤 경우에 사용해야 되는지?" "어떻게 동작하는지?" 등 머릿속에 개념이 정확하게 정리되어 있지 않았습니다. 다행히, 최근 토이 프로젝트를 진행하면서 공부할 기회가 있었습니다. 그때 공부했던 내용을 이번 포스팅을 통해서 정리해봤습니다.

### 1. 엔티티와 속성

데이터 구조 설계와도 관련성이 있는 내용이라 우선 ```엔티티(Entity)```와 ```속성(Attribute)```에 대한 정의부터 명확히 확인하고 시작해야 할 것 같습니다. 

**엔티티**
> DB테이블과 연결되고, 식별자를 통해서 영속적으로 조회·관리되는 데이터의 단위입니다.

**속성** 
> 엔티티의 상태(state)를 표현하는 구성 요소로,
DB 관점에서는 테이블의 컬럼(Column)에 해당합니다. 

아래 예제처럼 회원정보라는 엔티티에는 식별자뿐만 아니라, 이름, 전화번호, 이메일주소와 같은 여러 속성이 존재할 수 있습니다.
```
회원정보 (엔티티)
   ├─ id (PK, 식별자)
   ├─ 이름 (속성)
   ├─ 전화번호 (속성)
   ├─ 이메일주소 (속성)
   ... 
```

쉡게 엔티티는 DB테이블이고, 속성은 테이블에 속한 컬럼이라고 이해해도 될 것 같습니다.

### 2. @MappedSuperClass의 사용 목적

실무에서 데이터 구조를 설계하다보면 여러 ```엔티티```에서 공통적으로 사용할 수 있는 ```속성```들이 보입니다.

다음은 공통적으로 사용할 수 있는 대표적인 속성입니다.
```
- 생성일(createdAt)
- 변경일(updatedAt)
- 생성자(createdBy)
- 변경자(updatedBy)
```

공통적으로 사용할 수 있는 속성을 각각의 엔티티마다 정의해서 사용하게 되면, 다음과 같은 문제가 발생하게 됩니다. 

* 중복 코드 증가
* 유지보수 비용 증가 
* 컬럼 정책 변경 시 누락 위험

이러한 문제를 회피하기 위해서 아래 예제 코드처럼 @MappedSuperclass로 클래스를 선언합니다. 이렇게 선언된 BaseEntity는 여러 엔티티에서 공통으로 사용할 수 있는 속성들만 묶인 기본 객체가 됩니다. 실제 엔티티들은 BaseEntity를 상속받아서 사용하도록 하면 됩니다.  

```java
@MappedSuperclass
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### 3. @Entity와 @MappedSuperClass의 차이점

스프링부트의 JPA는 @Entity로 선언된 객체를 엔티티로 인식해서 연결된 테이블을 DB에 생성하게 됩니다. 하지만, @MappedSuperClass로 선언된 객체는 엔티티로 인식되지 않습니다. 따라서, 객체와 연결된 테이블도 생성되지 않습니다.

| 구분	| @Entity |	@MappedSuperclass|
| ---- | --- | --- |
| 테이블 생성| 	O	 | X |
| 상속 목적	| 다형성 | 공통 속성 재사용 |
| 조회 가능 |	O |	X |
| Repository 생성	| O	| X |

단순히 공통적으로 사용할 수 있는 속성만 제공하고, 다형성 조회(Polymorphic Query)는 필요하지 않은 상황에서는 @Entity가 아닌 @MappedSuperclass를 사용하는 것이 적합합니다. 만약, 다형성 조회가 필요한 상황이라면 ```@Entity+@Inheritance 조합```을 사용하는 것이 적합할 것입니다. 나중에 기회가 된다면 이 조합에 대해서도 공부해서 별도로 포스팅을 해보도록 하겠습니다.

### 4. 다형성 조회

다형성 조회(Polymorphic Query)는 부모 엔티티를 기준으로 조회했을 때, 부모와 모든 자식 엔티티까지 함께 조회되는 것을 의미합니다.결제 전체 내역을 한 번에 보고 싶을 때, 다형성 조회 기능을 활용하면 결제수단이 무엇이든 '결제'라는 개념으로 묶어서 한번에 조회하는 것이 가능합니다. 아래는 그것과 관련된 예제 코드입니다.

```java
/**
 * 결제(개념) : 부모 엔티티(클래스명 : Payment)
 * 결제수단(구현체) : 자식 엔티티(카드, 포인트, 현금 등)
 */

select p from Payment p where p.amount > 10000
```

### 5. 잘못된 @Entity 사용으로 발생하는 문제

다형성 조회가 필요하지 않아서 @MappedSuperClass를 사용하는 것이 적합한 상황임에도 불구하고, @Entity를 사용하게 되면 다음과 같은 문제가 발생하게 됩니다.

1️⃣ 불필요한 테이블 생성

아래의 예제 코드처럼 @Entity로 객체를 선언하면 JPA는 독립적인 엔티티로 인식하고 연결된 테이블을 DB에 생성하게 됩니다. 하지만, BaseEntity는 부모 엔티티로 자식 엔티티에서 공통으로 사용할 수 있는 속성들을 정의하는 목적만 가지고 있었습니다. 따라서 테이블이 생성됨으로써 불필요하게 자원을 낭비하고, 관리 요소만 증가시켰습니다.

```java
@Entity
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

2️⃣ 설계 혼란

BaseEntity와 연결된 테이블이 생성되면, Repository 생성의 대상이 되기도 합니다. 아래 예제 코드처럼 Reposiroty를 생성해도 됩니다. 하지만, 설계 의도가 흐려지면서
* “이 Repository를 실제로 어디서 써야 하지?”
* “이 엔티티는 무슨 책임을 가지는 거지?”

와 같은 혼란을 유발할 수 있습니다.

```java
public interface BaseEntityRepository
        extends JpaRepository<BaseEntity, Long> {
}
```

3️⃣ 의도하지 않은 다형성 조회 가능성

BaseEntity는 원래 조회의 대상이 아니었습니다. 하지만, 엔티티로 인식되어 연결된 테이블이 생기면서 본래 의도와는 다르게 다형성 조회의 대상이 될 수 있습니다.

```sql
select b from BaseEntity b
```
