---
layout: page
title: riddles
tagline: ripples
---
{% include JB/setup %}

##the quiter you become, the more you are able to hear.

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



