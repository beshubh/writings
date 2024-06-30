---
layout: default
title: Home
---

# Hi, I am Shubh!

## Tech Essays
{% for post in site.tech %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}

## Life Essays
{% for post in site.life %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
