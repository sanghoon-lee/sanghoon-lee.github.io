---
layout: post
title: "[토이 프로젝트] JPA 컬럼 암호화: (3) 프로젝트 구조 살펴보기"
date: 2026-01-26
categories: 토이프로젝트
---

지난 포스팅에서 애플리케이션 레벨 암호화를 어떤 방식으로 구현할지 고민했던 과정을 정리해봤습니다.

가장 중요하게 생각했던 점은 암·복호화 로직이 비즈니스 흐름이나 도메인 모델에 섞이지 않도록 명확하게 분리하는 것이었습니다. 그래서 암호화와 관련된 코드는 별도의 패키지로 모으고, 서비스 계층이나 엔티티 내부에서는 이를 전혀 의식하지 않아도 되도록 구조를 구성했습니다.

이제부터 프로젝트 구조를 살펴보겠습니다.

## 프로젝트 구조

```
├─ api
│  ├─ controller
│  └─ service
│
├─ domain
│  ├─ entity
│  └─ repository
│
├─ crypto
│  ├─ CryptoEngine
│  └─ converter
│
├─ config
```

각 패키지의 역할은 다음과 같습니다.

| 패키지 |	역할 |
| ---- | ---- |
| api |	컨트롤러 및 서비스 계층으로 비즈니스 흐름을 처리합니다. |
| domain | 리포지토리 및 엔티티 계층으로 실제 도메인의 데이터 모델과 DB테이블을 매핑시키고, CRUD 처리를 담당합니다. |
| crypto | 애플리케이션 레벨에서의 데이터 암·복호화를 전담합니다. |
| config | 애플리케이션 구동에 필요한 설정정보를 관리합니다. |

## api 패키지

토이 프로젝트에서 구현한 애플리케이션 레벨 암호화가 정상적으로 동작하는지 확인해보기 위한 목적으로 다음과 같이 간단한 2개의 앤드포인트 API를 정의했습니다. 

| Method | URI | 설명 |
| ---- | ---- | ---- |
| POST | `/api/v1/account` | 계정 정보를 저장합니다. |
| GET | `/api/v1/account` | 계정 정보를 조회합니다. |

API의 요청과 응답 시 모든 데이터는 평문으로 처리됩니다.데이터의 암·복호화는 내부에서 자동으로 처리되기 때문에, API를 사용하는 입장에서는 이를 의식할 필요가 없습니다.

## 참고 : 소스 코드 
* [JPA Crypto Conveter](https://github.com/sanghoon-lee/jpa-crypto-converter)

## 관련 포스팅

* [[토이 프로젝트] JPA 컬럼 암호화: (2) AttributeConverter를 선택한 이유](https://sanghoon-lee.github.io/2026/01/25/JPACryptoConverter2/)
* [[토이 프로젝트] JPA 컬럼 암호화: (1) 애플리케이션 레벨 암호화에 대한 고민](https://sanghoon-lee.github.io/2026/01/24/JPACryptoConverter/)
* [[트러블슈팅] 스프링에서 JPA Auditing이 동작하지 않는 문제 - @EnableJpaAuditing 빠진 경우](https://sanghoon-lee.github.io/2026/01/21/JPAAuditing/)
* [[학습기록] 스프링 @MappedSuperClass 어노테이션](https://sanghoon-lee.github.io/2026/01/07/MappedSuperClass/)