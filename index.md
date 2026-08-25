layout: default
title: Home
permalink: /
---

{% for post in site.posts %}
<article>
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <p>{{ post.date | date: "%Y-%m-%d" }}</p>
</article>
{% endfor %}
