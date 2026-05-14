---
layout: page
title: Tags
permalink: /tags/
---

{% for tag in site.tags %}
  <h2 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h2>
  <ul>
    {% assign sorted_posts = tag[1] | sort: 'date' | reverse %}
    {% for post in sorted_posts %}
      <li>
        <span style="color: #888; font-size: 0.85em;">{{ post.date | date: "%Y-%m-%d" }}</span>
        &nbsp;<a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
  </ul>
{% endfor %}
