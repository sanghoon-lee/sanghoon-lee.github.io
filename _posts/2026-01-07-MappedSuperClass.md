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

스프링부트에서는 개발 편의성을 보장하기 위해서 다양한 어노테이션이 제공됩니다. 하지만, 너무 많다보니 모든 어노테이션의 사용법을 정확히 숙지하고 사용하기는 사실상 어려웠습니다. 그냥 경험에 의존해서 습관적으로 사용하는 경우도 적지 않습니다. 

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

@MappedSuperclass도 그 중 하나였습니다. 종종 사용하면서도 머릿속에 개념을 명확하게 정리하지 못해서 타인에게 명확하게 설명하기는 어려웠습니다. 다행히 최근 혼자 토이 프로젝트를 진행하면서 @MappedSuperClass에 대해서 공부하면서 부족했던 지식을 조금은 채울 수 있었습니다. 

---

### 엔티티와 속성

우선 가장 기본적인 ```엔티티(Entity)```와 ```속성(Attribute)```에 대한 개념부터 명확히 정리했습니다.

**엔티티**
> 데이터베이스에서 관리하고 저장해야 객체로 물리 모델에서는 테이블로 표현(예 : 사원, 제품, 주문목록 등)

**속성** 
> 엔티티의 최소 단위로 물리 모델에서는 컬럼으로 표현(예 : 사번, 이름, 나이 등)

---

### 다형성 조회

JPA에서의 디형성 조회(Polymorphic Query)는 부모 엔티티를 기준으로 조회했을 때, 부모와 모든 자식 엔티티까지 함께 조회되는 것을 의미합니다.

아래 예시와 같은 경우에 유용합니다.

* 결제(개념) : 부모 엔티티
* 결제수단(구현체) : 자식 엔티티(카드, 포인트, 현금 등)

다형성 조회 기능을 활용하면 결제 전체 내역을 한 번에 보고 싶을 때, 구현체가 무엇이든 '결제'라는 개념으로 묶어서 조회가 가능합니다.

---

### 공통 속성의 재사용

실무에서 데이터 구조를 설계하다보면 ```엔티티```들이 공통적으로 사용할 수 있는 ```속성```들이 있습니다.

예를 들면 다음과 같은 속성들입니다.
- ID
- 생성일(createdAt)
- 변경일(updatedAt)
- 생성자(createdBy)
- 변경자(updatedBy)

일반적으로 부모 엔티티에 속성들을 정의하고, 자식 엔티티에서 상속받아서 사용합니다. 

공통적으로 사용할 수 있는 속성을 각 엔티티마다 개별적으로 선언해서 사용하게 되면 다음과 같은 문제가 발생하게 됩니다.

* 중복 코드 증가
* 유지보수 비용 증가 
* 컬럼 정책 변경 시 누락 위험

---

### @Entity와 @MappedSuperclass 

JPA에서는 @Entity로 선언하면 엔티티로 인식해서 연결된 테이블을 DB에 생성하게 됩니다. 하지만, @MappedSuperClass로 선언하면 엔티티로 인식되지 않기 때문에, 테이블은 생성하지 않고 속성들만 담을 수 있습니다.

**@Entity와 @MappedSuperClass 비교**

| 구분	| @Entity |	@MappedSuperclass|
| ---- | --- | --- |
| 테이블 생성| 	O	 | X |
| 상속 목적	| 다형성 | 공통 속성 재사용 |
| 조회 가능 |	O |	X |
| Repository 생성	| O	| X |

@MappedSuperClass를 사용하면 부모 엔티티가 DB에서 조회의 대상이 아닌 공통 속성만 정의하고 있는 순수한 기본 엔티티임을 명확하게 표현할 수 있습니다.

단순히 공통 속성의 재사용이 목적이고, 다형성 조회가 필요하지 않다면 @Entity 대신 @MappedSuperclass를 사용하는 것이 적합합니다.

---

### 부모 엔티티를 @Entity로 선언하는 경우에 발생할 수 있는 문제

부모 엔티티가 @Entity로 선언되면 아래와 같이 의도하지 않은 문제들이 발생할 수 있습니다. 

1️⃣ 불필요한 테이블 생성

```java
@Entity
public abstract class BaseEntity {
    @Id
    private Long id;

    private LocalDateTime createdAt;
}
```
위의 코드처럼 @Entity로 엔티티 클래스를 선언하면 JPA는 독립적인 엔티티로 인식하게 됩니다. 즉, BaseEntity에 정의된 속성들을 저장하기 위한 테이블이 생성되는 것입니다.

하지만, BaseEntity는 부모 엔티티(순수한 기본 엔티티)로 상속받은 자식 엔티티들이 공통으로 사용할 수 있는 속성들만 정의하는 것이 원래의 목적입니다. 테이블도 따로 생성할 필요는 없습니다.  

2️⃣ 설계 혼란

```java
public interface BaseEntityRepository
        extends JpaRepository<BaseEntity, Long> {
}
```

BaseEntity와 연결된 테이블이 생성되면 Repository 생성의 대상이 되기도 합니다. 물론, 위와 같이 Reposiroty를 생성해도 됩니다. 하지만, 
* “이 Repository를 실제로 어디서 써야 하지?”
* “이 엔티티는 무슨 책임을 가지는 거지?”

와 같은 설계 의도가 흐려지면서 혼란을 유발할 수 있습니다.

3️⃣ 의도하지 않은 다형성 조회 가능성

```sql
select b from BaseEntity b
```

BaseEntity를 상속받은 자식 엔티티 ChildEntityA와 ChildEntityB가 있다고 가정하면 위의 SQL문으로 BaseEntity, ChildEntityA, ChildEntityB가 한번에 조회됩니다.

---

