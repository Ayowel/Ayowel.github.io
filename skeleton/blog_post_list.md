---
layout: default
title: Latest posts
permalink: /blog/list
---

# All posts

{% for post in site.posts limit:5 %}
[{{ post.date | date: "%F" }} : {{ post.title | xml_escape }} \| {{ post.description | xml_escape }}]({{ post.url | prepend: site.baseurl | prepend: site.url }})
{% endfor %}
