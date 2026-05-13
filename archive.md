---
title: 文章
permalink: /archive/
description: 所有博客文章会按发布时间汇总在这里。
---

<ul class="post-list">
  {% for post in site.posts %}
    <li class="post-card">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      <div>
        <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
        {% if post.description %}
          <p>{{ post.description }}</p>
        {% endif %}
      </div>
    </li>
  {% endfor %}
</ul>
