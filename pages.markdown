---
layout: page
title: Pages
permalink: /pages/
nav: true
---

{% for page in site.pages %}
  {% if page.path contains 'pages/' %}
* [{{ page.title }}]({{ page.url | relative_url }}) {% if page.last_modified_at %}(Last updated: {{ page.last_modified_at | date: "%Y-%m-%d" }}){% endif %}
  {% endif %}
{% endfor %}
