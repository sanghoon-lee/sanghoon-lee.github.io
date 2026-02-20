---
layout: home
title: 🏠 Home
---

<style>
/* =========================
   Home Intro Layout
   ========================= */

.home-intro {
  max-width: 860px;
  margin: 2.5rem auto 1rem;
  padding: 1.5rem 1rem;
  display: grid;
  grid-template-columns: 180px 1fr;
  gap: 1.75rem;
  align-items: center;
}

/* =========================
   Profile Image
   ========================= */

.profile-wrap {
  display: flex;
  justify-content: center;
  align-items: center;
}

.profile-img {
  width: 180px;
  height: 180px;
  border-radius: 999px;          /* 완전 원형 */
  object-fit: cover;
  display: block;
  border: 1px solid rgba(0,0,0,0.08);
  box-shadow: 0 6px 16px rgba(0,0,0,0.12); /* 은은한 그림자 */
}

/* =========================
   Text Area
   ========================= */

.profile-text h1 {
  margin: 0 0 0.4rem;
  font-size: 1.6rem;
  font-weight: 800;
  line-height: 1.35;
}

.profile-text p {
  margin: 0.45rem 0;
  line-height: 1.7;
  color: #374151;
}

.profile-meta {
  margin: 0.9rem 0 0;
  padding-left: 1.1rem;
  color: #4b5563;
}

.profile-meta li {
  margin: 0.3rem 0;
}

/* =========================
   Badges
   ========================= */

.badges {
  margin-top: 0.9rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.badge {
  display: inline-block;
  font-size: 0.85rem;
  padding: 0.3rem 0.6rem;
  border-radius: 999px;
  border: 1px solid rgba(0,0,0,0.12);
  background: rgba(0,0,0,0.03);
  color: #111827;
}

/* =========================
   Responsive
   ========================= */

@media (max-width: 768px) {
  .home-intro {
    grid-template-columns: 1fr;
    gap: 1.25rem;
  }

  .profile-wrap {
    justify-content: flex-start;
  }

  .profile-img {
    width: 140px;
    height: 140px;
  }
}
</style>

<section class="home-intro">
  <div class="profile-wrap">
    <!-- 프로필 이미지 파일을 /assets/profile.jpg 로 두는 걸 추천 -->
    <img class="profile-img" src="/assets/profile.jpg" alt="준민아빠">
  </div>

  <ul class="hero-subtitle">
    <li>닉네임 : 준민아빠 </li>
    <li>한줄소개 : 이제는 개발실무보다 조직관리가 더 익숙해진 것 같습니다. 하지만, 개발실무는 그만큼 더 멀어졌습니다. </li>
    <li>관심분야 : 기술부채 청산, 조직관리</li>
    <li>블로그 개설일 : 2025.11.28</li>
  </ul>
</section>

