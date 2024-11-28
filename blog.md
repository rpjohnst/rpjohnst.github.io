---
title: Blog
layout: default
---

<header>
<h1>Posts</h1>
</header>

{% for post in site.posts %}
[{{ post.title }}]({{ post.url }})
{% endfor %}
