---
layout: default
title: David Lattimore
---

## Posts

<ul class="posts">
  {% for post in site.categories.posts %}
    <li class="post">
      <a href="{{ post.url }}">{{ post.title }}</a>
      <time class="publish-date" datetime="{{ post.date | date: '%F' }}">
        {{ post.date | date: "%Y-%m-%d" }}
      </time>
    </li>
  {% endfor %}
</ul>
