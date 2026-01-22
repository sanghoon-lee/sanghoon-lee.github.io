---
layout: post
title: "[트러블슈팅] 스프링에서 JPA Auditing이 동작하는 않는 문제 - @EnableJpaAuditing 빠진 경우"
date: 2026-01-12
categories: 트러블슈팅
---

<style>
.main-image {
  width: 100%;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

개발팀장으로 회의와 일정 관리, 의사결정, 보고 등 관리자 역할에 업무 시간의 대부분을 사용하게 되었습니다. 그러다 보니 자연스럽게 개발 실무와는 거리가 생기게 되었습니다. 그렇다고 해서 개발에 대한 감각을 완전히 놓고 싶은 것은 아닙니다. 그래서 가끔씩은 혼자 토이 프로젝트를 수행하면서 코드를 직접 작성해보기도 합니다. 다만, 함께 일하고 있는 팀원들과 비교하면 여전히 부족함을 많이 느낍니다. 이제 관리자 역할에는 익숙해졌지만, 코드 작성에는 조금 서툰 사람이 되어버린 것 같습니다.

### 문제의 시작

최근 수행했던 토이 프로젝트에서 엔티티(Entity)의 생성일과 수정일을 자동으로 관리하고 싶었습니다. 그래서 부모 엔티티(BaseEntity)에서 @CreatedDate, @LastModifiedDate 어노테이션을
생성일과 수정일 속성을 표현할 변수에 붙였습니다. 이렇게하면 부모 엔티티를 상속받은 자식 엔티티의 생성일과 수정일까지도 자동으로 관리될 것이라 생각했습니다.

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

DB에 데이터를 저장하는 테스트는 정상적으로 동작하는 것처럼 보였습니다. 하지만 H2 콘솔에 접속해서 저장된 결과를 조회해보니 기대했던 결과와 다르게
생성일과 수정일에 null이 저장되어 있었습니다.

<img class="main-image" src="/assets/images/h2console.png" alt="H2 콘솔 화면">

