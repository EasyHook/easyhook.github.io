---
layout: default
title: Tutorials
---
<h2>Managed (.NET) Tutorials</h2>
{% for tutorial in site.tutorials %}
  {% if tutorial.managed == true %}
 1. [{{ tutorial.title }}]({{ tutorial.url }})
  {% endif %}
{% endfor %}

<h2>Native Tutorials</h2>
{% for tutorial in site.tutorials %}
  {% if tutorial.managed == false %}
 1. [{{ tutorial.title }}]({{ tutorial.url }})
  {% endif %}
{% endfor %}

Any tutorial requests, feedback, errors or questions please head over to the tutorial source GitHub repository [found here](https://github.com/EasyHook/EasyHook-Tutorials).