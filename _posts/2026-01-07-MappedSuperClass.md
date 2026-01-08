---
layout: post
title: "[학습기록] 스프링부트 @MappedSuperClass 사용법"
date: 2026-01-07
categories: java
---

<style>
.main-image {
  width: 50%;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

스프링부트에서 다양한 어노테이션을 제공하면서 개발 편의성을 보장하고 있습니다. 하지만, 어노테이션의 종류가 너무 많다보니 정확한 동작 방식이나 사용 방법을 이해하지 못한 채 “이렇게 쓰는 것 같다”는 경험에 의존해서 습관적으로 사용하는 경우도 적지 않았습니다.

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

@MappedSuperclass 역시 그 중 하나였습니다. 자주 사용하면서도 동작 방식이나 사용 방법을 명확하게 설명하기 어려웠습니다. 제 스스로도 정리가 되어 있지 않다는 느낌을 받았습니다. 부끄럽지만, 제 실력이 부족한 탓이기도 했습니다. 하지만, 최근에 혼자 토이 프로젝트를 진행하면서 @MappedSuperClass에 대해서 공부할 기회를 가질 수 있었습니다. 그 때, 공부했던 내용을 정리해서 포스팅했습니다.

### @MappedSuperclass

