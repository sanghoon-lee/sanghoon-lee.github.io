---
layout: post
title: "[토이 프로젝트] JPA 컬럼 암호화: (3) AttributeConverter 기반 구현 구조 살펴보기"
date: 2026-01-26
categories: 토이프로젝트
---

이전 포스팅에서는 애플리케이션 레벨 암호화를 구현하기에 앞서 가졌던 고민을 정리하면서 그 해법으로 JPA의 `AttributeConverter`를 선택하게 된 이유를 살펴봤습니다.

이번에는 `AttributeConverter`를 활용해 실제로 애플리케이션 레벨 암호화를 어떻게 구현했는지 코드 중심으로 살펴보겠습니다.

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

## 관련 포스팅

* [[토이 프로젝트] JPA 컬럼 암호화: (2) JPA AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] JPA 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)