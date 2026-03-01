---
layout: page
title: Search
permalink: /search/
---

<div class="search-wrap">
  <input type="text" id="search-input" class="search-input" placeholder="Search posts..." autofocus />
  <div id="search-results" class="search-results"></div>
</div>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
  const searchIndex = fetch('{{ "/assets/search.json" | prepend: site.baseurl }}')
    .then(r => r.json())
    .then(posts => {
      const idx = lunr(function () {
        this.ref('url');
        this.field('title', { boost: 10 });
        this.field('tags',  { boost: 5  });
        this.field('excerpt');
        posts.forEach(p => this.add(p));
      });

      const data = {};
      posts.forEach(p => data[p.url] = p);

      const input   = document.getElementById('search-input');
      const results = document.getElementById('search-results');

      function doSearch(query) {
        results.innerHTML = '';
        if (!query.trim()) return;
        const hits = idx.search(query + '*');
        if (!hits.length) {
          results.innerHTML = '<p class="search-empty">No results for "' + query + '"</p>';
          return;
        }
        hits.forEach(h => {
          const p = data[h.ref];
          const el = document.createElement('div');
          el.className = 'search-result';
          el.innerHTML = `
            <a class="search-result-title" href="${p.url}">${p.title}</a>
            <p class="search-result-meta">${p.date}${p.tags ? ' · ' + p.tags : ''}</p>
            <p class="search-result-excerpt">${p.excerpt}</p>
          `;
          results.appendChild(el);
        });
      }

      input.addEventListener('input', e => doSearch(e.target.value));
      const params = new URLSearchParams(window.location.search);
      if (params.get('q')) { input.value = params.get('q'); doSearch(params.get('q')); }
    });
</script>
