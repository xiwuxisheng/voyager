---
bg: "tag.jpg"
layout: page
permalink: /posts/
title: "索引"
crawlertitle: "所有文章"
summary: "遥知不是雪，为有暗香来"
active: 迁客骚人
---

{% for tag in site.tags %}
  {% assign t = tag | first %}
  {% assign posts = tag | last %}

  <h2 class="category-key" id="{{ t | downcase }}">{{ t | capitalize }}</h2>

  <ul class="year">
    {% for post in posts %}
      {% if post.tags contains t %}
        <li>
          {% if post.lastmod %}
            <a href="{{ post.url | relative_url}}">{{ post.title }}</a>
            <span class="date">{{ post.lastmod | date: "%d-%m-%Y"  }}</span>
          {% else %}
            <a href="{{ post.url | relative_url}}">{{ post.title }}</a>
            <span class="date">{{ post.date | date: "%d-%m-%Y"  }}</span>
          {% endif %}
        </li>
      {% endif %}
    {% endfor %}
  </ul>

{% endfor %}
