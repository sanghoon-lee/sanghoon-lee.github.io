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
  margin: 2rem 0 1.5rem;
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
  text-align: left;
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
  <p class="hero-subtitle">
    기술부채를 줄이고, 팀의 행복과 서비스 품질향상을 함께 고민하는 평범한 개발팀장입니다.
  </p>
  <ul>
    <li>🔹 닉네임 : 준민아빠 </li>
    <li>🔹 한줄소개 : 이제는 개발자가 아닌 개발팀장이 직업이 되어버린 것 같습니다. </li>
    <li>🔹 관심분야 : 기술부채 청산, 조직관리</li>
  </ul>
</section>

