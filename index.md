---
layout: home
title: desvert's blog
---

Welcome to my personal blog — a collection of hacking notes, walkthroughs, and technical explorations. This is where I document what I’m learning, breaking, and building, one step at a time.

<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>