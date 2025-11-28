---
layout: default
title: Home
---

<style>
/* 페이지 전체 레이아웃 */
.page-layout {
  display: flex;
  max-width: 1000px;
  margin: 0 auto;
  gap: 24px;
}

/* 왼쪽 사이드바 */
.sidebar {
  width: 260px;
  padding: 16px 8px 16px 0;
  border-right: 1px solid #eee;
  font-size: 0.95rem;
}

/* 프로필 사진 */
.profile-img {
  width: 120px;
  height: 120px;
  object-fit: cover;
  border-radius: 50%;
  display: block;
  margin-bottom: 12px;
}

/* 오른쪽 메인 영역 */
.main {
  flex: 1;
  padding: 16px 0 16px 8px;
}

/* 포스트 목록 스타일 */
.post-list {
  list-style: none;
  padding-left: 0;
  margin: 0;
}

.post-list li {
  margin-bottom: 10px;
}

.post-list a {
  text-decoration: none;
}

.post-list a:hover {
  text-decoration: underline;
}

/* 날짜 스타일 */
.post-date {
  font-size: 0.8rem;
  color: #888;
}

/* SNS/링크 목록 */
.profile-links {
  list-style: none;
  padding-left: 0;
  margin-top: 12px;
}

.profile-links li {
  margin-bottom: 4px;
}
</style>

<div class="page-layout">
  <!-- 왼쪽 사이드바: 프로필 영역 -->
  <aside class="sidebar">
    <img class="profile-img" src="/assets/profile.jpg" alt="프로필 사진">

    <h2>이상훈</h2>
    <p>
      스팸전화 차단, 보안, 데이터 기반 서비스에 관심이 많은 개발자입니다.<br>
      이 블로그는 업무/사이드 프로젝트를 기록하는 공간입니다.
    </p>

    <ul class="profile-links">
      <li><a href="https://github.com/여기에_본인_아이디">GitHub</a></li>
      <li><a href="mailto:이메일주소@도메인">Email</a></li>
      <!-- 필요하면 더 추가 -->
      <!-- <li><a href="https://linked.in/…">LinkedIn</a></li> -->
    </ul>
  </aside>

  <!-- 오른쪽 메인: 포스트 목록 -->
  <main class="main">
    <h1>Posts</h1>
    <ul class="post-list">
      {% for post in site.posts %}
        <li>
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a><br>
          <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
        </li>
      {% endfor %}
    </ul>
  </main>
</div>
