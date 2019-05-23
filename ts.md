---
layout: page
title: "Teaching and studying"
menuitem: "Teach / Study"
---

This is a test, if it appears
<ul>
{% for post in site.categories.ts %}
   <li><span>{{ post.date | date_to_string }}</span> &nbsp; <a href="{{ post.url }}">{{ post.title }}</a>
   {{ post.excerpt }}
   </li>
{% endfor %}
</ul>
