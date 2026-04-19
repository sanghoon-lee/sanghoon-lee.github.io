---
layout: post
title: "[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제: @EnableJpaAuditing 빠진 경우"
date: 2026-01-21
description: JPA Auditing이 동작하지 않는 문제를 해결했던 과정을 정리한 글입니다.
image: /assets/images/troubleshooting.png
categories: 트러블슈팅
---

개발팀장 역할을 맡으면서 회의와 일정 관리, 의사결정, 보고 등 관리 업무에 대부분의 시간을 사용하고 있습니다. 

자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 이제는 팀원들과 실무적인 이야기를 나누다 보면 경험과 지식의 부족함이 많이 느끼지기도 합니다. 

그러다보니 자신있게 의사결정하기 어려운 순간들도 종종 있습니다. 

그래서 틈틈이 혼자 토이 프로젝트라도 진행해보면서 부족함을 채워보려고 노력하고 있습니다. 하지만, 충분하다고 말하기는 어려울 것 같습니다.

앞으로 더 열심히 공부해야 할 것 같습니다. 

---

## 1. 문제의 시작

JPA는 사실 제가 실무에서 깊게 다뤄본 경험이 없는 기술 중 하나입니다. 제대로 공부했다고 말하기에도 조심스러운 수준입니다. 

늦었지만 JPA에 대해서 공부를 시작하면서 토이 프로젝트에서 직접 사용해보고 있었습니다.

하지만 최근에 수행했던 토이 프로젝트에서는 사소한 설정 하나를 놓쳐 생각보다 많은 시간을 문제 해결에 소비해야 했습니다.

JPA에 대한 기초가 부족한 탓이었습니다.

---

## 2. 코드 구현

제가 원했던 것은 JPA Auditing 기능을 활용해서 데이터의 생성일과 수정일 정보가 애플리케이션에서 자동으로 관리되도록 구현하는 것이었습니다.

그래서 생성일과 수정일과 같이 공통으로 사용할 수 있는 속성들만 묶인 기본 객체를 정의하고 엔티티에서 이를 상속받아 사용하면 된다고 쉽게 생각이었습니다.

---

### 2.1. 기본 객체 정의

생성일과 수정일이 포함된 기본 객체 BaseEntity를 정의하면서 JPA Auditing 기능을 사용하기 위해 AuditingEntityListener를 등록했습니다.  

**createdAt과 **updatedAt** 필드에 각각 **@CreatedDate**, **@LastModifiedDate** 어노테이션을 붙여서 생성일과 수정일로 인식되도록 선언했습니다. 

이렇게만 하면 기본 객체를 상속받은 엔티티에 데이터가 추가되거나 갱신될 때, JPA Auditing 기능이 동작하면서 생성일과 수정일이 자동으로 기록될 것으로 기대했습니다. 

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
---

### 2.2. 엔티티 정의

기본 객체인 BaseEntity를 상속받은 Account 엔티티는 아래 코드처럼 정의했습니다.

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

---

## 3. 기대와 다른 동작

기대했던 결과는 다음과 같았습니다.

* 데이터 INSERT → 자동으로 생성일 기록
* 데이터 UPDATE → 지동으로 수정일 기록

하지만 막상 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 기대와 다르게 **생성일과 수정일이 모두 null**로 기록되고 있었습니다.

UPDATE도 동일한 결과였습니다.

<img class="sub-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면(생성일과 수정일이 null로 기록)">

“설정 하나쯤 빠졌겠지”라는 생각은 들었지만, 막상 어디가 문제인지 바로 떠오르지는 않았습니다. 

---

## 4. 문제의 해결

사실 문제가 발생한 원인은 아주 단순했습니다. @EnableJpaAuditing 어노테이션을 선언하지 않아서 JPA Auditing 기능이 활성화되지 못했던 것입니다. 

기본 객체인 BaseEntity에서 AuditingEntityListener를 등록했지만, 이름 그대로 리스너일 뿐입니다. 그것만으로는 JPA Auditing 기능이 활성화되지는 않습니다. 

제가 이걸 놓치고 있었던 것입니다. 

Config 클래스를 하나 생성하고, @EnableJpaAuditing 어노테이션을 선언하는 것으로 문제는 즉시 해결되었습니다. 

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

아래 코드처럼 별도의 Config 클래스를 추가하지 않고, Application 클래스에서 직접 @EnableJpaAuditing 어노테이션을 선언해도 상관은 없습니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

이후에 JPA Auditing 기능이 정상적으로 동작하는 것을 확인했습니다. 

이제는 데이터를 INSERT하거나 UPDATE하면 생성일과 수정일에 정상적인 값이 기록되었습니다.

<img class="sub-image" src="/assets/images/h2console2.png" alt="H2 콘솔 화면(생성일과 수정일이 정상적으로 기록)">

---

## 5. 마무리

이번 문제는 단순히 어노테이션 하나를 빠뜨린 것에서 시작된 것입니다. 

하지만, 그 과정에서 JPA Auditing은 “어노테이션을 붙이면 끝나는 기능”이 아니라, 명확한 활성화 조건을 이해하고 사용해야 하는 기능이라는 점을 다시 한 번 확인하게 되었습니다.

@EntityListeners는 리스너를 등록하는 설정일 뿐이고, @EnableJpaAuditing이 Auditing을 실제로 동작시키는 스위치라는 사실을 이번에 확실히 기억하게 되었습니다.

아주 사소한 실수였지만 비슷한 문제를 다시 겪지 않기 위해 그리고 같은 실수를 반복하지 않기 위해 이렇게 기록으로 남겨두려고 합니다. 

---

#JPAAuditing #JPA #트러블슈팅