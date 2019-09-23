---
layout: page
title: "Tools: various things I use and enjoy"
menuitem: "Tools"
permalink: /tools/
---

I tend to spend quite some time on tools I am working with (maybe a little more than is really needed). Configs for text editors, window managers, setting up backups, keyboards, software -- the list is pretty long. I have several notes planned here that I hope might be interesting to the public. However, if you have any specific question in mind, or if you noticed a part of my working setup that you find interesting -- please, let me know, I could prioritize it. Also, if you have a comment / addition to what is already here, that you think would be useful to include into the post -- please consider dropping me an email.

<ul>
{% for post in site.categories.tools %}
<li><span class="datebox">{{ post.date | date_to_string }}</span> &nbsp; 
<a href="{{ post.url }}">
{% if post.tags contains 'short' %} 
  {{ post.title }}</a>
  {{ post.content }}
{% else %}
  {{ post.title }} <b>(click for details)</b></a>
  {{ post.excerpt }}
{% endif %}
</li>
{% endfor %}
</ul>
