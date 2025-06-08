---
layout: page
title: Blog
permalink: /blog/
menu: true
cover: true
hide_description: true
---

<ul class="related-posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}" class="h4 flip-title">
        <span>{{ post.title }}</span>
      </a>
      <time class="heading faded fine" datetime="{{ post.date | date_to_xmlschema }}">
        {{ post.date | date: "%d %b %Y" }}
      </time>
      {% if post.excerpt %}
        <p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
