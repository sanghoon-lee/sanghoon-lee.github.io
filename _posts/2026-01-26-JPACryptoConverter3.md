---
layout: post
title: "[토이 프로젝트] JPA 컬럼 암호화: (3) AttributeConverter 기반 구현 구조 살펴보기"
date: 2026-01-26
categories: 토이프로젝트
---

이전 포스팅에서 애플리케이션 레벨 암호화의 구현 방식을 고민했던 과정을 정리해봤습니다. 이제부터는 그 고민의 결과로 선택한 `AttributeConverter` 인터페이스를 통해, 실제로 애플리케이션 레벨 암호화가 어떻게 구현되었는지 코드 중심으로 살펴보도록 하겠습니다.

## 프로젝트 구조

애플리케이션 레벨 암호화를 구현하면서 가장 중요하게 생각했던 점은,
암·복호화 로직이 비즈니스 코드나 도메인 모델에 섞이지 않도록
명확하게 분리하는 것이었습니다. 

이를 위해 암호화와 관련된 코드는 별도의 패키지로 모으고, 엔티티와 서비스 계층에서는 이를 전혀 의식하지 않아도 되도록 구조를 구성했습니다.

이번 토이 프로젝트의 전체 구조는 다음과 같습니다.

```
├─ api
│  ├─ controller
│  └─ service
│
├─ crypto
│  ├─ CryptoEngine
│  └─ converter
│
├─ domain
│  ├─ entity
│  └─ repository
│
├─ config
```

## 패키지별 역할

| 패키지 |	역할 |
| ---- | ---- |
| api |	컨트롤러 및 서비스 계층. 비즈니스 흐름을 담당하며 암·복호화 로직을 전혀 의식하지 않습니다. |
| domain | 엔티티와 리포지토리. 평문 도메인 모델만 유지하며, 암·복호화는 외부에서 처리됩니다. |
| crypto | 애플리케이션 레벨 암·복호화를 담당하는 핵심 영역. AttributeConverter와 암호화 엔진이 위치합니다. |
| config | 암호화 키, Converter, 기타 설정을 포함한 애플리케이션 설정 영역입니다. |


## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

## 관련 포스팅

* [[토이 프로젝트] JPA 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] JPA 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)