---
layout: default
title: 首页
---

<section class="hero">
  <div>
    <h1>慢慢浸佢，慢慢叹佢，有得食</h1>
  </div>
</section>

<section>
  <div class="section-heading">
    <h2>最新文章</h2>
    <a href="{{ '/archive/' | relative_url }}">查看全部</a>
  </div>

  <ul class="post-list">
    {% for post in site.posts limit: 5 %}
      <li class="post-card">
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
        <div>
          <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
          {% if post.description %}
            <p>{{ post.description }}</p>
          {% else %}
            <p>{{ post.excerpt | strip_html | truncate: 120 }}</p>
          {% endif %}
        </div>
      </li>
    {% endfor %}
  </ul>
</section>
