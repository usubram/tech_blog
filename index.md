---
layout: default
title: Home
---

# The Hands-On Architect

Welcome to my personal technical blog where I explore system design, architecture, and hands-on engineering.

## All Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span> - {{ post.date | date: "%B %d, %Y" }}</span>
    </li>
  {% endfor %}
</ul>
