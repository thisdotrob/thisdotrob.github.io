---
layout: page
title: Tags
permalink: /tags/
---

{% for tag in site.tags %}
<a class="page-link" href="/tag/{{ tag | first }}">{{ tag | first }}</a>
{% endfor %}
