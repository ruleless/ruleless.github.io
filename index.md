---
layout: page
title: ruleless
tagline: Try one more time!
---
{% include JB/setup %}

## 最近发表 (共{{site.posts.size}}篇)

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; {{ post.draft_flag }} <a href="{{ BASE_PATH }}{{ post.url }}.html">{{ post.title }}</a></li>
  {% endfor %}
</ul>
