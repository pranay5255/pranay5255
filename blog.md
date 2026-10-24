---
layout: default
title: Writing
description: "Articles by Pranay Kundu on coding agents, machine learning, and building with AI."
permalink: /blog/
---

# Writing

[← Home]({{ '/' | relative_url }})

{% for post in site.posts %}
## [{{ post.title | escape }}]({{ post.url | relative_url }})

{{ post.date | date: '%B %-d, %Y' }}{% if post.reading_time %} · {{ post.reading_time }} min read{% endif %}

{{ post.description }}

{% endfor %}
