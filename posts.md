---
layout: page
title: Posts
permalink: /posts/
---

<style>
  /* JS 준비 전에는 목록/페이지네이션 숨김 (깜빡임 방지) */
  .post-list,
  .pagination {
    display: none;
  }

  /* JS가 준비되면 보여줌 */
  .js-ready .post-list,
  .js-ready .pagination {
    display: block;
  }
</style>

<ul id="post-list" class="post-list">
  {% for post in site.posts %}
    <li class="post-item">
      <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>
    </li>
  {% endfor %}
</ul>

<div id="pagination" class="pagination"></div>

<script>
  (function () {
    // 1) 대상 찾기
    const listEl = document.getElementById('post-list');
    const paginationEl = document.getElementById('pagination');
    if (!listEl || !paginationEl) return;

    const items = Array.from(listEl.querySelectorAll('.post-item'));
    if (items.length === 0) return;

    // 2) 페이지 설정
    const perPage = 5; // 한 페이지에 보여줄 글 수
    let currentPage = 1;
    const totalPages = Math.ceil(items.length / perPage);

    // 3) 렌더 함수
    function renderPage(page) {
      currentPage = Math.min(Math.max(page, 1), totalPages);

      const start = (currentPage - 1) * perPage;
      const end = start + perPage;

      items.forEach((item, idx) => {
        item.style.display = (idx >= start && idx < end) ? '' : 'none';
      });

      renderPagination();
    }

    function renderPagination() {
      // 버튼 HTML 생성
      let html = '';

      // Prev
      html += `<button type="button" class="page-btn prev" ${currentPage === 1 ? 'disabled' : ''}>이전</button>`;

      // Pages (간단 버전: 전체 페이지 버튼)
      for (let p = 1; p <= totalPages; p++) {
        html += `<button type="button" class="page-btn num ${p === currentPage ? 'active' : ''}" data-page="${p}">${p}</button>`;
      }

      // Next
      html += `<button type="button" class="page-btn next" ${currentPage === totalPages ? 'disabled' : ''}>다음</button>`;

      paginationEl.innerHTML = html;

      // 이벤트 바인딩
      const prevBtn = paginationEl.querySelector('.prev');
      const nextBtn = paginationEl.querySelector('.next');

      prevBtn && prevBtn.addEventListener('click', () => renderPage(currentPage - 1));
      nextBtn && nextBtn.addEventListener('click', () => renderPage(currentPage + 1));

      paginationEl.querySelectorAll('[data-page]').forEach(btn => {
        btn.addEventListener('click', () => renderPage(parseInt(btn.dataset.page, 10)));
      });
    }

    // 4) JS 준비 완료 표시(숨김 해제) + 첫 페이지 렌더
    document.documentElement.classList.add('js-ready');
    renderPage(1);
  })();
</script>

<style>
  /* 페이지네이션 기본 스타일(원하면 main.scss로 옮겨도 됨) */
  .pagination {
    margin: 1.5rem 0 2rem;
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .page-btn {
    border: 1px solid #e5e7eb;
    background: #fff;
    padding: 0.4rem 0.7rem;
    border-radius: 8px;
    cursor: pointer;
  }

  .page-btn.active {
    border-color: #111;
    font-weight: 700;
  }

  .page-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
