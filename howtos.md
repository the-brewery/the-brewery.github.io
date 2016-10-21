---
layout: page
title:  How-tos
permalink: /howtos/
---

{% for post in site.posts %}
  {% if post.categories contains "howtos" %}
* [{{ post.title }}]({{ post.url }})
  {{ post.excerpt }}
  {% endif %}
{% endfor %}
