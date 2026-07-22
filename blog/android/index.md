---
layout: page
title: Android
permalink: /blog/android/
---

Android internals, kernel security, bootloader and firmware work.

{% for post in site.categories.android %}
- **{{ post.date | date: "%b %-d, %Y" }}** — [{{ post.title }}]({{ post.url | relative_url }})
{% else %}
Nothing here yet.
{% endfor %}

[← All posts]({{ "/blog/" | relative_url }})
