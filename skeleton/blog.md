---
layout: default
title: Latest posts
permalink: /blog
---

# Latest posts

[*See all posts*]({{ "/blog/list" | prepend: site.baseurl | prepend: site.url}})

{% for post in site.posts limit:10 %}
[{{ post.date | date: "%F" }} : {{ post.title | xml_escape }} \| {{ post.description | xml_escape }}]({{ post.url | prepend: site.baseurl | prepend: site.url }})
{% endfor %}

---

# {{ site.posts[0].title }}

{{ site.posts[0].content }}
