---
title: Test page
layout: default
---

# Index test

<ul>
{% for page in site.pages %}
	{% for name in site.sidebar %}
		{% if page.name == name %}
			<li><a href="{{ page.url }}">{{ page.title }}</a></li>
		{% endif %}
	{% endfor %}
{% endfor %}
</ul>

