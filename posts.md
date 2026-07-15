---
layout: page
title: Posts
permalink: /posts/
---

<style>
  /* JS 준비 전에는 목록과 페이지네이션 숨김 */
  .post-list,
  .pagination {
    display: none;
  }

  /* JS가 준비되면 목록 표시 */
  .js-ready .post-list {
    display: block;
  }

  .posts-archive-title {
    margin-bottom: 0.4rem;
  }
</style>

<h2 class="posts-archive-title">포스트 목록</h2>
<ul id="post-list" class="post-list">
  {% for post in site.posts %}
    <li class="post-item">
      <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>

      {% if post.tags %}
        <div class="post-tags">
          {% for tag in post.tags %}
            <span class="post-tag">#{{ tag }}</span>
          {% endfor %}
        </div>
      {% endif %}
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
      let html = '';

      // 현재 페이지 앞뒤로 보여줄 페이지 수
      const visibleRange = 1;

      // Prev
      html += `
        <button
          type="button"
          class="page-btn prev"
          aria-label="이전 페이지"
          ${currentPage === 1 ? 'disabled' : ''}>
          이전
        </button>
      `;

      /**
       * 페이지 번호 버튼 생성
       */
      function createPageButton(page) {
        return `
          <button
            type="button"
            class="page-btn num ${page === currentPage ? 'active' : ''}"
            data-page="${page}"
            ${page === currentPage ? 'aria-current="page"' : ''}>
            ${page}
          </button>
        `;
      }

      /**
       * 말줄임표 생성
       */
      function createEllipsis() {
        return `<span class="page-ellipsis" aria-hidden="true">…</span>`;
      }

      // 페이지 수가 적으면 전체 출력
      if (totalPages <= 7) {
        for (let page = 1; page <= totalPages; page++) {
          html += createPageButton(page);
        }
      } else {
        const startPage = Math.max(2, currentPage - visibleRange);
        const endPage = Math.min(totalPages - 1, currentPage + visibleRange);

        // 첫 페이지는 항상 표시
        html += createPageButton(1);

        // 첫 페이지와 현재 페이지 영역 사이 생략
        if (startPage > 2) {
          html += createEllipsis();
        }

        // 현재 페이지 주변 번호
        for (let page = startPage; page <= endPage; page++) {
          html += createPageButton(page);
        }

        // 현재 페이지 영역과 마지막 페이지 사이 생략
        if (endPage < totalPages - 1) {
          html += createEllipsis();
        }

        // 마지막 페이지는 항상 표시
        html += createPageButton(totalPages);
      }

      // Next
      html += `
        <button
          type="button"
          class="page-btn next"
          aria-label="다음 페이지"
          ${currentPage === totalPages ? 'disabled' : ''}>
          다음
        </button>
      `;

      paginationEl.innerHTML = html;

      // 이전 버튼
      const prevBtn = paginationEl.querySelector('.prev');
      prevBtn?.addEventListener('click', () => {
        renderPage(currentPage - 1);
        scrollToPostList();
      });

      // 다음 버튼
      const nextBtn = paginationEl.querySelector('.next');
      nextBtn?.addEventListener('click', () => {
        renderPage(currentPage + 1);
        scrollToPostList();
      });

      // 페이지 번호 버튼
      paginationEl.querySelectorAll('[data-page]').forEach(btn => {
        btn.addEventListener('click', () => {
          renderPage(Number(btn.dataset.page));
          scrollToPostList();
        });
      });
    }

    function scrollToPostList() {
      listEl.scrollIntoView({
        behavior: 'smooth',
        block: 'start'
      });
    }

    // 4) JS 준비 완료 표시(숨김 해제) + 첫 페이지 렌더
    document.documentElement.classList.add('js-ready');
    renderPage(1);
  })();
</script>

<style>
  /* JS 준비가 완료된 후 페이지네이션 표시 */
  .js-ready .pagination {
    margin: 1.5rem 0 2rem;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .page-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 38px;
    height: 38px;
    border: 1px solid #e5e7eb;
    background: #fff;
    padding: 0.4rem 0.7rem;
    border-radius: 8px;
    cursor: pointer;
    color: #333;
  }

  .page-btn:hover:not(:disabled):not(.active) {
    background: #f3f4f6;
  }

  .page-btn.active {
    border-color: #111;
    background: #111;
    color: #fff;
    font-weight: 700;
  }

  .page-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .page-ellipsis {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 24px;
    height: 38px;
    color: #6b7280;
  }

  /* 모바일 */
  @media screen and (max-width: 600px) {
    .js-ready .pagination {
      gap: 0.35rem;
    }

    .page-btn {
      min-width: 34px;
      height: 34px;
      padding: 0.3rem 0.55rem;
      font-size: 0.9rem;
    }

    .page-ellipsis {
      min-width: 18px;
      height: 34px;
    }
  }
</style>
