---
layout: default
title: Home
---

# 준민아빠의 블로그

개발, 보안, 서비스 운영 관련해서 정리해 두는 개인 기록입니다.  
아래는 최신 포스팅 목록입니다.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <small> — {{ post.date | date: "%Y-%m-%d" }}</small>
    </li>
  {% endfor %}
</ul>
