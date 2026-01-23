---
layout: post
title: "[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우"
date: 2026-01-21
categories: 트러블슈팅
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

개발팀장 역할을 맡으면서 회의와 일정 관리, 의사결정, 보고 등 관리 업무에 대부분의 시간을 사용하고 있습니다. 그러다 보니 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 팀원들과 실무적인 이야기를 나누다 보면 항상 경험과 지식의 부족을 느끼게 됩니다. 그러다보니 자신있게 의사결정하기 어려운 순간들도 종종 있습니다. 그래서 틈틈이 혼자 토이 프로젝트라도 진행해보면서 부족함을 채워보려고 노력하고 있습니다. 하지만, 그 정도로 충분하다고 말하기는 어려운 것 같습니다.

JPA는 사실 제가 실무에서 깊게 다뤄본 경험이 없는 기술 중 하나입니다. 제대로 공부했다고 말하기에도 아직은 조심스러운 수준입니다. 그래도
팀에서 현재 사용하고 있는 기술이기에 최소한의 개념이라도 제대로 정립해보고 싶었습니다. 그래서 최근에 아주 간단한 토이 프로젝트를 통해서
JPA를 직접 사용해봤습니다. JPA Auditing 기능을 활성화해서 데이터의 생성일과 수정일 정보가 자동으로 관리되도록 하고 싶었습니다.
단순하지만, 실무에서 자주 사용되는 기능으로 한번쯤은 직접 코드를 구현해보고 싶었거든요. 

하지만, 사소한 설정 하나를 놓친 탓에 생각보다 많은 시간을 문제 해결에 소비해야만 했습니다. JPA에 대한 기초가 부족한 것이 원인이었습니다.
반성하고, 앞으로 더 열심히 공부해야 할 것 같습니다. 

### 문제의 시작

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

JPA Auditing 기능의 사용을 위해 부모 엔티티에는 `@EntityListeners` 어노테이션을 통해 `AuditingEntityListener`도 등록했습니다. `createdAt`과 `updatedAt` 필드에 각각 `@CreatedDate`, `@LastModifiedDate` 어노테이션을 붙여서 생성일과 수정일로 인식되도록 했습니다. 이렇게만 하면 JPA Auditing 기능이 동작할 것으로 기대했습니다.

즉, 기대했던 결과는 다음과 같았습니다.

* 데이터 INSERT → 자동으로 생성일 기록
* 데이터 UPDATE → 지동으로 수정일 기록

하지만, 기대했던 결과가 나오지는 않았습니다. 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 아래 그림처럼 **생성일과 수정일이 모두 `null`**로 저장되고 있었습니다. 

<img class="main-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면(생성일과 수정일이 null로 기록)">

“설정 하나쯤 빠졌겠지”라는 생각은 들었지만, 막상 어디가 문제인지 바로 떠오르지는 않았습니다. 

### 원인

사실 원인은 아주 단순했습니다. @EnableJpaAuditing 어노테이션을 선언하는 것이 빠져 있었기 때문에, JPA Auditing 기능이 활성화되지 않았던 것입니다. 부모 엔티티에서 @EntityListener 어노테이션을 통해 등록했던 AuditingEntityListener는 이름 그대로 리스너일 뿐입니다. 그것만으로는 JPA Auditing 기능이 활성화되지는 않습니다. 제가 이걸 놓치고 있었던 것입니다. 

### 해결 방법

문제를 해결하기 위해서는 JPA Auditing 기능을 명시적으로 활성화해야 합니다. 이를 위해 Config 클래스를 하나 생성하고, @EnableJpaAuditing 어노테이션을 선언하면 됩니다.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

별도의 Config 클래스를 추가하지 않고, 아래처럼 메인 클래스에서 직접 @EnableJpaAuditing 어노테이션을 선언해도 상관은 없습니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

이후, JPA Auditing 기능이 정상적으로 동작했습니다. 데이터를 INSERT하고, H2 콘솔로 접속해서 저장된 데이터를 조회해보면 아래 그림처럼 생성일과 수정일에 모두 정상적인 값이 저장되어 있는 것을 확인할 수 있었습니다.
<img class="main-image" src="/assets/images/h2console2.png" alt="H2 콘솔 화면(생성일과 수정일이 정상적으로 기록)">

### 정리하며...

JPA Auditing을 사용하려면 **기능 활성화의 조건**을 정확히 이해하는 것이 중요합니다. 혹시라도 저와 비슷한 문제를 겪고 계신 분들에게 이 글이 작은 도움이 되기를 바랍니다. 그리고 개발팀장이라고 다 개발을 잘하는 것은 아닙니다....^__^

