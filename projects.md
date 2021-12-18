---
layout: page
title: Projects
permalink: /projects
---

Here you can find short write-ups of the projects on [my GitHub page](https://github.com/Ben-D-Anderson).
<div class="projects">
    <br/>
    {% for post in site.tags.project %}
        <h2 class="post-title">
            <a href="{{ post.url | relative_url }}">
                {{ post.title }}
            </a>
        </h2>

        <time datetime="{{ post.date | date_to_xmlschema }}" class="post-date"
        >{{ post.date | date_to_string }}</time
        >
        <br/>
    {% endfor %}
</div>