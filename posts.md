---
layout: page
title: 📝 Posts
permalink: /posts/
---

<style>
.posts-archive-title {
  margin-bottom: 0.4rem;
}

.posts-archive-intro {
  margin-top: 0;
  margin-bottom: 1.4rem;
  color: #666;
  font-size: 0.95rem;
}

/* 페이지네이션 스타일 */
.pagination {
  margin-top: 1.8rem;
  text-align: center;
}

.pagination-inner {
  display: inline-flex;
  gap: 0.4rem;
  align-items: center;
}

.pagination button {
  border: 1px solid #ddd;
  background: #fff;
  padding: 0.3rem 0.7rem;
  border-radius: 6px;
  font-size: 0.85rem;
  cursor: pointer;
}

.pagination button:hover:not(:disabled) {
  border-color: #bbb;
}

.pagination button.active {
  background: #333;
  color: #fff;
  border-color: #333;
}

.pagination button:disabled {
  opacity: 0.4;
  cursor: default;
}
</style>

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

   <!-- 페이지네이션 UI가 들어갈 영역 -->
  <div id="pagination" class="pagination"></div>

  <script>
    document.addEventListener('DOMContentLoaded', function () {
        const postsPerPage = 5;  // ▶ 한 페이지에 보여줄 포스트 수
        const items = Array.from(document.querySelectorAll('#post-list .post-item'));
        const totalItems = items.length;
        const totalPages = Math.ceil(totalItems / postsPerPage);
        const paginationContainer = document.getElementById('pagination');
        let currentPage = 1;

        if (totalPages <= 1) {
        // 페이지가 1개뿐이면 굳이 페이지네이션 안 보이게
        return;
        }

        function renderPage(page) {
        currentPage = page;

        // 포스트 show/hide
        items.forEach((item, index) => {
            const start = (page - 1) * postsPerPage;
            const end = start + postsPerPage;
            if (index >= start && index < end) {
            item.style.display = '';
            } else {
            item.style.display = 'none';
            }
        });

        // 페이지네이션 버튼 다시 그리기
        renderPagination();
        }

        function renderPagination() {
        paginationContainer.innerHTML = '';

        const nav = document.createElement('div');
        nav.className = 'pagination-inner';

        // 이전 버튼
        const prev = document.createElement('button');
        prev.textContent = '이전';
        prev.disabled = currentPage === 1;
        prev.addEventListener('click', () => renderPage(currentPage - 1));
        nav.appendChild(prev);

        // 페이지 번호 버튼
        for (let i = 1; i <= totalPages; i++) {
            const btn = document.createElement('button');
            btn.textContent = i;
            if (i === currentPage) {
            btn.className = 'active';
            }
            btn.addEventListener('click', () => renderPage(i));
            nav.appendChild(btn);
        }

        // 다음 버튼
        const next = document.createElement('button');
        next.textContent = '다음';
        next.disabled = currentPage === totalPages;
        next.addEventListener('click', () => renderPage(currentPage + 1));
        nav.appendChild(next);

        paginationContainer.appendChild(nav);
        }

        // 처음 1페이지 렌더링
        renderPage(1);
    });
</script>

</section>
