---
layout: post
title: "[토이 프로젝트] JPA 컬럼 암호화: (2) JPA AttributeConverter를 선택한 이유"
date: 2026-01-25
categories: 토이프로젝트
---

지난 포스팅에서는 DB 레벨 암호화와 애플리케이션 레벨 암호화 방식의 차이, 그리고 이 토이 프로젝트를 통해 무엇을 해보고 싶은지에 대해 설명했습니다.

이제부터는 조금 더 구체적으로, 애플리케이션 레벨 암호화를 실제 코드 구조로 어떻게 풀어낼 것인지에 대한 이야기로 넘어가려고 합니다.

애플리케이션 레벨 암호화를 구현하는 방법은 생각보다 다양합니다.
서비스 계층에서 직접 암·복호화를 수행할 수도 있고,
엔티티 내부에서 처리하는 방식도 떠올릴 수 있습니다.
하지만, 저는 암·복호화 로직이 비즈니스 코드 곳곳에 분산되지 않으면서 개발자가 의식적으로 통제할 수 있는 구조를 만들고 싶었습니다. 그래서 JPA가 제공하는 AttributeConverter를 사용해서 구현해보기로 했습니다.

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

## 관련 포스팅

* [[토이 프로젝트] JPA 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)