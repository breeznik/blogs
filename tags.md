---
layout: page
title: Tags
permalink: /tags/
---

{% assign sorted_tags = site.tags | sort %}
{% for tag in sorted_tags %}
  {% assign tag_name = tag | first %}
  {% assign tag_posts = tag | last %}
  <div class="tag-section" id="{{ tag_name | slugify }}">
    <h2 class="tag-heading">{{ tag_name }} <span class="tag-count">{{ tag_posts | size }}</span></h2>
    <ul class="tag-post-list">
      {% for post in tag_posts %}
      <li>
        <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        <span class="tag-post-date">{{ post.date | date: "%b %Y" }}</span>
      </li>
      {% endfor %}
    </ul>
  </div>
{% endfor %}
