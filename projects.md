---
layout: page
title: Projects
description: A list of my open source software development projects
permalink: /projects
---

Here you can find short write-ups of the programming projects on [my GitHub page](https://github.com/Ben-D-Anderson).
More information on each of the projects can be found in their respective GitHub repository.
<div class="projects">
    <br/>
    {% for post in site.tags.project %}
        <h2 class="post-title">
            <a href="{{ post.url | relative_url }}">
                {{ post.title }}
            </a>
        </h2>

        <time datetime="{{ post.date | date_to_xmlschema }}" class="post-date">{{ post.date | date_to_string }}</time>

        <br/>
    {% endfor %}
</div>