---
layout: page
title: "Studying and Teaching"
menuitem: "Edu"
permalink: /edu/
---

In this section I will summarize some materials that I hope might be useful for further teaching and/or self-study on matters that have something to do with operations research, data science and applied math (and maybe even some tiny notes on bioinformatics -- we'll see). 

I am very interested in polishing these, and making them actually useful -- so any thoughts and suggestions on the content, form or teaching process per se are highly appreciated. In particular, should it happen you find a mistake / typo (which is very well possible) -- please, let me know.

<ul>
{% for post in site.categories.edu %}
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
