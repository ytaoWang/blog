---
layout: default
title: archive
---

## Blog Post

{% for post in site.posts  %}
 * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endfor %}