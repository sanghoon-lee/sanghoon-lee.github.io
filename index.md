---
layout: home
title: 🏠 Home
---

<style>
/* Hero wrapper */
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 2rem 0 2.5rem;
  padding: 1.5rem 1rem;
}

/* Hero image: 자연스러운 크기 */
.hero-image {
  width: 80%;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}

/* Title (텍스트 중심 강조) */
.hero-title {
  margin: 0 0 0.6rem;
  font-size: 1.65rem;
  font-weight: 800;
  line-height: 1.45;
  text-align: center;
}

/* Subtitle alignment balancing */
.hero-subtitle {
  margin: 0 0 1.2rem;
  font-size: 0.92rem;
  line-height: 1.6;
  text-align: center;
  color: #4e4e4e;
  max-width: 600px;
}

/* Tags: 중앙 정렬 */
.hero-tags {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 0.45rem;
  margin-top: 0.8rem;
}

/* Responsive */
@media (max-width: 768px) {
  .hero-image {
    max-width: 280px;
  }
  .hero-title {
    font-size: 1.4rem;
  }
}
</style>

<section class="hero">
  <img class="hero-image" src="/assets/hero.png" alt="Hero image: 팀의 행복을 버그 없이 배포합니다">
  <h1 class="hero-title">
    팀의 행복을 버그 없이 배포합니다.
  </h1>
  <p class="hero-subtitle">
    기술부채는 줄이고, 개발자가 일하기 좋은 문화를 만들고 싶습니다.<br>
    하지만, 일정을 버그처럼 고치고 있네요.
  </p>
</section>
