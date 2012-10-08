---
title: Recent
layout: default
---


{% for post in site.posts limit:5 %}
<h2>{{ post.title }}</h2>
{{ post.description }}
<a href="{{ post.url }}">Full post.</a>
{% endfor %}