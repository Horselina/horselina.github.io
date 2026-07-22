---
layout: page
title: Blog
permalink: /blog/
---

## All posts

{% for post in site.posts %}
- **{{ post.date | date: "%b %-d, %Y" }}** — [{{ post.title }}]({{ post.url | relative_url }})
{% else %}
Nothing published yet.
{% endfor %}

## Categories

{% for cat in site.categories %}
- [{{ cat[0] }}]({{ "/blog/" | append: cat[0] | append: "/" | relative_url }}) — {{ cat[1] | size }} post{% if cat[1].size != 1 %}s{% endif %}
{% else %}
No categories yet.
{% endfor %}
