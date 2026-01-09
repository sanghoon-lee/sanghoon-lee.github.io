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

스프링부트에서는 개발 편의성을 보장하기 위해서 다양한 어노테이션이 제공됩니다. 하지만, 어노테이션의 숫자가 너무 많다보니 사용법을 정확히 숙지하지 못한 채, 경험에 의존해서 습관적으로 사용하는 경우도 적지 않습니다. 

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

저의 경우에는 @MappedSuperclass 어노테이션을 종종 사용하면서도 타인에게 사용법을 명확히 설명하기는 어려웠습니다. 머릿속에 개념이 명확하게 정리되어 있지 못해서 그랬던 것 같습니다. 

하지만, 최근 혼자 토이 프로젝트를 진행하면서 우연히 @MappedSuperClass에 대해서 공부할 수 있었습니다. 그 때 공부했던 내용을 오랫동안 기억하고 싶어서 블로그에 통해서 정리해봤습니다.

### @MappedSuperclass를 사용하는 이유

데이터 구조를 설계하다보면, 여러 ```엔티티(Entity)``` 에서 공통으로 사용할 수 있는 ```속성(Attribute)```들이 보이게 됩니다.

예를 들면 다음과 같은 속성들입니다.
- ID
- 생성일(createdAt)
- 변경일(updatedAt)
- 생성자(createdBy)
- 변경자(updatedBy)

이런 경우, 자식 클래스에서 상속받아서 사용할 수 있도록 공통 속성들을 부모 클래스에 정의하게 됩니다. 하지만, 부모 클래스가 @Entity로 선언되면 의도하지 않은 문제들이 발생할 수 있습니다. 

1️⃣ 불필요한 테이블 생성

```java
@Entity
public abstract class BaseEntity {
    @Id
    private Long id;

    private LocalDateTime createdAt;
}
```
이렇게 부모 클래스를 선언하면 스프링부트의 JPA는 BaseEntity를 독립적인 엔티티로 인식하게 됩니다. 그리고 실제로는 사용하지 않는 BaseEntity와 매핑되는 테이블이 DB에 생성됩니다.

DB에 테이블 생성없이 공통적으로 사용할 수 있는 속성들만 정의하려던 원래 목적과는 다르게 불필요한 테이블이 생성됩니다.

2️⃣ 설계 혼란 유발

```java
public interface BaseEntityRepository
        extends JpaRepository<BaseEntity, Long> {
}
```

BaseEntity와 매핑되는 테이블이 생성되면, Repository 생성의 대상이 되기 때문에 설계 의도가 흐려지게 됩니다. 기술적으로는 가능하지만,
* “이 Repository를 실제로 어디서 써야 하지?”
* “이 엔티티는 무슨 책임을 가지는 거지?”

와 같은 설계 혼란을 유발할 수 있습니다.

3️⃣ 상속 전략(@Inheritance)의 고민 요구

대부분의 경우, 공통 속성의 상속에는 상속 전략 자체가 필요하지 않습니다. 속성만 재사용하고 싶은데, 테이블 구조의 설계까지 함께 고민해야 하는 상황이 됩니다

4️⃣ 의도하지 않은 다형성 조회 가능성

```sql
select b from BaseEntity b
```

부모 클래스가 @Entity로 선언되면, 위와 같은 JPQL 조회가 가능해집니다. 하지만, BaseEntity는 직접 조회할 의도가 없는 클래스인 경우가 대부분입니다. 조회할 필요가 없는 대상이 조회가 가능한 엔티티가 되어 버립니다.
