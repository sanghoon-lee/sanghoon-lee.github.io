---
layout: post
title: "[트러블슈팅] 스프링에서 JPA Auditing이 동작하는 않는 문제 - @EnableJpaAuditing 빠진 경우"
date: 2026-01-12
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
개발팀장 역할을 맡으면서 회의와 일정 관리, 의사결정, 보고 등 관리 업무에 대부분의 시간을 사용하고 있습니다. 그러다 보니 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 비교적 최근에 나온 기술들에 대해서는 팀원들과 이야기를 나누다 보면 경험과 지식의 부족이 금방 드러나기도 합니다. 그래서 틈틈이 혼자 토이 프로젝트를 진행하면서 부족함을 조금이라도 채워보려고 노력하고 있습니다. 다만, 그 노력이 충분하다고 말하기는 어려운 것 같습니다.

JPA도 사실 제가 실무에서 깊게 다뤄본 경험이 없는 기술 중 하나입니다. 제대로 공부했다고 말하기에도 아직은 조심스러운 수준입니다. 그럼에도 팀에서는 JPA를 사용하고 있기 때문에, 기초적인 개념만이라도 정리해보고 싶다는 생각으로 간단한 토이 프로젝트를 진행했습니다. JPA Auditing 기능을 활용해 DB 테이블의 생성일과 수정일을 자동으로 관리해보고 싶었습니다. 데이터가 추가되면(Insert) 생성일이 기록되고, 데이터가 갱신되면(Update) 수정일이 기록되는 단순하지만, 실무에서도 자주 쓰이는 기능입니다.

하지만 설정 하나를 놓친 탓에 생각보다 많은 시간을 문제 해결에 쓰게 되었고, 그 과정에서 지금의 제 상태를 다시 한 번 돌아보게 되었습니다.
사소한 실수였지만, 동일한 실수를 반복하지 않도록 기록으로 남겨놓고 싶어서 이 포스팅을 작성하게 되었습니다.

### 문제의 시작

공통으로 관리하고 싶은 속성들은 부모 엔티티(BaseEntity)에서 선언했습니다. 엔티티마다 생성일과 수정일을 반복해서 정의하는 대신, 공통 부모에서 한 번에 처리하고 싶었습니다.

JPA Auditing 기능을 사용하기 위해 부모 엔티티에는 `@EntityListeners` 어노테이션을 통해 `AuditingEntityListener`를 등록해두었습니다.
또한 `createdAt`과 `updatedAt` 필드에 각각 `@CreatedDate`, `@LastModifiedDate` 어노테이션을 붙여 생성일과 수정일을 자동으로 관리하도록 설정했습니다. 이렇게만 설정해두면, 엔티티가 저장되거나 수정될 때 생성일과 수정일이 자연스럽게 기록될 것이라고 기대했습니다. 하지만, 실제로는 그렇지 않았습니다.

**부모 엔티티**
```java

/**
 * 아래 코드는 BaseEntity에 Auditing 설정을 해둔 부분입니다.
 * JPA가 엔티티를 persist/update 할 때 자동으로 날짜를 채워줄 것으로 기대했습니다.
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

H2 콘솔로 접속해서 저장된 데이터를 조회해보면 아래 그림처럼 **생성일과 수정일이 모두 `null`**로 저장되고 있었습니다. 

<img class="main-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면(생성일과 수정일이 null로 기록)">

<br> “설정 하나쯤 빠졌겠지”라는 생각은 들었지만, 막상 어디가 문제인지 바로 떠오르지는 않았습니다. 

### 원인

사실 원인은 아주 단순했습니다. @EnableJpaAuditing 설정이 빠져 있어서 JPA Auditing 기능이 동작하지 않았던 것입니다. AuditingEntityListener는 이름 그대로 리스너일 뿐이고, JPA Auditing 기능은 반드시 명시적으로 활성화해야만 동작합니다. 제가 이걸 놓치고 있었던 것입니다. JPA Auditing이 비활성화된 상태에서는 아무리 @CreatedDate, @LastModifiedDate 어노테이션이 붙어 있어도 자동으로 값이 채워지지 않습니다.

### 해결 방법

JPA Auditing 설정을 위해서 클래스를 하나 추가하고, @EnableJpaAuditing을 선언해서 문제를 해결했습니다.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

별도의 클래스를 추가하지 않고, 아래처럼 메인 클래스에서 직접 선언해도 됩니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

설정 후에는 다음과 같이 정상적으로 동작하는 것을 확인할 수 있었습니다.

* INSERT 시 → @CreatedDate 자동 세팅
* UPDATE 시 → 변경 감지(dirty checking)가 발생하면 @LastModifiedDate 자동 갱신

문제를 해결한 다음에 다시 H2 콘솔에 접속해서 저장된 데이터를 조회했습니다. 이번에는 아래 그림처럼 생성일과 수정일에 모두 정상적인 값이 저장되어 있는 것을 알 수 있었습니다.
<img class="main-image" src="/assets/images/h2console2.png" alt="H2 콘솔 화면(생성일과 수정일이 정상적으로 기록)">

### 정리하며

이번 문제는 아주 작은 설정 하나였지만, JPA Auditing을 사용하려면 **기능 활성화의 조건**을 정확히 이해하는 것이 중요하다는 교훈을 남겼습니다. 비슷한 문제를 겪는 분들에게 이 글이 작은 도움이 되기를 바랍니다. 그리고 개발팀장이라고 다 개발을 잘하는 것은 아닙니다....^__^

