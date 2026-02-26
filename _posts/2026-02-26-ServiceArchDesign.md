---
layout: post
title: "[학습기록] 소프트웨어 아키텍처: (3) 서비스 기반 아키텍처 스타일"
date: 2026-02-26
categories: 학습기록
---

<style>
.main-image {
  width: 200px;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

<img class="main-image" src="/assets/images/101.jpg" alt="소프트웨어 아키텍처 101">

현재 우리 팀이 운영하고 있는 서비스는 당연히 `마이크로서비스 아키텍처`(MSA) 스타일 기반으로 설계된 것이라고 알고 있었습니다.

독립적으로 실행되는 여러 도메인 서비스로 구성되어 있는 것처럼 보였기 때문입니다. 그리고 과거에 운영 업무를 담당했던 개발자들도 그렇게 이야기했다고 들었습니다.

저와 팀원들이 아키텍처에 대해 깊이 있는 지식을 가지고 있는 것은 아니었기 때문에, 별다른 의심 없이 그대로 받아들였던 것 같습니다.

물론, `마이크로서비스 아키텍처`라고 하기에는 어딘가 이상해 보이는 지점들도 분명히 있었습니다.

* 도메인 서비스들이 DB를 공유하고 있었고,
* 서비스 간 결합도가 생각보다 높았으며,
* 한 서비스의 변경이 다른 서비스에 영향을 주는 구조였습니다.

하지만, 
> “서비스를 운영해오면서 구조가 망가진 게 아닐까?” 

정도로만 가볍게 생각을 했었습니다.

그런데 이번에 **소프트웨어 아키텍처 101**을 읽으며 아키텍처 스타일에 대해 공부하는 과정에서 우리팀의 서비스가 구조가 망가진 `마이크로서비스 아키텍처`가 아니라는 사실을 알 수 있었습니다. 사실 처음부터 다른 아키텍처로 설계되었던 것입니다.

<style>
.arch-image {
  width: 100%;
  max-width: 900px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

<img class="arch-image" src="/assets/images/myservice.jpg" alt="서비스 토폴로지">
