---
layout: default
title: 首页
---

<section class="hero">
  <div>
    <h1>记录把想法做成产品的过程。</h1>
    <p>这里会写技术实现、产品判断、工程实践和一些阶段性复盘。先把博客搭起来，再让内容慢慢长出来。</p>
  </div>
  <aside class="hero-panel">
    <strong>近期关注</strong>
    <ul>
      <li>Jekyll 与 GitHub Pages</li>
      <li>AI 产品与工程实践</li>
      <li>长期项目的公开记录</li>
    </ul>
  </aside>
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
