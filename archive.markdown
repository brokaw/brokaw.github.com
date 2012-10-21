---
title: Recent
layout: default
---

{% for post in site.posts limit:5 %}
<h2>{{ post.title }}</h2>
{{ post.date | date_to_long_string }} -- {% if post.revised != nil %}Updated on {{ post.update | date_to_long_string }}.{% endif %}
{{ post.description }}

<a href="{{ post.url }}">Full post.</a>
{% endfor %}