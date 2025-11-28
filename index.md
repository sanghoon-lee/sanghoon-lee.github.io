---
layout: home
title: 🏠 Home
---

<style>
/* Hero 영역 */
.hero {
  margin: 2rem 0 1.5rem;
}

.hero-title {
  margin: 0 0 0.75rem;
  font-size: 1.9rem;
  font-weight: 700;
}

.hero-subtitle {
  margin: 0 0 1.25rem;
  color: #555;
  font-size: 0.96rem;
  line-height: 1.6;
}

/* 태그 라인 */
.hero-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin: 0;
  padding: 0;
  list-style: none;
}

.hero-tag {
  padding: 0.25rem 0.7rem;
  border-radius: 999px;
  border: 1px solid #ddd;
  font-size: 0.8rem;
  color: #666;
}

/* 포스트 리스트 섹션 안내 문구 */
.home-posts-header {
  margin: 2.5rem 0 0.5rem;
  font-size: 1.1rem;
  font-weight: 600;
}

.home-posts-intro {
  margin: 0 0 1.5rem;
  color: #666;
  font-size: 0.9rem;
}

/* 반응형 */
@media (max-width: 768px) {
  .hero-title {
    font-size: 1.6rem;
  }
}
</style>

<section class="hero">
  <h1 class="hero-title">기술부채는 청산하고, 좋은 개발문화는 남기는 개발팀장</h1>
  <p class="hero-subtitle">
    눈앞의 문제 해결하는 팀이 아니라, <br>
    지속 가능한 성장을 이끄는 팀을 지향합니다.
  </p>
  <ul class="hero-tags">
    <li class="hero-tag">🔨 Tech Debt Reduction</li>
    <li class="hero-tag">🤝 Development Culture</li>
    <li class="hero-tag">📊 Data-based Quality</li>
    <li class="hero-tag">🧑‍💼 Engineering Leadership</li>
  </ul>
</section>

<section>
  <h2 class="home-posts-header">최근 포스트</h2>
  <p class="home-posts-intro">
    기술부채 청산, 서비스 안정화, 개발문화 개선과 관련된 기록들을 모아두었습니다.
  </p>
</section>
