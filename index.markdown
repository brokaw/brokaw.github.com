---
title: Home
layout: default
description: The front page.
---

{% assign recent_post = site.posts.first %}
<h2>{{ recent_post.title }} </h2>
{{ recent_post.content }}
<p><a href="{{recent_post.url}}">permalink</a></p>
<div class="row">
<div class="span8">
<hr />
<h2>Recent Posts</h2>
</div>
</div>
<div class="row">
{% for post in site.posts limit:2 offset:1 %}
<div class="span4"><h3>{{ post.title }}</h3><p>{{ post.description }}</p>
<p><a href="{{ post.url }}"><small>More…</small></a></p></div>
{% endfor %}
</div>
<div class="row">
{% for post in site.posts limit:2 offset:3 %}
<div class="span4"><h3>{{ post.title }}</h3><p>{{ post.description }}</p>
<p><a href="{{ post.url }}"><small>More…</small></a></p></div>
{% endfor %}
</div>



