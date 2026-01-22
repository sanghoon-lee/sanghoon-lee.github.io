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

개발팀장으로 회의와 일정 관리, 의사결정, 보고 등 관리자 역할에 업무 시간의 대부분을 사용하게 되었습니다. 그러다 보니 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 그렇다고 해서 개발에 대한 감각을 완전히 놓고 싶은 것은 아닙니다. 그래서 가끔씩은 혼자 토이 프로젝트를 수행하면서 코드를 직접 작성해보기도 합니다. 다만, 함께 일하고 있는 팀원들과 비교하면 여전히 부족함을 많이 느낍니다. 이제 관리자 역할에는 익숙해졌지만, 코드 작성에는 조금 서툰 사람이 되어버린 것 같습니다.

### 문제의 시작

최근 수행했던 토이 프로젝트에서 엔티티(Entity)의 생성일과 수정일을 자동으로 관리하고 싶었습니다. 그래서 부모 엔티티(BaseEntity)에서 @CreatedDate, @LastModifiedDate 어노테이션을
생성일과 수정일 속성을 표현할 변수에 붙였습니다. 부모 클래스에 Auditing 설정을 해두었습니다. 이렇게하면 부모 엔티티를 상속받은 자식 엔티티의 생성일과 수정일까지도 자동으로 관리될 것이라 생각했습니다.

**부모 엔티티**
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
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

    // 저장시 암호화, 조회시 복호화
    @Convert(converter = StringEncryptConverter.class)
    private String phoneNumber;

    private String userName;
    private int sex;
    private int age;
    ....
```

DB에 데이터를 저장하는 테스트도 정상적으로 동작하는 것처럼 보였습니다. 하지만 H2 콘솔에 접속해서 저장된 결과를 조회해보니 기대했던 결과와 다르게
생성일과 수정일에 null이 저장되어 있었습니다. “이 정도면 설정 하나쯤 빠졌겠지”라는 생각은 들었지만,
막상 어디가 문제인지 바로 떠오르지는 않았습니다.

<img class="main-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면">

### 원인

원인은 아주 단순했습니다. @EnableJpaAuditing 설정이 빠져 있었습니다. AuditingEntityListener는 이름 그대로 리스너일 뿐이고,
Spring Data JPA의 Auditing 기능 자체는 명시적으로 활성화해야 동작합니다. Auditing이 비활성화된 상태에서는 @CreatedDate, @LastModifiedDate가 붙어 있어도 아무 값도 자동으로 채워지지 않습니다.

### 해결 방법

Auditing 설정 클래스를 하나 추가하고 @EnableJpaAuditing을 선언합니다.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

또는 메인 클래스에 직접 추가해도 됩니다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {
}
```

설정 후에는 다음과 같이 정상 동작합니다.

* INSERT 시 → @CreatedDate 자동 세팅
* UPDATE 시 → 변경 감지(dirty checking)가 발생하면 @LastModifiedDate 자동 갱신

### 정리하며

Auditing 어노테이션은 리스너 등록이고,
Auditing을 실제로 동작시키는 스위치는 @EnableJpaAuditing입니다.

그리고 개인적으로는 한 줄을 더 덧붙이고 싶습니다.

팀장을 오래 해도, 기본 설정은 가끔 까먹는다.

이번 이슈는 아주 사소한 설정 하나였지만,
지금의 제 상태를 돌아보게 만드는 계기가 되었습니다.
그래서 더더욱 이런 실수일수록 기록으로 남겨두는 것이 중요하다고 느꼈습니다.