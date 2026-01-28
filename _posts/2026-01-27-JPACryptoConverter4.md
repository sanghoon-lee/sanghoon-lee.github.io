---
layout: post
title: "[토이 프로젝트] DB 컬럼 암호화: (4) 코드 살펴보기"
date: 2026-01-27
categories: 토이프로젝트
---

지난 포스팅들을 통해서 애플리케이션 레벨 암호화의 구현을 검토하게 된 계기와 구현 방식으로 JPA의 `AttributeConverter`를 선택하게 된 이유, 그리고 전체 프로젝트 구조까지 정리해봤습니다.

이제 어떤 코드 흐름을 통해 애플리케이션 레벨 암호화가 동작하는지를 핵심 코드 중심으로 이야기해볼 차례입니다. 복잡한 세부 구현사항을 일일이 설명하기 보다는 `AttributeConverter`를 중심으로 각 구성 요소가 어떤 역할을 맡고, 어떻게 연결되어 있는지에 초점을 맞춰 설명하겠습니다.

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
