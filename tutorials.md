---
layout: default
title: Tutorials
---
{% assign sorted_tutorials = (site.tutorials | sort: 'index') %}
<h2>Managed Tutorials (C#/.NET)</h2>
{% for tutorial in sorted_tutorials %}
  {% if tutorial.managed == true %}
 1. [{{ tutorial.title }}]({{ tutorial.url }})
  {% endif %}
{% endfor %}

<h2>Native/unmanaged Tutorials (C++)</h2>
{% for tutorial in sorted_tutorials %}
  {% if tutorial.managed == false %}
 1. [{{ tutorial.title }}]({{ tutorial.url }})
  {% endif %}
{% endfor %}

Any tutorial requests, feedback, errors or questions please head over to the tutorial source GitHub repository [found here](https://github.com/EasyHook/EasyHook-Tutorials).