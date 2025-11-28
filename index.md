---
layout: home
title: 🏠 Home
---

<style>
/* Hero 영역 스타일 개선 */
.hero {
  margin: 2.8rem 0 2rem;
  padding: 2rem 2rem;
  border-radius: 14px;
  background: #fafafa;
  border: 1px solid #eee;
}

/* 제목 */
.hero-title {
  margin: 0 0 1rem;
  font-size: 2rem;
  font-weight: 800;
  line-height: 1.45;
  color: #111;
}

/* 부제 */
.hero-subtitle {
  margin: 0 0 1.6rem;
  color: #4e4e4e;
  font-size: 0.95rem;
  line-height: 1.7;
}

/* 태그 라벨 */
.hero-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin: 0;
  padding: 0;
  list-style: none;
}

.hero-tag {
  padding: 0.35rem 0.85rem;
  border-radius: 20px;
  background: #fff;
  border: 1px solid #ddd;
  font-size: 0.82rem;
  color: #555;
}

/* 반응형 개선 */
@media (max-width: 768px) {
  .hero {
    padding: 1.5rem 1.25rem;
  }
  .hero-title {
    font-size: 1.55rem;
  }
}
</style>

<section class="hero">
  <h1 class="hero-title">
    팀의 행복을 버그 없이 배포합니다.
  </h1>
  <p class="hero-subtitle">
    기술부채는 줄이고, 개발자가 일하기 좋은 문화를 만들고 싶습니다.<br>
    하지만, 일정을 버그처럼 고치고 있네요.
  </p>
  <ul class="hero-tags">
    <li class="hero-tag">🧑‍💼 Engineering Leadership</li>
    <li class="hero-tag">🔧 Tech Debt Recovery</li>
    <li class="hero-tag">🤝 Culture Crafting</li>
  </ul>
</section>

<section>
  <h2 class="home-posts-header">최근 포스트</h2>
  <p class="home-posts-intro">
    기술부채 청산, 서비스 안정화, 개발문화 개선과 관련된 기록들을 모아두었습니다.
  </p>
</section>
