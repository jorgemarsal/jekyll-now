---
layout: page
title: Business
permalink: /biz/
---
<ul class="posts">
{% for post in site.tags.biz limit: 20 %}
  <div class="post_info">
    <li>
         <a href="{{ post.url }}">{{ post.title }}</a>
         <span>({{ post.date | date:"%Y-%m-%d" }})</span>
    </li>
    </div>
  {% endfor %}
</ul>
