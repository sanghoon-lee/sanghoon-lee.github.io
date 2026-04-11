---
layout: post
title: "[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제: @EnableJpaAuditing 빠진 경우"
date: 2026-01-21
description: JPA Auditing이 동작하지 않는 문제를 해결했던 과정을 정리한 글입니다.
image: /assets/images/troubleshooting.png
categories: 트러블슈팅
---

<style> 
.post-image { 
    width: 100%; 
    max-width: 800px; 
    height: auto; 
    margin-bottom: 1.4rem; 
    border-radius: 12px; 
    display: block; 
} 
</style>

개발팀장 역할을 맡으면서 회의와 일정 관리, 의사결정, 보고 등 관리 업무에 대부분의 시간을 사용하고 있습니다. 그러다 보니 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 이제는 팀원들과 실무적인 이야기를 나누다 보면 경험과 지식의 부족함이 많이 느끼지게 됩니다. 

그러다보니 자신있게 의사결정하기 어려운 순간들도 종종 있습니다. 그래서 틈틈이 혼자 토이 프로젝트라도 진행해보면서 부족함을 채워보려고 노력하고 있습니다. 하지만, 그 정도로 충분하다고 말하기는 어려운 것 같습니다.

JPA는 사실 제가 실무에서 깊게 다뤄본 경험이 없는 기술 중 하나입니다. 제대로 공부했다고 말하기에도 아직은 조심스러운 수준입니다. 그래도 최근에 아주 간단한 토이 프로젝트를 통해서 JPA를 직접 사용해봤습니다. 

JPA Auditing 기능을 활성화해서 데이터의 생성일과 수정일 정보가 자동으로 관리되도록 하고 싶었지만, 사소한 설정 하나를 놓친 탓에 생각보다 많은 시간을 문제 해결에 소비해야만 했습니다. JPA에 대한 기초가 부족한 것이 원인이었습니다.

<img class="main-image" src="/assets/images/troubleshooting.png" alt="트러블슈팅">

반성하고, 앞으로 더 열심히 공부해야 할 것 같습니다. 

---

## 1. 문제의 시작

생성일과 수정일과 같이 공통으로 사용될 수 있는 속성들은 부모 엔티티(BaseEntity)에서 정의히고, 자식 엔티티에서 상속받아 사용하도록 했습니다. 

**부모 엔티티**
```java

/**
 * 아래 코드는 BaseEntity에 JPA Auditing 설정을 해둔 부분입니다.
 * JPA가 엔티티에 데이터를 insert/update 할 때 자동으로 날짜를 채워줄 것으로 기대했습니다.
**/

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

**자식 엔티티**
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

JPA Auditing 기능을 사용하기 위해 부모 엔티티에는 `@EntityListeners` 어노테이션을 통해 `AuditingEntityListener`를 등록했습니다. 또한, `createdAt`과 `updatedAt` 필드에 각각 `@CreatedDate`, `@LastModifiedDate` 어노테이션을 붙여서 생성일과 수정일로 인식되도록 했습니다. 이렇게만 하면 JPA Auditing 기능이 동작할 것으로 기대했습니다.

기대했던 결과는 다음과 같았습니다.

* 데이터 INSERT → 자동으로 생성일 기록
* 데이터 UPDATE → 지동으로 수정일 기록

하지만, 막상 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 기대와 다르게 아래 그림처럼 **생성일과 수정일이 모두 `null`**로 저장되고 있었습니다. 

<img class="main-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면(생성일과 수정일이 null로 기록)">

“설정 하나쯤 빠졌겠지”라는 생각은 들었지만, 막상 어디가 문제인지 바로 떠오르지는 않았습니다. 

---

## 2. 문제의 원인

사실 원인은 아주 단순했습니다. @EnableJpaAuditing 어노테이션을 선언하는 것이 빠져 있어서 JPA Auditing 기능이 활성화되지 못했던 것입니다. 

부모 엔티티에서 @EntityListener 어노테이션을 통해 등록했던 AuditingEntityListener는 이름 그대로 리스너일 뿐입니다. 그것만으로는 JPA Auditing 기능이 활성화되지는 않습니다. 제가 이걸 놓치고 있었던 것입니다. 

---

## 3. 해결 방법

JPA Auditing 기능을 명시적으로 활성화하면 문제는 즉시 해결됩니다. Config 클래스를 하나 생성하고, @EnableJpaAuditing 어노테이션을 선언하면 됩니다.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

별도의 Config 클래스를 추가하지 않고, 아래 코드처럼 메인 클래스에서 직접 @EnableJpaAuditing 어노테이션을 선언해도 상관은 없습니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

이후, JPA Auditing 기능이 정상적으로 동작하는 것을 확인했습니다. 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 아래 그림처럼 생성일과 수정일에 모두 정상적인 값이 저장되어 있었습니다.
<img class="post-image" src="/assets/images/h2console2.png" alt="H2 콘솔 화면(생성일과 수정일이 정상적으로 기록)">

---

## 4. 정리하며...

이번 문제는 단순히 어노테이션 하나를 빠뜨린 것에서 시작된 것입니다. 하지만, 그 과정에서 JPA Auditing은 “어노테이션을 붙이면 끝나는 기능”이 아니라, 명확한 활성화 조건을 이해하고 사용해야 하는 기능이라는 점을 다시 한 번 확인하게 되었습니다.

`@EntityListeners`는 리스너를 등록하는 설정이고, Auditing을 실제로 동작시키는 스위치는 `@EnableJpaAuditing`이라는 사실을 이번에 확실히 기억하게 되었습니다.

아주 사소한 설정 하나였지만, 
비슷한 문제를 다시 겪지 않기 위해 
그리고 같은 실수를 반복하지 않기 위해
이렇게 기록으로 남겨두려고 합니다.

그리고 이 글은 팀장으로서 기본을 다시 점검하기 위한 기록이기도 합니다.

---

#JPAAuditing #JPA #트러블슈팅