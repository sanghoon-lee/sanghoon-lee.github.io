---
layout: page
title: 📝 Posts
permalink: /posts/
---

<section class="posts-archive">
  <h1 class="posts-archive-title">포스트 목록</h1>
  
  <ul class="post-list">
    {% for post in site.posts %}
      <li>
        <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a>
        </h3>
      </li>
    {% endfor %}
  </ul>
</section>
