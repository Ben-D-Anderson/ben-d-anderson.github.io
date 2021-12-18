---
layout: page
title: Projects
permalink: /projects
---

<ul>
    {% for post in site.categories.project %}
        <h1 class="post-title">
        <a href="{{ post.url | relative_url }}">
            {{ post.title }}
        </a>
        </h1>

        <time datetime="{{ post.date | date_to_xmlschema }}" class="post-date"
        >{{ post.date | date_to_string }}</time
        >
        <br/>
        <br/>
        <br/>
    {% endfor %}
</ul>