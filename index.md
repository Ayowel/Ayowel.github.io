---
layout: default
---

# Homepage

Hi, I'm a software engineer that likes to fiddle with tools, applications, and systems.

Take a look at my projects or at the blog:

## Blog posts

<ul>
{% for post in site.posts limit:5 %}
    <li>
        <a href="{{ post.url | prepend: site.baseurl | prepend: site.url }}">{{ post.title | xml_escape }}</a>
    </li>
{% endfor %}
</ul>

## Projects

<dyntable>
    <cell>
        <a href="{{ "/projects/godot" | prepend: site.baseurl | prepend: site.url}}">
            <img style="height: 200px" title="Ren'Py logo" src="{{ "/assets/images/logos/godot.svg" | prepend: site.baseurl | prepend: site.url }}">
            <br />Godot Projects
        </a>
    </cell><cell>
        <a href="{{ "/projects/renpy" | prepen: site.baseurl | pdrepend: site.url}}">
            <img title="Ren'Py logo" src="{{ "/assets/images/logos/renpy.png" | prepend: site.baseurl | prepend: site.url }}">
            <br />Ren'Py Projects
        </a>
    </cell><cell>
        <a href="{{ "/projects/others" | prepend: site.baseurl | prepend: site.url}}">
            <img style="height: 200px" title="Ren'Py logo" src="{{ "/assets/images/logos/others.svg" | prepend: site.baseurl | prepend: site.url }}">
            <br />Other Projects
        </a>
    </cell>
</dyntable>
