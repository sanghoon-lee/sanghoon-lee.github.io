---
layout: post
title: "[트러블슈팅] Spring JPA Auditing이 동작하지 않는 문제 해결: @EnableJpaAuditing 설정 누락"
date: 2026-01-21
description: Spring Data JPA에서 @CreatedDate와 @LastModifiedDate가 null로 저장되는 문제를 해결하면서 @EnableJpaAuditing 설정과 AuditingEntityListener의 역할을 정리한 글입니다.
image: /assets/images/troubleshooting.png
categories: 트러블슈팅
tags:
  - 자바
  - DB
---

Spring Data JPA에서 생성일과 수정일을 자동으로 관리할 수 있도록 `@CreatedDate`와 `@LastModifiedDate`를 사용했지만, 데이터 저장 후에 값이 null로 저장되는 문제가 발생했습니다.

`@EntityListeners(AuditingEntityListener.class)`는 등록되어 있었지만, JPA Auditing 기능 자체를 활성화하는 `@EnableJpaAuditing` 설정이 빠져 있었던 것이 원인이었습니다.

---

## 1. 문제의 시작

제가 원했던 것은 엔티티 이벤트를 감지하는 JPA Auditing 기능을 활용해서 데이터의 생성일과 수정일 정보가 애플리케이션에서 자동으로 관리되도록 구현하는 것이었습니다. 

그래서 **createdAt**과 **updatedAt** 필드에 각각 `@CreatedDate`, `@LastModifiedDate` 어노테이션을 붙여서 생성일과 수정일로 인식되도록 선언했습니다. 

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    ...
}
```

**참고**

> `@EntityListeners(AuditingEntityListener.class)`는 엔티티 이벤트를 감지하기 위한 리스너 등록 설정입니다.

이어서 엔티티로 사용할 `Account`가 `BaseEntity`를 상속받도록 구현했습니다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Account extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String userName;
    private int sex;
    private int age;
    ....
```

이렇게 하면 `BaseEntity`를 상속받은 `Account` 엔티티에 데이터가 추가되거나 갱신될 때, 생성일과 수정일이 자동으로 기록될 것으로 기대했습니다. 

**참고**

> @MappedSuperclass의 사용방법이나 엔티티의 개념은 이전에 작성했던 [[학습기록] JPA @MappedSuperclass로 공통 엔티티 속성 관리하기](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)에서 다룬 적이 있습니다.

---

## 2. 기대와 다른 동작

기대했던 결과는 다음과 같았습니다.

* 데이터 INSERT → 자동으로 생성일 기록
* 데이터 UPDATE → 자동으로 수정일 기록

하지만 막상 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 기대와 다르게 생성일과 수정일이 모두 **null**로 기록되고 있었습니다. UPDATE도 동일한 결과였습니다.

<img class="sub-image" src="/assets/images/h2console.png" alt="JPA Auditing 설정 누락으로 createdAt updatedAt 값이 null로 저장된 H2 콘솔 화면">

<mark>“설정 하나쯤 빠졌겠지”</mark>라는 생각은 들었지만, 막상 어디가 문제인지 바로 떠오르지는 않았습니다. 

---

## 3. 문제의 해결

사실 문제가 발생한 원인은 아주 단순했습니다. `@EnableJpaAuditing` 어노테이션을 선언하지 않아서 JPA Auditing 기능이 활성화되지 못했던 것입니다. 

`BaseEntity`에서 `@EntityListeners(AuditingEntityListener.class)`를 통해 엔티티 이벤트를 감지하기 위한 리스너를 등록했습니다. 하지만, 그것만으로는 JPA Auditing 기능이 활성화되지는 않습니다. 

제가 이걸 놓치고 있었던 것입니다. 

Config 클래스를 하나 생성하고, `@EnableJpaAuditing` 어노테이션을 선언하는 것으로 문제는 즉시 해결되었습니다. 

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

별도의 Config 클래스를 추가하지 않고, Application 클래스에서 직접 `@EnableJpaAuditing` 어노테이션을 선언해도 상관은 없습니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

이후에 JPA Auditing 기능이 정상적으로 동작하면서 생성일과 수정일에 정상적인 값이 기록되었습니다.

<img class="sub-image" src="/assets/images/h2console2.png" alt="생성일과 수정일이 정상적으로 기록된 H2 콘솔 화면">

---

## 4. 마무리

이번 문제는 단순히 어노테이션 하나를 빠뜨린 실수였습니다. 하지만 각 설정이 어떤 역할을 하는지 이해하는 것이 중요하다는 점을 다시 한번 느낄 수 있었습니다.

개발팀장 역할을 맡으면서 회의와 일정 관리, 의사결정, 보고 등 관리 업무에 대부분의 시간을 사용하고 있습니다. 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 이제는 팀원들과 실무적인 이야기를 나누다 보면 경험과 지식의 부족함을 많이 느끼기도 합니다. 

그러다 보니 자신있게 의사결정하기 어려운 순간들도 종종 있습니다. 그래서 틈틈이 혼자 토이프로젝트라도 진행해보면서 부족함을 채워보려고 노력하고 있습니다. 하지만, 충분하다고 말하기는 어려울 것 같습니다.

앞으로도 실무에서 자주 사용하는 기술들을 직접 구현하고 검증하면서, 단순히 사용하는 것을 넘어 내부 동작 원리까지 이해할 수 있도록 꾸준히 학습하려고 합니다.