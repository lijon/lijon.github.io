---
title: Test page
layout: default
---

# Index

<ul>
{% for name in site.sidebar %}

  {% assign page = site.[name] %}

<li><a href="{{ page.url }}">{{ page.title }}</a></li>

{% endfor %}
</ul>

