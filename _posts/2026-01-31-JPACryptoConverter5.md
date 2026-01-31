---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (5) 동작방식 및 제약사항"
date: 2026-01-31
categories: 토이프로젝트
---

지난 포스팅까지 다룬 내용을 바탕으로,
이번 포스팅에서는 실제 코드가 어떤 흐름으로 동작하는지를
차례대로 따라가 보려고 합니다.

그에 앞서, 먼저 엔티티가 어떻게 정의되어 있고
그중 어떤 필드가 암호화 대상인지를 정리해보겠습니다.

## 엔티티의 정의

애플리케이션 레벨 암호화의 동작을 확인하기 위해 다음과 같은 구조의 엔티티를 사용했습니다.

| 필드명 | 데이터 타입 | 내용 | 암호화 대상 |
| --- | --- | --- | ---- |
| id | Long | ID | X |
| phoneNumber | String | 전화번호(예 : 010-1234-5678) | O |
| userName | String | 이름(예 : 홍길동) | X |
| sex | int | 성별 - 1:남성,0:여성 | X |
| age | int | 나이 | X |

전화번호는 개인정보에 해당하기 때문에, 실무에서도 당연히 암호화가 필요한 대상입니다. 그래서 암호화 대상 속성으로 전화번호(`phoneNumber`)를 선택했습니다.

`Account` 엔티티는 다음과 같이 정의되어 있습니다.

```java
@Entity
public class Account extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 저장 시 암호화, 조회 시 복호화
    @Convert(converter = StringEncryptConverter.class)
    private String phoneNumber;

    private String userName;
    private int sex;
    private int age;
}
```

## 관련 포스팅

* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)