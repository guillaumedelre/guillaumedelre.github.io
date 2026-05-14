---
layout: page
title: Categories
permalink: /categories/
description: "All articles grouped by category: development, devops, and IoT."
---

{% for category in site.categories %}
  <h2 id="{{ category[0] | slugify }}">{{ category[0] }}</h2>
  <ul>
    {% assign sorted_posts = category[1] | sort: 'date' | reverse %}
    {% for post in sorted_posts %}
      <li>
        <span style="color: #888; font-size: 0.85em;">{{ post.date | date: "%Y-%m-%d" }}</span>
        &nbsp;<a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
  </ul>
{% endfor %}
