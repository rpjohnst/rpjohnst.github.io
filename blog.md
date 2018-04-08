---
title: Blog
layout: default
---

## Posts

{% for post in site.posts %}
[{{ post.title }}]({{ post.url }})
{% endfor %}
