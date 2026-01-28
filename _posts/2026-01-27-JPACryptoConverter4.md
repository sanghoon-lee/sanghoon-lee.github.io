---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기"
date: 2026-01-27
categories: 토이프로젝트
---

지난 포스팅들을 통해서 애플리케이션 레벨 암호화의 구현을 검토하게 된 계기와 구현 방식으로 JPA의 `AttributeConverter`를 선택하게 된 이유, 그리고 전체 프로젝트 구조까지 정리해봤습니다.

이제 어떤 코드 흐름을 통해 애플리케이션 레벨 암호화가 동작하는지를 핵심 코드 중심으로 이야기해볼 차례입니다. 복잡한 세부 구현사항을 일일이 설명하기 보다는 `AttributeConverter`를 중심으로 각 구성 요소가 어떤 역할을 맡고, 어떻게 연결되어 있는지에 초점을 맞춰 설명하겠습니다.

## AttributeConverter는 언제 호출되는가?

| 데이터 저장  | 데이터 조회 |
| ------ | ----- |
| 1. 애플리케이션에서 엔티티를 생성하고 값을 설정합니다. 이 시점에서 엔티티의 필드 값은 모두 **평문**입니다. <br>2. JPA가 엔티티를 DB에 저장하려고 할 때,`@Convert`가 선언된 필드를 발견합니다.<br>3. 해당 필드에 등록된 `AttributeConverter`의 `convertToDatabaseColumn()` 메서드가 호출됩니다.<br>4. 이 메서드에서 반환된 값이 DB 컬럼에 저장됩니다. | 1. JPA가 DB로부터 엔티티 데이터를 조회합니다. <br>2. `@Convert`가 선언된 컬럼을 읽어오는 과정에서 `AttributeConverter`가 다시 호출됩니다.<br>3. 이번에는 `convertToEntityAttribute()` 메서드가 실행됩니다.<br>4. DB에서 읽어온 값이 변환되어 엔티티 필드에는 **평문 값**이 설정됩니다. |

## 포스팅 시리즈

* [[토이 프로젝트] DB 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[토이 프로젝트] DB 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] DB 컬럼 암호화: (3) 프로젝트 구조 살펴보기](https://sanghoon-lee.github.io/2026/01/26/JPACryptoConverter3/)
* [토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기

## 관련 포스팅

* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)
